# `main.rs` 解读

## 概述

`main.rs` 是 mediad 的二进制入口（`src/main.rs`），负责把 mediad 这个库 crate 组装成一个可运行的 Linux 守护进程。它承担：

- 命令行参数解析（`clap`）；
- 日志初始化（`tracing` + stderr）；
- tokio 异步运行时构建；
- 配置文件加载（与 `robotd` 共享 `/etc/robot/robotd.toml`）；
- 各子系统按依赖顺序初始化：Web 控制台 → producer 身份 → GStreamer 管线 → 自动曝光 loop → duck 检测器 → WebRTC 会话循环；
- 优雅地接受 peer、为每个 peer 起一个任务；
- 非 Linux 平台上的 stub（让 crate 其余部分可在任意平台编译跑测试）。

文件头注释明确两个**有意为之的非目标**：

```rust
// It does not authenticate. Anyone who reaches the signalling port can drive the robot and see
// its camera. That is a decision, not an omission
// -- 它不做认证。任何能到达信令端口的人都能驾驶机器人、看到它的摄像头。这是决定，不是疏漏。
```
配对 PIN 是共享的 `000000`，加门只会给每次连接加一步、却什么都证明不了。让机器人从 LAN 外可达的 bridge 在会话到达前双向认证。

```rust
// It is not on the recovery path. If mediad will not start, the robot still walks, still takes
// an update, and is still reachable over Bluetooth.
// -- 它不在恢复路径上。mediad 起不来，机器人照样走、照样收更新、照样能通过蓝牙联系上。
```
这就是为什么它可以依赖 release asset 里的 plugin 和设备节点组，而 `updaterd` 不行。

---

## 命令行参数（`Args`，clap Parser）

| 参数 | 默认值 | 含义与设计理由 |
|---|---|---|
| `--host` | `0.0.0.0` | 信令服务器绑定地址。默认所有接口，**这是要点**：只绑 loopback 意味着 LAN 上的 peer 根本连不上，每次会话都得过 bridge，那就没必要有本地模式了。 |
| `--port` | `8443` | 信令端口。与 `webrtcsink` 自带 signaller 默认一致，客户端不用传参。 |
| `--web-port` | `8080` | 控制台 HTTP 端口。**两个端口，但用户只会打这一个**：`webrtcsink` 拥有 `--port` 上的 listener，只告诉它 host 和 port，页面不能作为它上面的 route。未来麦克风或浏览器手柄需求出现时，会合并成一个端口 + 自己的信令服务器 + 证书。 |
| `--config` | `/etc/robot/robotd.toml` | 参数文件。与 `robotd` 读同一个文件，`[media]` 段是本守护进程的段。`robotctl configure` 编辑这个文件。曾几何时这些是 `ExecStart` 上的 flag，release installer 重写它，改一个就要 systemd drop-in——没人会为了回答"为什么视频模糊"去翻 drop-in。 |
| `--camera-device` | `/dev/video0` | rkisp 暴露多个节点，`video0` 是主路径。 |
| `--exposure` | `600` | 传感器初始曝光（行数，每行 ≈19µs）。驱动启动值下画面是黑的而非只是暗，所以第一帧前必须写传感器。装了 3A 引擎的板子会从这里收敛；没装的就一直保持这个值。 |
| `--analogue-gain` | `1024` | 初始模拟增益，256=1x。 |
| `--rotate` | `90` | 相机相对竖直方向顺时针偏转角（0/90/180/270）。**默认 90，因为头部相机物理上转了四分之一圈**，这是唯一写下这个事实的地方。它不再意味着"旋转像素"——而是告诉显示方，由显示方免费旋转（控制台用 CSS transform 在 GPU 上做）。在这里旋转曾花掉 145% 一个核、掉到 22fps。 |
| `--no-auto-exposure` | false | 关闭软件 AE loop。loop 默认开，因为板上 3A 引擎只做一次。用于固定曝光的标定拍摄，或某块板子的引擎真的会持续收敛。 |
| `--flip-in-pipeline` | false | 在管线内旋转，让**编码后的流**就是正的。**默认关**：它会打断 `mpph264enc` 到 SoC 2D 引擎的零拷贝路径，MPP 每帧都软件转；实测 97°C、CPU 降频到 408MHz、30fps 相机只出 8fps。只有消费者自己不能旋转时才开。 |

---

## `main()` 的启动顺序（Linux 分支）

```rust
#[cfg(target_os = "linux")]
fn main() -> ExitCode { ... }
```

### 1. 解析参数 + 初始化日志

```rust
let args = Args::parse();
tracing_subscriber::fmt()
    .with_env_filter(...)   // 读 RUST_LOG，缺省 info
    .with_writer(std::io::stderr)
    .init();
```

### 2. 打印启动身份

```rust
// Before anything that can fail, so a journal that reports a startup failure also reports
// which build failed. Every other daemon does this for the same reason.
// -- 在任何可能失败的事情之前，这样 journal 报告启动失败时也报告了是哪个构建失败的。
//    其他所有守护进程都因为同一原因这么做。
duck_ipc_proto::log_startup_identity!("mediad");
```

### 3. 构建 tokio 运行时

```rust
let runtime = match tokio::runtime::Runtime::new() { ... };
```
`Runtime::new()` 是多线程 runtime（`multi_thread` flavor）。失败直接 `ExitCode::FAILURE`。

### 4. 提前拒绝坏的旋转角

```rust
// Refused before anything starts: a bad angle is a typo on a command line, and the daemon
// should say so rather than opening a camera first.
// -- 在任何事开始之前拒绝：坏角度就是命令行上的笔误，守护进程应该直接说，而不是先去开摄像头。
// Validated even when the pipeline will not use it, because it is still what every consumer
// is told about the mount
// -- 即使管线不会用它也校验，因为它仍然是每个消费者被告知的安装角度——笔误不该以"没人能应用的旋转"形式到达控制台。
let mount = mediad::pipeline::Rotation::from_degrees(args.rotate)?;
```

### 5. 加载配置

```rust
let explicit = args.config.is_some();
let config = args.config.clone().unwrap_or_else(mediad::config::default_path);
let params = mediad::config::load(&config, explicit);
let (media, detect) = (params.media, params.detect);
```
一次读一个文件：`[detect]` 也是 mediad 的段（机器人的开关只该有一个地方）。然后打 info 日志：camera、quality label、width/height/fps、bitrate、congestion_control。

### 6. 把角度翻译成检测器的语言

```rust
let turn = duck_detect::Turn::from_degrees(args.rotate)?;
```
检测器把旋转折进它本来就要做的重采样里，所以管线里什么都不用做。

### 7. 决定是否管线内旋转

```rust
let rotation = if args.flip_in_pipeline {
    tracing::warn!(...);   // 警告：会失去编码器零拷贝
    mount
} else {
    mediad::pipeline::Rotation::None
};
```

### 8. 进入 `runtime.block_on(async move { ... })`

#### 8.1 先起 Web 控制台（在管线之前）

```rust
// The console, before the pipeline: it is the page that says a robot's pipeline would not start,
// so it should be up first — and it needs nothing from GStreamer.
// -- 控制台在管线之前：它就是那个说"机器人管线起不来"的页面，所以它应该先起来——而且它不依赖 GStreamer。
```
关键容错：

```rust
// A page that cannot be served does not cost the video. A refused bind is almost always a port
// already in use, which Restart=always cannot fix by trying again; a robot that streams and
// answers control calls with no console is much better than one that does neither.
// -- 一个服务不起来的页面不该搭上视频。bind 被拒几乎总是端口被占，Restart=always 重试也修不好；
//    一个能推流、能应答控制调用、只是没控制台的机器人，比两个都不行的机器人好得多。
```
所以 bind 失败只打 error，守护进程继续跑。

#### 8.2 学 producer 身份（在管线之前）

```rust
// Before the pipeline, because webrtcsink's meta is set as the element is built and a producer
// that registered without a name would keep it until this daemon restarts.
// -- 在管线之前，因为 webrtcsink 的 meta 是元素构建时设置的，没带名字注册的 producer 会一直保持那个状态
//    直到守护进程重启。
```
这是一次到 `configd` 的 unix-socket 往返；`configd` 可能还没起，所以有界、失败只打 warning 不退出。

#### 8.3 构造 `Source` 与 `Settings`

```rust
let source = if media.camera {
    Source::Camera(Camera { device, exposure, analogue_gain })
} else {
    Source::Test
};
```
帧尺寸/码率/帧率仍然是 pinned 而非协商——tee 的两个分支都依赖这个答案，消费者若得猜，源一变就第一次猜错。变的只是数字来源：config 里一个命名 quality，而非三个没人设得动的 flag。

#### 8.4 启动管线

```rust
let (_pipeline, mut channels, frames) = mediad::pipeline::start(source.clone(), &producer, &settings)?;
```
失败直接 `ExitCode::FAILURE`——错误信息里写明哪一步失败、通常原因是什么（缺 plugin、缺库、或设备节点打不开）。

`frames` 是 tee 上的 raw tap：AE loop 用它测光，`architecture.md §5.3` 里的 `get_frame` surface 是它其余的用途。分支从一开始就在，而不是后加——往活管线里插 tee 是另一个难看得多的问题。

#### 8.5 启动自动曝光（管线之后）

```rust
// After the pipeline, because it meters the pipeline's own frames — and only with a real camera,
// since a test pattern has no sensor to write and the loop would spend the daemon's life
// reporting that it cannot.
// -- 在管线之后，因为它测的是管线自己的帧——而且只在真摄像头下起，因为测试图案没有传感器可写，
//    loop 会把守护进程一生都花在报告"我不行"上。
```
`_exposure` 是停止句柄，活在这个作用域里，也就是守护进程一生。

#### 8.6 启动 duck 检测器（可选，失败不致命）

```rust
// A detector that was asked for and cannot start is a warning, not a failure: the camera, the
// console and the control channel are all still worth having, and "mediad refused to boot
// because a model file moved" is a bad trade.
// -- 被要求但起不来的检测器是 warning，不是失败：摄像头、控制台、控制通道仍然值得有，
//    "因为模型文件挪了位置 mediad 拒绝启动"是个糟糕的交易。
```
注意 `sampler_turn`：若管线内 flip 了，tee 上的帧已经是正的，采样器不能再转一次；否则按 `turn` 转。

#### 8.7 为每个 peer 起会话

```rust
while let Some(channel) = channels.recv().await {
    ...
}
```

- 每个 peer 一个 mpsc channel（256 容量）给上游回复；
- 一个 `upstream::Pool`，每个 peer 自己的连接——这样一个 peer 长达数分钟的更新不会让另一个的 telemetry 沉默；
- 主动 push 一次 `video_notification`（best-effort，可能在 datachannel 还没开时就到，会被丢）；控制台是主动 `media.video` 拉的；
- 若检测器在，订阅 `sightings` broadcast，把检测结果作为通知推给 peer——与 `robot.state` 同一条通道，无轮询，每 peer 一个订阅者，慢消费者拖住检测器。遇到 `Lagged(_)` 就跳过——只有最新的 sighting 有价值，这正是有界 channel 的用途；`Closed` 则退出任务。
- `tokio::spawn(mediad::session::run(...))` 跑每个 peer 的会话。

#### 8.8 循环退出

```rust
// The pipeline outlived its consumers, which means webrtcsink stopped producing them.
// -- 管线比它的消费者活得更久，意味着 webrtcsink 不再产生消费者了。
tracing::warn!("no longer accepting peers");
ExitCode::FAILURE
```

---

## 非 Linux stub

```rust
#[cfg(not(target_os = "linux"))]
fn main() -> ExitCode {
    let _ = Args::parse();
    eprintln!("mediad runs on the robot; this host is not Linux");
    ExitCode::FAILURE
}
```
mediad 是 Linux 守护进程：对着 Rockchip VPU 和 V4L2 采集路径驱动 GStreamer。crate 其余部分是可移植的、测试在任何地方都能跑——所以这是一个 stub，而不是在整个 crate 上加 `cfg`。

---

## 设计模式与踩坑点汇总

### 初始化顺序里的依赖逻辑

| 顺序 | 子系统 | 为什么是这个位置 |
|---|---|---|
| 1 | 日志 | 任何后续失败都要有东西记 |
| 2 | `log_startup_identity` | 在任何可能失败的事之前，便于 journal 区分构建 |
| 3 | tokio runtime | 后面所有 `.await` 的底座 |
| 4 | 校验 `--rotate` | 笔误要早拒绝，别先去开摄像头 |
| 5 | 读 config | 后续所有子系统都要它的值 |
| 6 | Web 控制台 | 它是报"管线起不来"的页面，得先在；且它不依赖 GStreamer；bind 失败不致命 |
| 7 | producer::learn | webrtcsink 的 meta 在元素构建时设置，晚了就带错名字 |
| 8 | GStreamer 管线 | 核心依赖 |
| 9 | AE loop | 要管线的帧；只在真摄像头下起 |
| 10 | duck 检测器 | 失败不致命；要管线的帧 |
| 11 | peer 会话循环 | 所有基础设施就绪后开始服务 |

### 容错哲学

- **Web bind 失败 → error 继续**：视频不能因为没控制台而没了。
- **producer learn 失败 → warning 继续**：unix socket 往返，`configd` 可能还没起。
- **检测器起不来 → warning 继续**：模型文件挪位置不该让整个守护进程拒启动。
- **管线起不来 → 直接 FAILURE**：这是核心功能，没有它 mediad 什么都不是。
- **旋转角非法 → 直接 FAILURE**：笔误，越早说越好。

### 优雅退出

- 没有显式的信号处理逻辑在 `main.rs` 里——依赖 systemd 的 `SIGTERM`/`SIGINT`。tokio runtime 被 drop 时，所有 `tokio::spawn` 的任务被取消；AE 线程的 `Stop(Arc<AtomicBool>)` 虽然不被显式触发，但进程退出会带走它。
- `_exposure` 用下划线前缀绑定，明确表示"只是为了在作用域里活着"。
- 会话循环退出意味着 `webrtcsink` 不再产生 peer，这是异常情况，打 warning 并以 FAILURE 退出——让 systemd `Restart=always` 重启。

### 与其他模块的关系

- **`mediad::pipeline`**：GStreamer 管线封装，`Source`/`Camera`/`Settings`/`Rotation`/`start()`。
- **`mediad::web`**：控制台 HTTP 服务（`page()`、`serve()`）。
- **`mediad::producer`**：向 `configd` 学习 producer 名字、release、API 版本。
- **`mediad::config`**：读 `[media]` 和 `[detect]` 段。
- **`mediad::exposure`**：软件 AE loop（见 `exposure.rs解读.md`）。
- **`mediad::detect`**：duck 检测器，`spawn_first`、`notification`。
- **`mediad::session`**：每个 peer 的会话，`Video`、`video_notification`、`run`。
- **`mediad::upstream::Pool`**：到 robot API 的上游连接池，每 peer 一个。
- **`duck_ipc_proto`**：跨 crate 的启动身份宏、build info 宏。
- **`duck_detect`**：检测器 crate，`Turn` 类型。

---

## 一句话总结

`main.rs` 本身没有算法，它是**一份"启动顺序的论证"**：每个子系统在哪一步起、为什么不能更晚、为什么失败时继续还是退出，都写在注释里。整份文件读下来，最重要的信息不是代码做了什么，而是**为什么是这个顺序、为什么有的失败是 error 有的只是 warning**——这些都是在机器人现场被各种"启动顺序竞态"和"端口被占"教训出来的。
#（注：内容由AI生成）
