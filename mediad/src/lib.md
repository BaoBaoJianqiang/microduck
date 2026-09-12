# lib.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\lib.rs`

## 核心设计决策

`mediad` 是摄像头、麦克风、WebRTC 与远程网关的集合体。当前已实现的是控制通道端到端，而视频通道本身尚缺。模块组织如下：

- **`route`**：WebRTC 对等端可以发起哪些调用（`remote-webrtc.md` §5）。
- **`upstream`**：到五个持有答案的服务的连接，每（服务，车道）一个。
- **`session`**：管道；行进、行出，回复从不被解析。
- **`config`**：`robotd.toml` 中的 `[media]`，由 `robotctl configure` 编辑。
- **`web`**：由驱动它的守护进程自身提供的控制台页面。
- **`producer`**：在对等端协商任何东西之前，本机器人宣告自己是谁。
- **`pipeline`**（仅 Linux）：`webrtcsink`，信令服务器在本进程内，`mpph264enc` 在其前，每个对等端一个 `control` 数据通道，接入 `session::run`。

关键设计意图：`session::run` 刻意与传输无关——它只接收行并返回行，因此无需 WebRTC 对等端即可测试，且未来若接 WebSocket 表面也无需改动。

`pipeline`、`exposure`、`detect` 三个模块以 `cfg(target_os = "linux")` 门控，而非 feature 开关：让 `cargo test` 在笔记本（非 Linux）和板上都能诚实编译；feature 默认关闭的模块等于没人编译。

## 模块声明

| 模块 | 平台 | 职责 |
|---|---|---|
| `config` | 全平台 | `[media]` 与 `[detect]` 配置加载 |
| `producer` | 全平台 | 生产者身份（name/serial/release/api_version） |
| `route` | 全平台 | WebRTC 传输下的调用许可表 |
| `session` | 全平台 | 单个对等端的控制通道管道 |
| `upstream` | 全平台 | 到五个服务的连接池 |
| `web` | 全平台 | 控制台 HTTP 服务 |
| `pipeline` | Linux | GStreamer 管线 + 数据通道 |
| `exposure` | Linux | 软件自动曝光 |
| `detect` | Linux | 画面中的鸭子检测 |

## 关键摘要

- 本 crate 把"谁应答一个调用、应答会占用连接多久"放在 `proto::Call::destination` 一次，`route` 只管"这个传输上是否允许"。
- `pipeline`/`exposure`/`detect` 全部依赖 GStreamer/V4L2，故只在 Linux 编译；其余部分（路由、会话、上游、控制台）可在笔记本上测试。
