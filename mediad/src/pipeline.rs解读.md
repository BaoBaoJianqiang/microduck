# `pipeline.rs` 解读

> 对应文件：`mediad` crate 中的 `src/pipeline.rs`（1430 行，是全 crate 最大的单文件）。
> 运行平台：Rockchip RK3568 机器人（Radxa Zero 3 一类板子），IMX219 摄像头经 rkisp ISP 采集，VPU 上 `mpph264enc` 硬件编码，WebRTC 推流到浏览器控制台。
> 本文按"角色 → 设计意图 → 逐件拆解 → 踩坑点 → 测试 → 模块关系"组织，代码注释一律采用"英文原文 —— 中文翻译"并列格式。

---

## 1. 概述：这个文件在 crate 里是什么

`pipeline.rs` 是 `mediad`（Microduck 机器人的媒体守护进程）里**唯一**跟硬件、GStreamer、V4L2、VPU 直接打交道的模块，也是**唯一不跨平台**的模块。`lib.rs` 用 `#[cfg(target_os = "linux")]` 把它和 `exposure.rs`、`detect.rs` 一起 gate 住：

```rust
/// The GStreamer pipeline and the datachannel. Linux only — see the crate manifest for why the
/// gate is by target rather than by feature.
// GStreamer 管线和数据通道。仅 Linux —— 见 crate manifest 说明为什么按 target 而不是按 feature gate。
#[cfg(target_os = "linux")]
pub mod pipeline;
```

文件头注释把它的定位说得很清楚：

> Linux only, the way `padd`'s evdev tap is: the daemon runs on the robot, and everything else in this crate — `route`, `session`, `upstream` — is portable and stays testable on a laptop.
>
> —— 跟 `padd` 的 evdev 抓键一样只跑在 Linux 上：守护进程跑在机器人上，而本 crate 的其余部分（`route`、`session`、`upstream`）都是可移植的，能在笔记本上跑测试。在这里（而不是用 feature gate）做平台隔离，是为了让 `cargo test` 在两边都诚实。

**核心设计意图**：在传感器旁边就地完成"采集 → 切流 → 硬件编码 → WebRTC 推流"，同时**零拷贝**地把原始帧旁路给感知（目标检测）和软件自动曝光用。整个文件的注释密度极高，几乎每一个"为什么这么写"都是一次真机踩坑后的结论。

### 管线形状（文件头 ASCII 图的翻译）

```text
                                                ┌─ queue ─ webrtcsink ─ setup → mpph264enc
 videotestsrc | camera ─ UYVY ─ videoflip ─ tee ┤          │
                                                └─ queue ─ │  run-signalling-server=true
                                       (leaky)     │
                                          │        └─ consumer-added → "control" channel
                                      appsink
                                     latest frame
```

即：**源 → capsfilter 锁死格式/分辨率/帧率 → （可选 videoflip）→ tee 分叉**：

- **视频分支**：`queue → webrtcsink`（`webrtcsink` 自己挑编码器，本工程打了补丁让它认识 `mpph264enc`）；
- **原始帧分支**：`leaky queue（1 帧深）→ appsink`，喂给 `Frames` 这个 last-value-wins 快照。

---

## 2. 顶层设计决策（踩坑点的总纲）

文件头的 doc 注释连续讲了六条"为什么"，每一条都是真机上量出来的结论，是全文最重要的部分。

### 2.1 tee 放在原始 NV12/UYVY 上、编码器之前

> **The tee is on raw NV12, before the encoder**, and that placement is the point of it. ... Both need pixels, and taking them off the encoded branch would mean decoding what we just encoded.
>
> —— tee 放在编码器之前的原始流上，这个位置就是全部意义所在。感知模块和 `get_frame` 按需取帧接口都要像素；如果从已编码分支取，就得把刚编码出来的 H.264 再解码一遍。

对应 `architecture.md` §2（感知要贴着传感器做特征推导，而不是把像素搬给 `robotd`）和 §5.3（服务端程序只要一两秒一帧加状态 blob）。

### 2.2 为什么是 UYVY 而不是 NV12：纯测量结论

> NV12 because that is what the rkisp capture path emits ... **Nothing converts and nothing rotates.**
>
> —— 原注释说 NV12 是 rkisp 出的格式。但正文实测后把 caps 锁成了 `UYVY`（见 `CAPTURE_FORMAT` 与 start 里的实测表）：要 `NV12` 会选到双层非连续的 `NM12`，`v4l2src` 在这个驱动上打不满帧率；单层 `UYVY` 才能跑满 29.3 fps。而且 `mpph264enc` 的 sink pad 列了 `UYVY`，它把 4:2:2→4:2:0 交给 SoC 2D 引擎（RGA）做，CPU 零开销。

实测表（720p，300 帧）：

| caps | 2 buffers | 4+ buffers |
|---|---|---|
| `NV12`（选中双层 `NM12`） | 19.5 fps | 19.6 fps |
| `UYVY`（单层） | 19.7 fps | **29.3 fps** |

### 2.3 绝不在管线里旋转（videoflip 事故）

这是全文被反复引用的最大坑：

> This defaulted to a quarter turn for one afternoon and cost 145% of a core. ... Measured on the robot: 97 °C, the CPU throttled from 1.8 GHz to 408 MHz, 1565 frames lost by `v4l2src` in one session, and 8 fps out of a 30 fps camera.
>
> —— 有一个下午默认加了 90° 旋转，结果烧掉一个核 145% 的算力。`mpph264enc` 本来把 UYVY→NV12 交给 RGA 零成本做完；`videoflip` 输出的 buffer RGA 拒绝（`10000 is unsupport format` / `RGA_BLIT fail: Bad address`），MPP 被迫**每帧软件转格式**。实测：97 °C、CPU 从 1.8 GHz 降到 408 MHz、30 fps 摄像头只出 8 fps、一个会话丢 1565 帧。

结论：**旋转是消费者的事，而且对两个消费者都免费**——浏览器端用 CSS transform 在 GPU 上转；感知端把旋转折进它本来就要做的重采样。管线里只剩一个显式开关 `--flip-in-pipeline`（`Rotation` 非 `None`），把代价写下来后让它成为"选择"而不是"默认"。

### 2.4 每分支独立 queue，原始分支 leaky

> A `tee` without queues runs its branches on one thread, so a slow consumer stalls the others ... The raw branch drops old frames rather than applying backpressure, which is the semantics ...: the *latest* snapshot, non-blocking, last-value-wins.
>
> —— 没有 queue 的 tee 把所有分支跑在一个线程上，慢消费者会卡死别的分支；这里慢了就会让感知拖慢视频轨。原始分支丢弃旧帧而不是反向压流，语义就是"最新快照、非阻塞、last-value-wins"。读者卡住，丢的是帧，永远不是编码器。

### 2.5 webrtcsink 自己进程内跑信令服务器

> `webrtcsink` runs the signalling server in this process (`run-signalling-server`, with `signalling-server-host` and `-port`), so the separate `gst-webrtc-signalling-server` binary never has to be built or shipped — what we ship from that upstream is a `.so`.
>
> —— 信令服务器跑在本进程里，不必单独构建/分发 `gst-webrtc-signalling-server` 二进制；上游只以 `.so` 形式随插件发布。

### 2.6 把编码器交给 webrtcsink，而不是自己编完再喂

管线曾经是 `mpph264enc ! h264parse ! webrtcsink`，能工作但悄悄丢掉两样东西：

> with pre-encoded input `webrtcsink` cannot reach the encoder, so its congestion control cannot adapt the bitrate to the link, and a peer's PLI cannot produce a keyframe, leaving a viewer that lost one broken until the next periodic GOP.
>
> —— 喂预编码流时 webrtcsink 够不到编码器：拥塞控制没法按链路自适应码率；对端 PLI（丢包后请求关键帧）也无法触发关键帧，丢过一帧的观看者要等到下一个周期 GOP 才能恢复。

代价是：对 webrtcsink 不认识的编码器，它会在前面塞软件 `videoconvert ! videoscale`。它不认识 `mpph264enc`，所以**随插件发布时打了补丁**给它加上这条 arm（见 `patches/` in `pollen-robotics/microduck-gst-plugins`）。补丁和这套架构是绑定的——没补丁的话这样反而比预编码更慢。

### 2.7 信号处理函数里绝不能 panic（非展开 abort）

这是贯穿全文件的一条铁律，文件头专门有一节 "What is not verified"：

> Nothing in a signal handler here may panic. These closures are invoked from C, so a panic does not unwind — it aborts the process, and the journal shows `thread caused non-unwinding panic` ... The first board run died exactly that way, from `tokio::spawn` on a GStreamer thread that has no runtime.
>
> —— 信号处理闭包是从 C 调进来的，panic 不会 unwinding 而是直接 abort 整个进程，journal 里只看得到 `g_closure_invoke` 的 backtrace，看不出真正原因。第一次上板就是这么死的：在没有 tokio runtime 的 GStreamer 线程里 `tokio::spawn`。

由此推出两条工程纪律：

1. 在有 runtime 的地方抓 `Handle`，闭包里显式 `runtime.spawn`；
2. 每个信号名在 `connect` / `emit_by_name` / `set_property` **之前**都先 `SignalId::lookup` / `has_property` 查一遍——这些调用对不存在的名字是直接 panic。

残留风险只剩"签名存在但 arity 变了"，那种会以"警告指出参数个数"的形式出现，而不是 abort。

---

## 3. 关键类型与函数逐一拆解

### 3.1 `enum Rotation`（L108–156）

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Rotation {
    None,      // Leave the frame as the sensor delivered it. —— 传感器怎么给就怎么留
    Cw90,      /// A quarter turn clockwise: what this robot's mount needs. —— 顺时针 90°，本机器人安装角度
    Cw180,
    Cw270,     /// A quarter turn anticlockwise, for a head assembled the other way round. —— 逆时针 90°，反向装的头
}
```

- **`from_degrees(u32)`**：只接受 0/90/180/270，其余报错。报错信息特意写全 `"rotation must be 0, 90, 180 or 270 degrees clockwise, not {other}"`，让打错字的人一眼看懂。
- **`video_direction(self)`**：映射到 `videoflip` 的 `video-direction`。注意 GStreamer 的命名是 `90r`=顺时针、`90l`=逆时针（不是我们自己的命名）。`None` 返回 `None`——**根本不构造 videoflip 元素**，这趟零成本。
- **`output(width, height)`**：旋转后的帧尺寸。90/270 交换宽高。这是个 load-bearing 函数：`Frame` 带着 buffer 实际尺寸，一旦猜错，消费者会把 720x1280 的图当 1280x720 读，"不是失败，而是斜着花屏"，还会被怪到摄像头头上。

### 3.2 `struct Settings`（L164–182）

```rust
pub struct Settings {
    pub host: String,              // 信令绑定地址（全接口）
    pub port: u32,                 // 信令端口，注意不是 web 端口（web 归 crate::web）
    pub bitrate: u32,              // 起始码率 bps；拥塞控制会动它；disabled 时它就是定死速率
    pub congestion_control: robotd_params::CongestionControl,
    pub width: u32, pub height: u32, pub fps: u32,
    pub rotation: Rotation,
}
```

注释解释了为什么用一个 struct 而不是八个位置参数：

> One value rather than six positional arguments. `start` had reached eight of them, and two of those are a `width` and a `height` of the same type: a call site that swapped them would compile and produce a portrait stream.
>
> —— 用一个结构体而不是六个位置参数。`start` 曾经到了八个参数，其中 width/height 同类型，调用点写反了也能编译通过，然后出一路竖屏流。命名字段让这种错误"无法表达"，第七个设置项也不用再改签名。

### 3.3 `enum Source` / `struct Camera`（L185–208）

```rust
pub enum Source {
    Test,   // 无摄像头也能跑，整个会话（信令/协商/datachannel/控制 API）都能被驱动起来
    Camera(Camera),
}

pub struct Camera {
    pub device: String,
    pub exposure: u32,          // 单位是 sensor lines，约 19µs/行
    pub analogue_gain: u32,     // 256 = 1x，最高 2816 = 11x
}
```

`Camera` 的注释是又一条踩坑史：

> These are a starting exposure, not a policy. A capture with the driver's boot defaults comes out black ... Rockchip's 3A engine converges once at stream start and then stops — and does not manage even that if it missed the stream-start event.
>
> —— 这只是**起步曝光**，不是策略。用驱动 boot 默认值拍出来是黑的，所以第一帧前必须有人写 sensor。之后由 `crate::exposure` 接管测光——因为 Rockchip 3A 引擎只在流启动时收敛一次就停了，错过 stream-start 事件连那一次都没有。

### 3.4 `struct Frame` / `struct Frames`（L211–245）

```rust
#[derive(Debug, Clone)]
pub struct Frame {
    pub width: u32,
    pub height: u32,
    pub format: &'static str,   // 带 CAPTRUE_FORMAT 而不是假定，换格式时读字节的人不会静默误解
    pub data: Vec<u8>,
}

#[derive(Clone, Default)]
pub struct Frames(Arc<Mutex<Option<Frame>>>);
```

`Frames` 是 last-value-wins 快照：appsink 回调每次直接**替换**而不是排队。

- **`inspect(&self, read)`**：不动拷贝地在锁内读一个数。为什么需要它？自动曝光循环只要一个 luma 均值，"把 1.8 MB 的帧每秒拷两遍去平均其中 11k 字节，是一次没人需要的 memcpy"。
- **`latest(&self)`**：克隆整帧返回，给检测等要拿整像素的消费者用。

### 3.5 `struct Channel`（L248–253）

```rust
pub struct Channel {
    pub inbound: mpsc::Receiver<String>,   // 对端发来的行
    pub outbound: mpsc::Sender<String>,  // 要发给对端的行
}
```

一个 peer 一路 `control` datachannel，两端就是这两个 mpsc。`pipeline::start` 返回 `(Pipeline, mpsc::Receiver<Channel>, Frames)`——**管线归调用方持有**，drop 它就停会话，shutdown 语义自然成立。

### 3.6 `start()`：主装配函数（L259–576）

按顺序做的事：

1. **算输出尺寸** `rotation.output(width, height)`；
2. `set_gstreamer_log_threshold()`（在 `gst::init` 之前塞 `GST_DEBUG` 环境变量）；
3. `gst::init()`；
4. `bridge_gstreamer_log()`（必须在 init 之后，见 3.9）；
5. **抓 tokio runtime handle**——这是硬性要求：

```rust
let runtime = tokio::runtime::Handle::try_current().context(
    "pipeline::start must be called from inside a tokio runtime: the datachannel writer is \
     spawned onto it from a GStreamer signal thread, which has no runtime of its own",
)?;
```

> —— start 必须在 tokio runtime 里调用：datachannel 写任务是从 GStreamer 信号线程 spawn 上去的，而那些线程自己没有 runtime。

6. **建源**：`Source::Test` 用 `videotestsrc` 并显式 `is-live=true`（模拟摄像头的 live 时钟行为，否则测试源会往前跑赢时钟）；`Source::Camera` 走 `camera_source`；
7. **capsfilter 锁死 caps**：`video/x-raw, format=UYVY, width, height, framerate=fps/1`。注释强调是 pinned 而非协商——tee 的两个分支都依赖这个答案，裸消费者猜错格式的代价是"源一变就错"；
8. **可选 videoflip**：只有 `rotation.video_direction()` 返回 `Some` 才建，且报错信息直接告诉运维 `sudo /usr/local/sbin/robot-setup-gstreamer`；
9. **建 tee、video_queue、webrtcsink**，配信令服务器、`meta`、`video-caps`、`start-bitrate`、拥塞控制、`encoder-setup` 钩子、consumer 信号；
10. **建原始分支**：`queue(max-size-buffers=1, leaky=downstream)` + `appsink(sync=false, max-buffers=1, drop=true)`；
11. 全部 `add_many` 上管线，`link_many` 串起源→capsfilter→[flip]→tee，再分别 link 两条分支；
12. **tee 的 src pad 是 request pad**，所以两个分支用 `link_tee_branch` 单独请求 `src_%u`；
13. `meter_capture_rate` 挂在 capsfilter 的 src pad（tee 之前）；
14. `watch_bus` 起总线监听线程；
15. `set_state(Playing)` 启动。

### 3.7 webrtcsink 的关键属性配置（L392–463）

```rust
sink.set_property("run-signalling-server", true);
sink.set_property("signalling-server-host", host);
sink.set_property("signalling-server-port", port);
```

**`meta` 属性（机器人身份）**——先 `has_property` 再 set：

```rust
if sink.has_property("meta") { ... } else { tracing::warn!(...) }
```

> `set_property` panics on a name the element does not have, and a panic here is a daemon that will not start — costing the video and the control channel to gain a producer's name.
>
> —— 对元素没有的属性名 set_property 会 panic，这里 panic 就是守护进程起不来：为了一个生产者名字把视频和控制通道都搭进去。这是本函数最新碰的东西，最容易拼错，宁可匿名也不能起不来。

**`video-caps` 只 offer H.264**：

```rust
sink.set_property("video-caps", gst::Caps::builder("video/x-h264").build());
```

> Left alone `webrtcsink` proposes everything it can encode: ... but `vp9enc` and `av1enc` in *software*. A browser preferring AV1 would have this robot software-encoding AV1 on four Cortex-A55s, which is not a degraded stream but a dead control loop.
>
> —— 不加限制时 webrtcsink 会把能编的全 offer 出去，其中 vp9/av1 是纯软件的。浏览器偏好 AV1 的话，这台机器人就在四个 Cortex-A55 上软编 AV1——那不是"降质"，而是**控制回路死掉**。

**故意不在 caps 里写 `profile` 字段**：

> `webrtcsink` reads one off these caps and does `H264_PROFILES_COMPAT.iter().position(..).expect("Unsupported H264 profile")` — a panic, in a plugin, for a value it does not know. Omitting the field skips that path, and the profile is set on the encoder itself in `wire_encoder_setup` where it belongs.
>
> —— 在 caps 里写 profile 会触发 webrtcsink 插件内部 `.expect(...)` 直接 panic。省略这个字段就绕过那条路径，profile 在 `wire_encoder_setup` 里在编码器本体上设，那才是它该在的地方。

另一条历史：这个限制曾一度没生效，因为 `mpph264enc` 的 pad 模板漏了 `constrained-baseline`（webrtcsink 发现流程硬要这个 profile）；插件 release 从 v3 起打了补丁。老插件的机器人走到这里现在会**响亮失败（一个 producer 都没有）**而不是悄悄发 VP8。

### 3.8 原始帧分支与 `wire_frames`（L465–624）

```rust
let raw_queue = gst::ElementFactory::make("queue")
    .property("max-size-buffers", 1u32)
    .property("max-size-bytes", 0u32)
    .property("max-size-time", 0u64)
    .property_from_str("leaky", "downstream")
    .build()?;
```

appsink：

```rust
let appsink = gst_app::AppSink::builder()
    .caps(&out_caps)
    .sync(false)      // 快照不等时钟：消费者不渲染，pacing 只加延迟
    .max_buffers(1)
    .drop(true)
    .build();
```

`wire_frames` 的 `new_sample` 回调：

```rust
let Ok(map) = buffer.map_readable() else {
    // A buffer that will not map is not worth failing the pipeline over ...
    // 映射不了的 buffer 不值得搞垮管线 —— 下一帧也就一帧之遥，这条分支设计上就是咨询性的。
    tracing::debug!("a raw frame would not map");
    return Ok(gst::FlowSuccess::Ok);
};
...
*frames.0.lock().expect("frame lock") = Some(frame);  // Replaced, not queued: last-value-wins is the contract.
```

注释还解释了为什么 UYVY 好：单层平面，map 不需要像 NM12 那样把两个非连续平面拷成一块；`to_vec` 仍是这条分支上唯一一次拷贝。

### 3.9 日志桥接：`set_gstreamer_log_threshold` + `bridge_gstreamer_log`（L626–693）

为什么拆成两个函数，且一个在 `gst::init` 前、一个在后？

- **阈值必须在 init 前进环境变量**，因为 init 时就读 `GST_DEBUG`；
- **日志函数替换必须在 init 后**。早先在 init 前换，结果 WARNING/ERROR 到了，但 INFO 及以下永远不到——`GST_DEBUG=v4l2bufferpool:4` 啥也打不出来，两个本应由 `GST_INFO` 回答的采集问题只能靠帧率反推，两次都推错了。

> What is lost by moving it is a handful of registry-scan lines from before `init`, which said nothing anyone wanted.
>
> —— 移到 init 后损失的只是 init 前几行插件注册表扫描，没人关心。

总线不够用，还得桥日志，因为：

> `webrtcsink` drops a codec whose discovery pipeline fails with a `gst::warning!` and nothing else — ... and that goes to GStreamer's debug log, not the bus.
>
> —— webrtcsink 丢弃发现失败的 codec 时只打 `gst::warning!`，那去的是 GStreamer debug log 而不是总线。所以一台 offer 了 VP8 而不是 H.264 的机器人全程沉默，其实它一直在跟一个没人读的日志说话。

### 3.10 总线监听 `watch_bus`（L699–744）

> A dedicated thread rather than `bus.add_watch`, which needs a GLib main loop this daemon does not run, and rather than a tokio task, because `timed_pop` blocks.
>
> —— 用专门线程而不是 `bus.add_watch`（那需要本守护进程根本没跑的 GLib main loop），也不是 tokio task（`timed_pop` 会阻塞）。

循环 `bus.timed_pop(None)` 阻塞等消息；只把 **Error / Warning** 转发到 tracing（带 `src`、`error`、`debug` detail——detail 通常才是真正命名原因的部分：caps 不匹配、设备打不开）；其余状态变更/stream-status 一律静默（要用时 `GST_DEBUG` 开）。

为什么必须看总线：

> This was learned the hard way. ... Neither reaches `tracing`, so the journal showed a session starting, a session ending, and no reason for either. Two rounds of guessing went into diagnosing something GStreamer was already saying out loud.
>
> —— 学费是这么交的：journal 里只看得到"会话开始、会话结束"，没有任何原因；猜了两轮，其实 GStreamer 早就大声说出来了。

### 3.11 摄像头源：`camera_source` / `pin_sensor_mode` / `find_sensor`（L752–1024）

**为什么用 `v4l2src` 而不是手写 V4L2 循环**：曾经认为驱动会丢第三帧、裸字节要 `rawvideoparse blocksize=...` 且 stride padding 一出现就静默出错。但这两条都属于"子进程"形态：`v4l2src` 会挂 `GstVideoMeta` 描述真实布局，丢帧也有小修法（见 `raise_capture_buffers`）。

**曝光/增益走 `extra-controls` 而不是 `v4l2-ctl`**：

```rust
let controls = gst::Structure::builder("c")
    .field("exposure", camera.exposure as i32)
    .field("analogue_gain", camera.analogue_gain as i32)
    .build();
```

> Exposure and gain go through `extra-controls` rather than a `v4l2-ctl` call, so they are applied by whoever opens the device — including after a re-open we did not initiate.
>
> —— 谁打开设备谁就会把这两个控制量带上，包括不是我们发起的重开。

**`pin_sensor_mode`：把 IMX219 从 boot 模式切出来**（boot 模式锁 21 fps）。sensor 启动在 3280x2464，rkisp 缩放能给 720p 但是按全分辨率帧率；1920x1080 才是 30 fps 模式。它在启动时 shell out 一次 `media-ctl`：

```rust
let format = format!("\"{entity}\":0[fmt:SRGGB10_1X10/1920x1080]");
```

失败不致命（采集仍工作，只是慢），但要大声警告——"丢三分之一帧从远端看像网络问题"。

**`find_sensor` 的错误信息设计**是另一个范本：

> Every way this fails says which one it was. An earlier version returned `Option` and reported "no imx219 entity" for all of them, which sent the first real run chasing the overlay when the actual cause was `media-ctl` being denied `/dev/media0`.
>
> —— 三种失败要三种修法，但从外面看长得一样：没有 /dev/media\*（overlay 没开，要 reboot）；有节点但读不了拓扑（权限：进程不在 video 组，`systemctl` 起而不是 `sudo -u`）；读了几个都没有 imx219（overlay 名字写错）。早期版本全都报同一句话，第一次真机排错排错了方向。

### 3.12 常量 `CAPTURE_FORMAT` / `CAPTURE_BUFFERS`（L791–806）

```rust
/// What the tee carries, and what both branches therefore see.
///  —— tee 携带的格式，两个分支都看它。
pub const CAPTURE_FORMAT: &str = "UYVY";

/// How many capture buffers to ask for. Three is the cliff; four leaves one spare.
///  —— 要几个采集 buffer。3 是悬崖，4 留一个余量。
const CAPTURE_BUFFERS: u32 = 4;
```

实测表（`v4l2-ctl --stream-mmap=N`，720p NV12，300 帧）：

| buffers | 2 | 3 | 4 | 6 |
|---|---|---|---|---|
| seconds | 15.2 | 10.3 | 10.3 | 10.3 |

19.7 fps vs 传感器 29.2 fps，而 `v4l2src` 默认落到 2 个 buffer。

### 3.13 `raise_capture_buffers`：全文件最深的一个 hack（L808–915）

这是全文技术含量最高的一段，用 pad probe 在 Allocation 查询上改写池深度。注释贴了 `gst_v4l2_object_decide_allocation` 的 C 源码逻辑：

```text
can_share_own_pool = (has_video_meta || !obj->need_video_meta);
...
if (pushing_from_our_pool) {
    own_min = min + obj->min_buffers + 2;
    if (!update) own_min += 2;   /* update == 查询带了 pool */
} else {
    own_min = MAX(obj->min_buffers + 1, GST_V4L2_MIN_BUFFERS(obj));
}
```

rkisp 既不实现 `V4L2_CID_MIN_BUFFERS_FOR_CAPTURE`（`obj->min_buffers=0`），也不提供连续 NV12（只有双层 NM12，需要 `GstVideoMeta` 才能描述）。实测：

| 链 | `own_min` | fps |
|---|---|---|
| `UYVY ! queue ! fakesink` | `0+0+2+2` | 29.3 |
| `UYVY ! videoconvert ! fakesink` | `0+0+2` | 19.7 |
| `UYVY ! mpph264enc` | `0+0+2` | 19.7 |
| `NV12 ! fakesink` | else 分支 `MAX(1,2)` | 19.7 |

要凑够 4 必须同时满足两件事：

1. **带上 `GstVideoMeta`**，否则 `can_share_own_pool=false`，走 else 分支无视查询里的一切（还会每帧拷进通用池）；
2. **第一个池的 min 不能是 0**，因为任何下游元素提议 pool 都会置 `update` 标志、丢掉那个 `+2`。`GstVideoEncoder::propose_allocation` 正好提一个 `min=0` 的 pool——所以只要下游挂着 `mpph264enc` 就够触发。

**最阴的一点**：proposal 实现是**覆写池 0 而不是追加**，所以"去程写的 min"会被编码器回程的 0 盖掉。pad probe 双向都触发，于是每次看到查询都重写一次池 0，**最后一句话必须是我们的**：

> That is the bug that made three earlier versions of this function look like they were being ignored.
>
> —— 就是这个 bug 让这个函数前三个版本看起来"根本没生效"。

probe 里还只对前 4 次 pass 打 info 日志：因为"我们的改写被覆盖了"和"我们的改写站住了但被无视了"是两种不同 bug，帧率看不出区别，必须看 pool 前后对比。

### 3.14 `set_congestion_control`（L1026–1059）

> **Set rather than inherited.** `gcc` is the element's own default, so naming it changes nothing today — which is the point: what a plugin we ship from a pinned release defaults to is not a decision this robot should discover it inherited on the day upstream changes it.
>
> —— 显式设置而不是继承默认。`gcc` 恰好是元素自己的默认，所以今天写了等于没写——但意义在于：从固定 release 插件里继承来的默认值，不该有一天上游改了它、本机器人才"发现"自己被改了。

而且因为这是函数里**唯一一个来自配置文件而非字面量的值**，要防御性设置：

> `set_property_from_str` panics both on a property the element lacks and on a nickname its enum does not know ... So the property is looked up and the nickname resolved through the enum's own class; either failing leaves the element on its default and says so.
>
> —— 先 `find_property`，再用 `EnumClass::to_value_by_nick` 解析昵称；任一失败都让元素留在自己默认并打警告，远比"机器人起不来"强。

### 3.15 `wire_encoder_setup`（L1061–1133）

`webrtcsink` 每建一个编码器就发一次 `encoder-setup`（每个消费者一次，再加一次发现探测）。这是 webrtcsink 接管编码器后**唯一**能配编码器的地方。两个设置都是量出来的、且失败时都不像编码器设置问题：

- **`profile=baseline`**：产出 `h264parse` 报告的 `constrained-baseline`（`profile-level-id 42e01f`），是 WebRTC 互通底线。默认是 High，新浏览器能谈，老对端不行；
- **`header-mode=each-idr`**：每个 IDR 都重复 SPS/PPS。默认只在第一帧带 SPS/PPS，**晚加入或丢了那个包的对端永远解不出来**。

返回 `false`，让 webrtcsink 继续叠自己的配置——码率归它管，这正是整套架构的目的。

防御性细节：

- 先 `SignalId::lookup("encoder-setup")`，没有就直接报错（没有它编码器就是 High profile、SPS/PPS 只在第一帧）；
- 回调里按工厂名 `mpph264enc` 才设 `profile`/`header-mode`——只有它有这俩属性，瞎设会 panic（信号里 panic=abort）；
- 只有"真实消费者"才 warn。发现阶段会对 `mppvp8enc`、`mpph265enc` 各 fire 一次，那俩**也是硬件**，在那里喊狼来了等于每次启动都误报；对真实 peer 才大声——因为 `video-caps` 已经把 offer 限到 H.264，再来别的就意味着限制失效，有人正在 `robotd` 控制回路共享的那几个大核上软件编码。

### 3.16 `Consumers` 类型与 `meter_capture_rate`（L1135–1245）

```rust
type Consumers = Arc<std::sync::atomic::AtomicU32>;
```

用 `AtomicU32` 而不是锁：它在 GStreamer 线程上被 `consumer-added/removed` 写、在采集 probe 上被另一个线程读，两边都不能阻塞。

**为什么把测速挂在 capsfilter 的 src pad（tee 之前）**——位置即一切：

> This lived on the tee's raw branch first, which sits behind a deliberately leaky one-buffer queue — so it measured what survived that queue. ... On the pad *before* the tee there is nothing between here and the driver.
>
> —— 早先挂在 tee 的原始分支上，那是在故意漏帧的单 buffer 队列之后，测的是"活下来的"帧率。rkvenc 中断数测的是编码器消费的（还在 webrtcsink 自己的 queue 和 videorate drop-only 后面）；v4l2src 的 lost-frames 警告只数驱动序列号空洞，源慢的时候它沉默。**本次 bring-up 每个错误方向都来自这三种假数据之一。**

**驱动级丢帧靠 buffer offset 检测**：`v4l2src` 把 V4L2 序列号留在 offset 上，跳号就是"驱动采了但我们没收到"——这才是该上报的数；自己 leaky 队列丢的是"选择"，不算病。

健康判断用 90% 而不是相等（传感器时钟 ≠ CPU 时钟，窗口边界两边各落一帧不是故障），且**只在状态翻转时打日志**（每秒一行早晚会被 grep 忽略掉）。统计通过 `proto::publish_camera_stats` 上报，发送失败**故意忽略**——每秒钟都跑，警告一次就是每秒一条警告。

### 3.17 `wire_consumers` / `open_control_channel`（L1247–1380）

机器人主动建 `control` 数据通道，而不是等对端建——对端连上后什么都不建也有控制面。

- `consumer-removed` 用 `fetch_update` + `saturating_sub` 而不是 `fetch_sub`：防"误触发的移除"把计数绕回四十亿观众；
- `consumer-added` 闭包里 `try_send` 到 channel 通道，满了说明没人在收会话（bug 而不是背压），打 error 而不是阻塞 GStreamer 信号处理器；
- `open_control_channel` 里**逐一先查** `create-data-channel`、`on-message-string`、`send-string` 三个信号，缺一个就返回 Err（视频轨还活着，只是这一路控制通道不开），而不是在 `emit_by_name` 里 panic；
- 类型标注 `WebRTCDataChannel` 不是装饰：`GstObject` 是 `Send`，裸 `glib::Object` 不是，不标那个 writer 任务编译不过；
- 收方向：`on-message-string` 里 `try_send`，满了就**丢一帧控制命令**并 warn——丢控制帧很糟，但阻塞 GStreamer 信号处理器更糟，会卡死整条管线包括视频；
- 发方向：在 runtime 上 spawn 一个 writer 任务，循环 `outbound_rx.recv()` 然后 `send-string`。**session 层完全不知道 GStreamer 的存在**。

---

## 4. 重要常量汇总

| 常量 | 值 | 含义 |
|---|---|---|
| `CAPTURE_FORMAT` | `"UYVY"`（pub） | tee 携带、两个分支都看到的格式。单层平面；`NV12` 会选到双层 NM12 导致 `v4l2src` 打不满帧率；`mpph264enc` 吃 UYVY 并在 RGA 上免费完成 4:2:2→4:2:0。 |
| `CAPTURE_BUFFERS` | `4`（私有 const） | 向 v4l2src 要的采集 buffer 数。实测 3 是断崖（19.7 fps），4 才 29.3；`v4l2src` 默认落 2。 |
| 默认 `GST_DEBUG` | `*:WARNING` | 未设环境变量时兜底；WARNING 够抓到 codec 被丢/元素拒绝，又够安静。 |
| appsink | `max-buffers=1, drop=true, sync=false` | 原始分支：只留最新一帧、不等时钟。 |
| raw queue | `max-size-buffers=1, leaky=downstream` | 慢读者丢最老帧、保最新帧。 |
| Channel mpsc | 各 64 容量 | 控制通道收发缓冲。 |
| `channels` 通道 | 容量 4 | 待交付给 session 的 peer 控制通道。 |

---

## 5. 状态管理与总线消息处理

文件里**没有**手写一套 PLAYING/PAUSED/NULL 状态机：状态切换交给 GStreamer 自己，代码只做两件事：

1. **启动**：装配完所有元素、link 完、挂好 probe 和 bus 监听后，一次性 `pipeline.set_state(gst::State::Playing)`，失败用 `.context("the pipeline would not start")?` 包住；
2. **生命周期归调用方**：`start` 把 `gst::Pipeline` 返回给 `main`，`main` 用 `_pipeline` 持有它——drop 即停会话，shutdown 自然成立。注释原话：

> The pipeline is returned rather than kept here so the caller owns its lifetime: dropping it stops the session, which is what a shutdown should do.
>
> —— 管线返回给调用方持有生命周期，而不是自己藏着：drop 它就停会话，这正是 shutdown 该做的。

总线消息只处理 Error / Warning 两类并转发到 journal；EOS 和状态变更消息**故意忽略**（频率高、日常无用，真要看就 `GST_DEBUG` 开）。注意它没有实现 EOS 后的自愈/重启逻辑——那是 systemd `Restart=` 的事，`mediad` 的设计哲学是"不在恢复路径上"。

---

## 6. 动态 pad 管理

`tee` 的 src pad 是 **request pad**（`src_%u`），不是 always pad，所以两条视频分支不能和源链路一起 `link_many`，必须显式请求：

```rust
fn link_tee_branch(tee: &gst::Element, branch: &gst::Element) -> Result<()> {
    let src_pad = tee.request_pad_simple("src_%u")
        .ok_or_else(|| anyhow!("the tee would not give a source pad"))?;
    let sink_pad = branch.static_pad("sink")
        .ok_or_else(|| anyhow!("the branch has no sink pad"))?;
    src_pad.link(&sink_pad)...
}
```

这就是题目里说的"检测用 appsink 分支"的动态分叉点——虽然 tee 本身在启动时就固定插入（`main.rs` 注释："live 管线里后插 tee 是另一个难得多的问题"），两条分支的 pad 仍然是请求式的。没有运行时增删分支的逻辑。

---

## 7. 时钟同步与延迟控制

- **视频分支**：交给 webrtcsink 默认行为（直播时钟，congestion control 按链路调码率）；
- **原始分支**：appsink 显式 `sync=false`——快照要的是"最新帧一出现就给我"，pacing 只会给不渲染任何东西的消费者加延迟；
- **videotestsrc 源**：显式 `is-live=true`，让测试源像摄像头一样按时钟出帧，而不是跑赢时钟；
- **采集速率窗口**：`meter_capture_rate` 用 1 秒滑动窗口实测 fps，而不是信任 caps 上的理论值。

---

## 8. 测试要点（L1382–1430）

`#[cfg(test)] mod tests` 只有三个单测，全是纯逻辑、不依赖 GStreamer（这也是"可移植部分在笔记本上可测"哲学的体现）：

1. **`a_quarter_turn_swaps_the_frame_size`**：验证 `Rotation::output` 轴交换。为什么这个测试要命：

> Get this wrong and a consumer reads a 720x1280 picture as 1280x720 — which is not a failure, it is a diagonal smear, and the kind of thing that gets blamed on the camera.
>
> —— 错了不是报错，是斜着花屏，还会被怪到摄像头。

2. **`clockwise_is_90r_and_identity_is_nothing_at_all`**：验证 `video_direction` 映射（Cw90→`90r`、Cw270→`90l`、Cw180→`180`、None→`None`）。注释警告：方向名写反的话，控制台的"拖动看向"会把注视映射到同一几何上，180° 错误会让机器人看向操作员指向的反方向，而不只是画面侧着。另外断言 None 不是 `Some("identity")`——那样反而会白建一个元素。
3. **`only_right_angles_are_accepted`**：0/90/180/270 全过；45/89/91/360/1 全拒且报错信息包含 `"0, 90, 180 or 270"`。不做四舍五入，打错字就当场拒。

注意：**没有任何测试真正装配 GStreamer 管线**。管线正确性靠真机 bring-up 文档（`media-bringup.md`、`architecture.md`、`remote-webrtc.md`）和运行时观测保证，单测只兜住"纯函数不会算错"这部分。

---

## 9. 与其他模块的关系

```
                        ┌──────────────────────────────────────┐
                        │              main.rs                 │
                        │  解析 CLI → 加载 robotd.toml [media] │
                        │  → Producer::learn → pipeline::start │
                        └──────────────┬───────────────────────┘
                                       │ (source, producer, Settings)
                                       ▼
        ┌──────────────────── pipeline.rs（本文件）────────────────────┐
        │                                                              │
        │  videotestsrc/v4l2src → capsfilter(UYVY) → [videoflip?] → tee │
        │                          │                       ├─ queue → webrtcsink
        │                       probe(测速)                 │      │ 信令服务器(进程内)
        │                          │                       │      ├─ meta ← producer.rs
        │                          │                       │      ├─ encoder-setup → mpph264enc
        │                          │                       │      └─ consumer-added → Channel
        │                          │                       └─ leaky queue(1) → appsink
        │                          │                              │
        └──────────────────────────┼──────────────────────────────┼───┘
                                   │                              │
                          Frames (Arc<Mutex<Option<Frame>>>)  Channel{inbound,outbound}
                                   │                              │
                 ┌─────────────────┴──────────┐         ┌────────┴─────────────┐
                 ▼                            ▼         ▼                      ▼
          exposure.rs                   detect.rs   session::run        web.rs / 浏览器
     每 500ms inspect()            自己的线程(不抢    每行 JSON-RPC     console 读 video
     取 luma 均值，反写             tokio worker)，   转发到 upstream   通知，CSS 旋转
     sensor exposure/gain          读 latest() 整帧  Pool；视频几何
                                   letterbox_from_uyvy  由 Video{w,h,rotate} 告知
```

- **`producer.rs`**：`Producer::fields()` 产出 `name/serial/release/api_version` 四元组，本文件把它塞进 webrtcsink 的 `meta` structure，随信令服务器的 `list` 应答交给每个 peer——peer 在协商前就知道找到的是哪台机器人。查不到名字只是 warn，管线照起。
- **`session.rs`**：本文件产出的 `Channel{inbound,outbound}` 被 `main` 逐个喂给 `session::run`。session 是传输无关的"管子"，不解析应答。本文件还把"画面宽高 + 安装旋转角"通过 `session::Video` 告诉 session，session 用 `media.video` 调用/通知把 `rotate` 发给浏览器——因为**管线不再转像素**，浏览器必须自己知道相机歪了多少度。
- **`exposure.rs`**：拿 `Frames::inspect` 每 500ms 抽 luma，阻尼步进反写 sensor。它直接 `use crate::pipeline::{CAPTURE_FORMAT, Frame, Frames}`，是原始分支的第一个消费者。
- **`detect.rs`**：鸭子检测器，独立 OS 线程（推理 60ms 阻塞，不能占 tokio worker），拿 `Frames` 整帧，把旋转折进它本来就做的重采样（`letterbox_from_uyvy` + `Turn`）。
- **`config.rs` / `main.rs`**：`Settings` 从 `[media]` 段来；`--flip-in-pipeline` 才让 `Rotation` 非 None；`--rotate 90` 这个安装角同时被 `Rotation::from_degrees` 和 `duck_detect::Turn::from_degrees` 校验，任一个非法直接拒启动。
- **`web.rs`**：控制台页面由本进程另起 8080 端口服务，和 webrtcsink 的信令端口（默认 8443）是两个端口；web 起不来不影响视频和控制。
- **`route.rs` / `upstream.rs`**：本文件完全不碰；控制通道的 JSON-RPC 路由由 session 经 upstream Pool 转给后端服务。

---

## 10. 一页带走：这个文件教会我们的工程教训

1. **性能问题先怀疑零拷贝路径**：一个 `videoflip` 看似无害，实际打挂了 RGA，整机 97 °C、降频到 408 MHz、帧率掉到 8 fps。
2. **GStreamer 上"能跑"和"对"差得很远**：预编码喂 webrtcsink 能跑，但丢了拥塞控制和 PLI 关键帧。
3. **C 回调里 panic = 整个进程 abort**：所以先查信号/属性是否存在，再调；闭包里只做格式化和转发。
4. **tokio 跨线程要显式抓 Handle**：GStreamer 线程上没有 runtime，`tokio::spawn` 直接 panic。
5. **测量位置决定数据可信度**：测速必须在 tee 之前；丢帧必须看驱动序列号 offset，不能看自己 leaky 队列丢了多少。
6. **对配置/上游插件的漂移要防御性编程**：`has_property`、`SignalId::lookup`、`EnumClass` 解析失败都降级为 warn，绝不为一个设置项让机器人起不来。
7. **错误信息要区分病因**：`find_sensor` 三种失败三种修法，早期版本统一报错直接把人带沟里。
8. **跨平台边界用 `cfg(target_os)` 划清**：Linux only 的硬件绑定关在这一个文件里，crate 其余部分和单测保持可移植、可在笔记本上跑。
#（注：内容由AI生成）
