# `lib.rs` 解读

## 概述

`lib.rs` 是 `mediad` crate 的根文件（40 行），本身**没有任何可执行代码**——它只做三件事：

1. 用一段 crate-level doc comment 把整个守护进程的模块地图讲清楚；
2. 用 `pub mod` 声明所有公共模块；
3. 用 `#[cfg(target_os = "linux")]` 把平台相关的三个模块（`pipeline`、`exposure`、`detect`）隔离在 Linux 之外。

它在 crate 中的角色是**架构索引**：新人打开这个文件就能知道"mediad 到底由哪几块组成、哪些是可移植的、哪些是板子专属的"。文件头那段注释不是装饰，而是一份有意为之的"设计导览"，引用了多份外部设计文档（`remote-webrtc.md`、`webrtc-console.md` 等）的具体章节号，说明代码与设计文档是同步维护的。

## 模块地图（按 doc comment 顺序）

文件头注释把模块分成了三组：

### 第一组：控制通道（端到端，但通道本身还在建设）

```rust
//! What exists so far is the control channel, end to end but for the channel itself:
//! —— 目前已有的是控制通道，端到端打通，只差通道本身还没接进来：
//!
//! - [`route`] — which calls a WebRTC peer may make. `remote-webrtc.md` §5.
//!   —— WebRTC 对端允许发起的调用清单。
//! - [`upstream`] — connections to the five services that own the answers, one per (service, lane).
//!   —— 与五个"持有答案"的上游服务之间的连接，每个 (服务, 通道) 一条。
//! - [`session`] — the pipe. Lines in, lines out, replies never parsed.
//!   —— 管道。行进来，行出去，从不解析回复。
```

- **`route`**：定义对端（通过 WebRTC datachannel）能调用哪些方法。是 RPC 路由表。
- **`upstream`**：mediad 自己也是个客户端——它要去连 robotd 体系里的其他五个守护进程（configd 等）拿数据。这个模块管连接池，按 (service, lane) 维度各一条 socket。
- **`session`**：会话管道。注释特意强调"transport-agnostic on purpose"（刻意与传输层无关）——它吃行、吐行，不解析回复，因此**不需要 WebRTC 对端在场就能测试**，将来换成 WebSocket 表面（设计文档 §11）也不用改它。

### 第二组：可移植的辅助模块

```rust
//! - [`config`] — `[media]` in `robotd.toml`: what the stream is, edited with `robotctl configure`.
//!   —— robotd.toml 里的 [media] 段：流是什么，用 robotctl configure 编辑。
//! - [`web`] — the console page, served by the daemon it drives. `webrtc-console.md` §1.
//!   —— 控制台页面，由它所驱动的守护进程自己提供。
//! - [`producer`] — who this robot says it is, before a peer negotiates anything. §5.
//!   —— 这台机器人在对端开始任何协商之前先自报家门。
```

这三个模块**不碰 GStreamer、不碰板子硬件**，所以在 macOS / Windows 开发笔记本上能编译、能跑测试。这是 `config.rs` 把自己独立成模块的根本原因（详见 `config.rs` 解读）。

### 第三组：Linux-only 的核心媒体路径

```rust
//! [`pipeline`] is the rest, and the only part that is not portable: `webrtcsink` with the
//! signalling server in this process, `mpph264enc` in front of it, and a `control` datachannel
//! per peer wired to [`session::run`].
//! —— pipeline 是剩下的部分，也是唯一不可移植的：进程内自带信令服务器的 webrtcsink，
//! 它前面的 mpph264enc（瑞芯微 MPP 硬件 H.264 编码器），以及每个对端一条接到 session::run 的 control datachannel。
```

## 关键声明：模块与 cfg 门控

```rust
pub mod config;
pub mod producer;
pub mod route;
pub mod session;
pub mod upstream;
pub mod web;

/// The GStreamer pipeline and the datachannel. Linux only — see the crate manifest for why the
/// gate is by target rather than by feature.
/// —— GStreamer 管线与 datachannel。仅限 Linux——为什么用 target 门控而不是 feature，见 crate manifest。
#[cfg(target_os = "linux")]
pub mod pipeline;

/// Auto-exposure, in software, because the board's 3A engine does not do it. Linux only for
/// [`pipeline`]'s reason — it meters the frames the pipeline taps off the tee.
/// —— 软件自动曝光，因为板子的 3A 引擎不支持。仅限 Linux，原因与 pipeline 相同——它计量的是 pipeline 从 tee 上
/// 分叉出来的那些帧。
#[cfg(target_os = "linux")]
pub mod exposure;

/// Looking for other ducks in the frames on the tee. Linux only for [`exposure`]'s reason: it
/// reads the same raw branch, in the same pixel format the pipeline names.
/// —— 在 tee 上的帧里寻找其他鸭子。仅限 Linux，原因与 exposure 相同：它读同一条 raw 分支，
/// 像素格式也是 pipeline 指定的那种。
#[cfg(target_os = "linux")]
pub mod detect;
```

设计要点：

- **为什么用 `target_os` 而不是 Cargo feature 来门控**：注释明确指向 `Cargo.toml`（crate manifest）。这里隐含的理由是——feature 门控意味着"在笔记本上也能选择开启 pipeline"，但 pipeline 依赖 GStreamer、MPP、`webrtcsink`、NPU 运行时，这些在 macOS 上根本装不上；强行用 feature 会让"笔记本编译"变成一件需要额外开关的麻烦事。直接按操作系统门控，笔记本上 `cargo build` 永远成功，Linux 板子上自动获得全部媒体能力。
- **`exposure` 和 `detect` 的 Linux-only 理由是"传递"的**：它们不直接依赖 GStreamer，但它们**消费 pipeline 从 tee 上分叉出的原始帧**——像素格式、缓冲池、时间戳都是 pipeline 定义的。在没有 pipeline 的平台上，这两个模块没有意义。注释里"Linux only for [`pipeline`]'s reason"和"Linux only for [`exposure`]'s reason"是一种**继承式论证**：只要 pipeline 是 Linux-only，下游消费者天然也是。
- **软件自动曝光（`exposure`）的存在理由**：注释一句话点出——"because the board's 3A engine does not do it"（板子的 3A 引擎不做这件事）。3A = Auto Focus / Auto Exposure / Auto White Balance。这块瑞芯微板子的硬件 ISP 3A 管线缺自动曝光，所以只能在软件层自己测光、自己调增益。这解释了为什么 `exposure` 要"meter the frames the pipeline taps off the tee"——它必须看到原始未编码帧才能测光。
- **`detect` 找的是"other ducks"**：目标检测模型的训练目标是"在画面里识别出别的 Microduck 机器人"，而不是通用物体检测。它读 tee 上的同一条 raw 分支，像素格式与 pipeline 命名一致——这意味着 detector 和编码器共享同一个缓冲池，零拷贝。

## 重要常量

本文件**没有定义任何常量**。它唯一的"常量"是模块划分本身：

- **永远可移植**：`config`、`producer`、`route`、`session`、`upstream`、`web`。
- **仅 Linux**：`pipeline`、`exposure`、`detect`。

这种划分本身就是最值得记住的事实——它决定了在开发笔记本上能跑哪些测试、在 CI 上能编译哪些 target。

## 测试要点

`lib.rs` 本身没有 `#[cfg(test)]` 模块。它的"测试"是间接的：

- 可移植模块的测试在 macOS/Windows CI 上跑；
- Linux-only 模块的测试在板子或 Linux CI 上跑。

这种分层本身就是一种测试策略——`config.rs` 文件头那句"`main` is Linux-only, so anything living there is not compiled — let alone tested — on the machine it is written on"（`main` 是 Linux-only 的，任何放在里面的代码在写代码的机器上根本不会被编译，更不会被跑测试）正是在解释为什么可移植性这么重要：**写代码的人用的是笔记本，能在笔记本上跑的测试才会被跑**。

## 与其他模块的关系

- **`main.rs`**（不在本文件中声明，因为是二进制 crate 的入口）：作为可执行文件入口，组装所有模块。它是 Linux-only 的，所以不能放任何可移植逻辑。
- **`route` / `upstream` / `session`**：控制通道铁三角。`session::run` 是传输无关的行管道，被 pipeline 里的 datachannel 接线；`route` 定义 session 允许调的方法；`upstream` 是 session 拿去访问其他守护进程的出口。
- **`config`**：被 `main` 调用，产出 `Params` 喂给 `pipeline` 和 `detect`。
- **`web`**：被 `main` 拉起，独立监听一个端口，与视频管线解耦。
- **`producer`**：在 pipeline 启动前调用，把"我是谁"填进 `webrtcsink` 的 meta 结构。
- **`pipeline`**：唯一不可移植的模块，是 mediad 的心脏——GStreamer `webrtcsink` + `mpph264enc`，每个对端一条 control datachannel 接到 `session::run`。
- **`exposure` / `detect`**：消费 pipeline 的 tee 分叉，一个测光、一个找别的鸭子。

## 设计哲学小结

`lib.rs` 用 40 行回答了三个问题：

1. **mediad 是什么**：摄像头、麦克风、WebRTC、远程网关的守护进程。
2. **哪些代码必须在板子上**：碰 GStreamer / MPP / NPU / tee 缓冲的三个模块，按 `target_os = "linux"` 门控。
3. **哪些代码必须在笔记本上也能跑**：配置、Web 控制台、生产者身份、路由、上游连接池、会话管道——这些是"控制面"，必须可测。

这种"**控制面可移植、数据面 Linux-only**"的切分，是整个 crate 能在工程师笔记本上快速迭代、又能在板子上跑硬件加速媒体管线的关键。文件头引用的设计文档章节号（`remote-webrtc.md §5`、`webrtc-console.md §1` 等）则说明：这份模块地图不是拍脑袋分的，每一刀都有设计文档背书。
#（注：内容由AI生成）
