# `detect.rs` 解读

## 概述

`detect.rs` 是 mediad（Microduck 机器人的媒体守护进程）中的**目标检测集成模块**，负责在摄像头帧流上寻找其他 Microduck 机器人。它集成了 `duck-detect` 库，在 GStreamer pipeline 的 tee 分支原始帧上运行推理。

**核心设计意图**体现在文件头注释的两段话中：

> *Looking for other ducks in the frames already on the tee.*
> —— 在 tee 上已经流过的帧里寻找其他鸭子。

> *`architecture.md` §2 wants perception next to the sensor — deriving features rather than shipping pixels to `robotd` — and §5.3 wants a frame on demand.*
> —— `architecture.md` §2 要求感知紧贴传感器——在本地提取特征而非把像素传给 `robotd`——§5.3 要求按需取帧。tee 的原始分支自 pipeline 写好以来就一直存在，本模块是第一个读取它的东西。

文件开头明确了两个关键架构决策：

1. **一个独立线程，不是 tokio task**（*A thread, not a task*）：推理每帧阻塞 60ms，而 tokio runtime 同时服务 WebRTC 信令。如果检测器占用它的 worker，会让会话建立出现无谓的卡顿。
2. **以热节流为节拍**（*Paced, and the pace is a thermal number*）：在 Radxa Zero 3 上全速跑检测器会到 95°C，CPU 降频到 408 MHz。两秒看一次（2 Hz）对"那边有没有鸭子"已经足够，代价约一个核的十分之一。

## 关键结构体与函数

### `Sighting` — 一次"目击"

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Sighting {
    pub width: u32,
    pub height: u32,
    pub found: Vec<Detection>,
    pub took_ms: f64,
}
```

一次检测的完整结果：画面尺寸（消费端据此缩放框坐标）、检测到的框列表、以及推理+解码耗时（毫秒，按单帧计，不取平均）。

注释说明：
> *The frame the boxes are in, so a consumer can scale them to whatever it is drawing on.*
> —— 框所在帧的尺寸，让消费端可以按自己的画布缩放。

> *Inference plus decode, in milliseconds — on the frame, not averaged.*
> —— 推理加解码的总耗时，毫秒——按这一帧计，不取平均。

### `Detector` — 检测器句柄

```rust
#[derive(Clone)]
pub struct Detector {
    pub sightings: broadcast::Sender<Arc<Sighting>>,
    looks: Arc<AtomicU64>,
    seen: Arc<AtomicU64>,
}
```

对外暴露的句柄：一个 `broadcast::Sender` 用于发布目击流，两个原子计数器用于 `robot.health` 风格的报告。`looks` 是总看了多少次，`seen` 是其中有多少次找到了东西。

`counters()` 方法用 `Ordering::Relaxed` 加载——因为这只是统计信息，不参与同步决策。

### `Backend` 枚举 — NPU vs CPU 推理后端

```rust
enum Backend {
    Npu(duck_detect::rknn::Model),
    Cpu(duck_detect::onnx::Model),
}
```

**按模型文件扩展名选择后端**，而非配置开关：

> *Chosen by the model's own extension rather than by a config switch: a `.rknn` only runs on the NPU and an `.onnx` only runs on the CPU, so asking somebody to say both is asking them to contradict themselves.*
> —— 由模型自身的扩展名决定，而不是配置开关：`.rknn` 只能跑在 NPU 上，`.onnx` 只能跑在 CPU 上，让人同时指定两者等于让人自相矛盾。

`Backend::open` 根据扩展名分发到 rknn 或 onnx 模型加载，并记录 API 版本、驱动版本等信息。如果 `.rknn` 模型 NPU 不接受（驱动未就绪），会 fallback 到 CPU——此时日志写明这一点。

### `spawn_first` — 多模型容错启动

```rust
pub fn spawn_first(
    models: &[std::path::PathBuf],
    frames: Frames,
    hz: f64,
    threshold: f32,
    turn: Turn,
) -> Result<Detector>
```

按列表顺序尝试加载模型，第一个成功的就用。全部失败时给出带排查指引的错误：

> *For the NPU: sudo /usr/local/sbin/robot-setup-npu*
> —— NPU 问题时的排查命令。

fallback 用 `warn` 而非静默吞掉，因为从 NPU 降到 CPU 是个值得在日志中看到的决策——60ms 和"别人的 CPU 上 60ms"是不同的事。

### `spawn` — 启动检测线程

这是本文件的核心。关键设计点：

#### 1. 专用 OS 线程，而非 tokio task

```rust
std::thread::Builder::new()
    .name("duck-detect".into())
    .spawn(move || { ... })
```

用命名 OS 线程而非 `tokio::spawn`，因为推理是 60ms 的阻塞工作，不能占 tokio runtime 的 worker。

#### 2. 截止时间节拍（deadline-based pacing）

```rust
// Paced from the deadline rather than by sleeping a period after the work, so a
// slow inference eats its own slot instead of drifting the whole loop later.
// —— 从截止时间开始节拍，而不是工作完成后再睡一个周期，
//    这样慢的推理吃掉自己的时间段，而不会让整个循环往后漂移。
let now = Instant::now();
if next > now {
    std::thread::sleep(next - now);
}
next += period;
```

这是经典的实时调度技巧：不是"做完→睡一会→再做"，而是维护一个 `next` 截止时间，每次循环从截止时间开始睡。如果某帧推理慢了，它吃掉的是自己的时间片，不会让后续帧的节拍漂移。

#### 3. 优雅停止条件

```rust
// The sender is the only thing keeping this thread alive: when the pipeline goes
// away, so does the receiver count — but a broadcast with no receivers is not an
// error, so the loop ends when the *sender* is dropped, which happens when the
// `Detector` does.
// —— 唯一让线程存活的是发送者：pipeline 消失时接收者计数也消失——
//    但没有接收者的 broadcast 不是错误，所以循环在发送者被 drop 时结束，
//    也就是 `Detector` 被 drop 时。
if sightings.receiver_count() == 0 && Arc::strong_count(&looks) == 1 {
    tracing::debug!("nothing holds the detector; stopping");
    return;
}
```

线程在没有任何人持有 `Detector` 且没有 broadcast 接收者时自动退出。

#### 4. 心跳日志：增量统计

```rust
const REPORT_LOOKS: u64 = 20;
```

每 20 次看一眼打一条 info 日志。关键设计：**打的是增量而非累计**。

> *A heartbeat answers "is it seeing a duck now", and a cumulative `seen` cannot: one found twenty minutes ago and one found this second both read `seen=1`, for ever.*
> —— 心跳回答的是"它现在有没有看到鸭子"，累计 `seen` 做不到：二十分钟前找到一个和这一秒找到一个，都永远显示 `seen=1`。

日志同时包含 `starved`（没有帧可看的次数），用于区分三种外部无法区分的状态：线程死了、tee 静默了、以及房间里真没鸭子。

#### 5. 帧获取与格式校验

```rust
let Some(frame) = frames.latest() else {
    starved += 1;
    continue;
};
if frame.format != crate::pipeline::CAPTURE_FORMAT {
    tracing::warn!(...);
    return;
}
```

`frames.latest()` 是 tee 上 leaky queue 的最新帧——非阻塞、last-value-wins。如果格式不对（tee 换了格式），检测器直接退出而非瞎猜。

#### 6. 直接 letterbox，不走全帧转换

```rust
// One pass from the tee's 4:2:2 straight into the model's square. Converting the
// whole 720×1280 frame and shrinking it afterwards cost 345 ms of a 407 ms look.
// —— 从 tee 的 4:2:2 直接一次送入模型的正方形。
//    先转整帧 720×1280 再缩小，在 407ms 的一看中要花 345ms。
let fit = letterbox_from_uyvy(&frame.data, ...);
```

这是一个重要的性能踩坑点：直接从 UYVY 4:2:2 格式做 letterbox 缩放到模型输入尺寸，避免了全帧色彩空间转换的巨大开销。

#### 7. 推理失败去重

```rust
// Once per distinct message: a failure that repeats at 2 Hz would be 7000
// identical lines an hour, which is how a journal stops being read.
// —— 每种不同的错误消息只报一次：2 Hz 重复失败一小时就是 7000 行相同内容，
//    日志就是这样变得没人看的。
```

用 `last_error` 缓存上次错误文本，相同错误不重复打日志。

#### 8. 检测结果以"直立"坐标发布

```rust
// Upright, because that is the space the boxes are in — a consumer scaling them
// against the *camera's* dimensions would have them sideways.
// —— 用直立坐标，因为框就在这个空间里——消费端按摄像头原始坐标缩放会是侧的。
let (upright_w, upright_h) = turn.upright(frame.width as usize, frame.height as usize);
```

因为 pipeline 不做 `videoflip`（会破坏硬件编码器的零拷贝路径），旋转是消费端的事。检测框按旋转后的"直立"尺寸报告，浏览器据此在 CSS 旋转后的视频元素上正确叠加。

### `video_notification` — 视频流通知

```rust
pub fn video_notification(width: u32, height: u32, rotate_degrees: u32) -> String
```

向 peer 告知视频尺寸和安装旋转角度的 JSON-RPC notification。核心点：

> *The rotation is the point of this. Nothing rotates pixels any more, so the stream a browser receives is the picture the camera took — sideways, on a robot whose camera is mounted a quarter turn off.*
> —— 旋转是重点。不再有任何东西旋转像素，浏览器收到的流就是摄像头拍的原始画面——侧着的，因为机器人摄像头安装偏了四分之一圈。

浏览器无法仅凭宽高比推断旋转角度（180° 安装和直立安装宽高比一样），必须显式告知。

### `notification` — 单次目击的 JSON-RPC 通知

```rust
pub fn notification(sighting: &Sighting) -> String
```

把手写的 JSON 拼成一行：`{"jsonrpc":"2.0","method":"media.detections","params":{...}}`。

> *Hand-built rather than through serde: it is five numbers per box at 2 Hz, the shape is fixed, and this crate has no serde dependency to add for it.*
> —— 手写而非用 serde：每个框五个数字，2 Hz，形状固定，没必要为此加 serde 依赖。

无 `id`——notification 模式，页面把无 id 的行当作流式数据处理（和 `robot.state` 一样）。

## 重要常量

| 常量 | 值 | 含义 |
|---|---|---|
| `REPORT_LOOKS` | `20` | 每看 20 次打一条心跳日志。2 Hz 下即每 10 秒一条 |
| broadcast channel 容量 | `8` | `sightings` 的 broadcast 通道容量 |
| `hz` 下限 | `0.1` | `period = 1.0 / hz.max(0.1)`，防止 hz=0 导致除零 |

## 测试要点

### `a_sighting_serialises_as_a_notification`

验证 `notification()` 产出合法 JSON-RPC notification：
- `jsonrpc` 字段为 `"2.0"`
- `method` 为 `"media.detections"`
- **没有 `id` 字段**（notification 模式）
- 框坐标和 score 数值正确

### `an_empty_sighting_is_still_sent`

> *A detector that goes quiet when it sees nothing leaves the last duck drawn on screen for ever, which looks exactly like a duck that is still there.*
> —— 检测器看不到东西就静默，会让最后一只鸭子永远画在屏幕上，看起来就像鸭子还在。

验证空检测结果仍然发出消息（`boxes` 为空数组），让页面能清除叠加框。

## 与其他模块的关系

```
pipeline.rs (Frames)  ──tee 原始帧──▶  detect.rs
                                         │
                                         ▼ broadcast
                                    session.rs / web.rs
                                    (JSON-RPC 推送)

duck_detect 外部 crate
  ├── rknn::Model  (NPU 推理)
  └── onnx::Model  (CPU 推理)
```

- **上游**：`crate::pipeline::Frames` 提供 tee 上的最新帧（`latest()` 方法，leaky queue，last-value-wins）。`CAPTURE_FORMAT` 定义了期望的像素格式。
- **下游**：`Detector.sightings` broadcast 通道被 Web 控制台消费，经 `notification()` 序列化为 JSON-RPC 推送给浏览器。
- **外部依赖**：`duck_detect` crate 提供 `Detection`、`Turn`、`decode`、`letterbox_from_uyvy` 以及 rknn/onnx 两种模型后端。
- **`exposure.rs`**：同属于 Linux-only 模块，同样从 tee 读帧（自动曝光）。`REPORT_LOOKS` 的设计理由与 `exposure` 的 `REPORT_TICKS` 一脉相承。
#（注：内容由AI生成）
