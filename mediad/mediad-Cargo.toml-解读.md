# mediad Cargo.toml（媒体守护进程）解读与架构梳理

> 分析对象：`mediad/Cargo.toml`（73 行），microduck 机器人的媒体守护进程 crate——摄像头、麦克风、WebRTC 与远程网关。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `mediad`（media daemon）的构建清单——运行在机器人上的媒体管线守护进程，负责摄像头采集、音频、WebRTC 推流和远程控制台。它是整个 workspace 中**第一个引入 GStreamer（第二个 C 依赖）的 crate**，需要专用交叉编译 sysroot。

**关键特征**：
- **平台门控**：GStreamer 依赖在 `[target.'cfg(target_os = "linux")'.dependencies]` 下——笔记本上 `cargo test` 不需要 GStreamer，板上才编译。
- **生产依赖最小化**：axum 关默认特性（只开 http1+tokio），不引入 JSON extractor/multipart/tracing 中间件。
- **复用已有依赖**：用 axum（而非手写 HTTP 解析器）因为它已在 Cargo.lock 中（updater crate 的测试镜像用）。
- **跨 crate 复用**：依赖 duck-detect（NPU 检测）、robotd-params（配置 schema）、duck-ipc-proto（IPC 协议）。
- **GStreamer 版本策略**：`v1_22` 是 gst-plugins-rs 自己声明的下限，不是板上碰巧的版本（板上 1.26.2）——这样构建失败会报真实不兼容而非"某台机器人包版本意外"。

**与之前分析的关联**：
- 依赖 `duck-detect`（我们分析过其 Cargo.toml 和 duck-bench.rs）——mediad 调用检测能力。
- 配置来自 `/etc/robot/robotd.toml` 的 `[media]` 和 `[detect]` 段——与 robotd-params schema 共享。
- 这是板上第二个 C 依赖（第一个是 padd 的 libudev/gilrs）。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 Cargo.toml |
| package name | mediad |
| description | "Camera, mic, WebRTC — and the remote gateway" |
| 版本 | workspace 统一 |
| edition/license | workspace 继承 |
| 行数 / 已读范围 | 73 行，全文已读 |
| 类型 | 二进制 crate（daemon），非 lib |
| 运行平台 | Linux only（GStreamer 管线） |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | package name="mediad" | L2 | 媒体守护进程 | confirmed |
| F2 | description 提到 Camera/mic/WebRTC/gateway | L6 | 四个职责 | confirmed |
| F3 | GStreamer 是 Linux-only target dep | L47 | cfg(target_os="linux") | confirmed |
| F4 | 注释称这是"第二个 C 依赖" | L13 | 第一个是 libudev/gilrs in padd | confirmed |
| F5 | cross-sysroot.sh 存在为 GStreamer | L15 | scripts/cross-sysroot.sh | confirmed |
| F6 | 依赖 duck-ipc-proto（path） | L19 | IPC 协议 | confirmed |
| F7 | 依赖 robotd-params（path） | L23 | 配置 schema 共享 | confirmed |
| F8 | 依赖 duck-detect（path） | L26 | NPU 检测+ONNX fallback+decoder | confirmed |
| F9 | axum 0.8.9，default-features=false | L35 | 只开 http1+tokio | confirmed |
| F10 | axum 已在 Cargo.lock（updater 用） | L28 | 已知工具链兼容 | confirmed |
| F11 | gstreamer 0.24 + v1_22 | L53 | 下限是插件声明的 floor | confirmed |
| F12 | 板上 GStreamer 版本 1.26.2 | L50 | 注释中提到 | confirmed |
| F13 | gstreamer-app (appsink) 用于 NV12 原始帧 | L56 | 不解码已编码帧 | confirmed |
| F14 | gstreamer-video (GstVideoMeta) | L61 | v4l2src 零拷贝 | confirmed |
| F15 | gstreamer-webrtc 用于 data channel | L57 | 类型安全 GstWebRTCDataChannel | confirmed |
| F16 | glib 0.21 | L62 | GStreamer 依赖 | confirmed |
| F17 | tokio features: rt/multi-thread/net/io-util/sync/time/macros | L38 | WebRTC 异步运行时 | confirmed |
| F18 | clap derive | L36 | CLI 参数 | confirmed |
| F19 | dev-dep: tempfile 3 | L71 | 假 daemon 在真实 unix socket 上测试 | confirmed |
| F20 | dev-dep: duck-ipc-proto test-support feature | L69 | every_call() 穷举测试 | confirmed |

---

## 四、依赖架构详解

### 4.1 生产依赖（跨平台部分）

| 依赖 | 版本 | 关键特性 | 作用 |
|------|------|----------|------|
| duck-ipc-proto | path | — | IPC 协议定义（与其他 daemon 通信） |
| robotd-params | path | — | /etc/robot/robotd.toml 的 [media]/[detect] schema |
| anyhow | 1 | — | 错误处理 |
| duck-detect | path | — | 检测引擎（letterbox+turn、NPU 绑定、ONNX fallback、decoder） |
| axum | 0.8.9 | default-features=false, http1, tokio | 控制台单一路由（GET 一个文件） |
| clap | workspace | derive | CLI 参数解析 |
| serde_json | workspace | — | JSON |
| tokio | workspace | rt, rt-multi-thread, net, io-util, sync, time, macros | 异步运行时 |
| tracing | workspace | — | 日志 |
| tracing-subscriber | workspace | env-filter | 日志过滤 |

### 4.2 Linux-only 依赖（GStreamer 管线）

| 依赖 | 版本 | 特性 | 作用 |
|------|------|------|------|
| gstreamer | 0.24 | v1_22 | GStreamer 核心绑定 |
| gstreamer-app | 0.24 | v1_22 | appsink（从 tee 取 NV12 原始帧） |
| gstreamer-webrtc | 0.24 | v1_22 | WebRTC data channel（类型安全） |
| gstreamer-video | 0.24 | v1_22 | GstVideoMeta（v4l2src 零拷贝） |
| glib | 0.21 | — | GLib 基础类型 |

### 4.3 开发依赖

| 依赖 | 特性 | 作用 |
|------|------|------|
| duck-ipc-proto | test-support | every_call() 穷举测试列表 |
| tempfile | 3 | 真实 unix socket 上的假 daemon |
| tokio | rt-multi-thread, macros | 异步测试 |

---

## 五、设计决策深度解读

### 5.1 为什么按 target 门控而非 feature gate

注释（L8–L11）明确说：
> "Gating by target rather than by feature keeps `cargo test` honest on both — a feature that is off by default is a module nobody compiles."

**含义**：如果用 `#[cfg(feature = "gstreamer")]` 且默认关，那么笔记本上跑 `cargo test` 时 GStreamer 代码根本不编译——错误不会被发现。用 `cfg(target_os = "linux")` 意味着：
- 笔记本上：非 GStreamer 部分（路由表、session pipe、upstream pool）正常编译测试。
- 板上：GStreamer 部分编译。
- **跨平台代码始终被编译**，不会因为 feature 默认关而成为死代码。

### 5.2 为什么选 axum 而非手写 HTTP

注释（L27–L31）解释：
> "The alternative was a hand-rolled HTTP/1.1 responder: about sixty lines of hand-written request parsing bound to 0.0.0.0, written to avoid a dependency the build already resolves, where `hyper` underneath this is the most-read implementation of that parser in the language."

**权衡**：
- 手写 60 行 HTTP 解析器绑定 0.0.0.0 → 安全风险大。
- axum/hyper 是 Rust 生态最受审查的 HTTP 解析器。
- axum 已经在 Cargo.lock 中（updater crate 的测试镜像用它），所以不增加新的依赖树。

**但为了精简**：关了 default-features，只开 http1+tokio——因为这个控制台只服务一个 GET 文件，不需要 JSON extractor、multipart、forms、tracing 中间件。

### 5.3 为什么 GStreamer 版本下限是 v1_22

注释（L48–L52）：
> "`v1_22` on both, which is not a guess: it is the floor `gst-plugins-rs`'s own webrtc crate declares, and it is what gates `WebRTCDataChannel` into existence. The board runs 1.26.2, so this is comfortably satisfied — and pinning the floor at what the plugin requires rather than at what the board happens to have means a build failure here would name a real incompatibility rather than an accident of one robot's package versions."

**版本策略**：
- 下限不是"板上碰巧有什么"，而是"代码需要什么"。
- WebRTCDataChannel 类型需要 GStreamer ≥ 1.18，gstreamer-webrtc 插件声明 floor 是 1.22。
- 板上 1.26.2 远超下限。
- 如果未来有人在旧版 GStreamer 上构建，会得到清晰的版本错误，而非"这台机器人的包版本意外"。

### 5.4 为什么需要 gstreamer-video（GstVideoMeta）

注释（L58–L61）：
> "`GstVideoMeta`, which the camera source has to advertise in the ALLOCATION query or v4l2src copies every frame — see `advertise_video_meta`. Typed, so the meta API type comes from the bindings rather than a hand-written GType lookup."

**性能含义**：如果相机源不在 ALLOCATION 查询中声明 GstVideoMeta，v4l2src 会**每帧复制**——在嵌入式板上这是巨大的性能损失。用 typed binding 而非手写 GType 查找，编译期保证类型安全。

### 5.5 为什么需要 appsink 而非从编码分支取帧

注释（L54–L56）：
> "`appsink`, for the raw NV12 branch off the tee — perception and `get_frame` both want pixels, and taking them off the encoded branch would mean decoding what was just encoded."

**架构**：GStreamer pipeline 用 tee 分出两路——一路编码送 WebRTC，一路 appsink 取原始 NV12 给感知。从编码分支取帧意味着"编码后再解码"，浪费 CPU。

### 5.6 为什么需要 gstreamer-webrtc（而非 untyped glib::Object）

注释（L44–L46）：
> "`gstreamer-webrtc` is here for the data channel: a `GstWebRTCDataChannel` is what `create-data-channel` returns, and reaching for untyped `glib::Object` calls to avoid one dependency would be a poor trade."

**类型安全权衡**：用 untyped glib::Object 调用可以省一个依赖，但失去类型安全。data channel 的 API 足够复杂，类型安全值得这一个依赖。

### 5.7 为什么配置 schema 共享

注释（L20–L22）：
> "The same crate `robotd` parses that file with and the same one `robotctl configure` edits it through, so the schema, the defaults and the editor cannot drift from what is read here."

**单一事实源**：
- robotd-params 定义 `/etc/robot/robotd.toml` 的 schema。
- robotd（守护进程管理器）解析它。
- robotctl（命令行工具）编辑它。
- mediad 读取 `[media]` 和 `[detect]` 段。
- 四方共享同一个 schema crate，不会漂移。

---

## 六、跨 crate 依赖关系

```
mediad
├── duck-ipc-proto  (path)  ← IPC 协议，与其他 daemon 对话
├── robotd-params   (path)  ← 配置 schema，robotd/robotctl/mediad 共享
├── duck-detect    (path)  ← NPU 检测引擎（我们分析过）
│   ├── libloading → libonnxruntime.so（运行时 dlopen）
│   └── ort = "=2.0.0-rc.11"
├── axum 0.8.9               ← 控制台 HTTP（已在 Cargo.lock）
├── tokio                    ← 异步运行时
├── clap                     ← CLI
├── tracing                  ← 日志
└── [Linux only]
    ├── gstreamer 0.24 (v1_22)
    ├── gstreamer-app        ← appsink NV12
    ├── gstreamer-webrtc     ← WebRTC data channel
    ├── gstreamer-video      ← GstVideoMeta 零拷贝
    └── glib 0.21
```

---

## 七、效果主张与责任闭合卡（EFFECT）

### 7.1 主张："GStreamer 只在 Linux 上编译"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | GStreamer 相关 5 个依赖 | L47–L62 |
| 触发者 | cargo build / test | — |
| 当前装配/选择/开关 | `[target.'cfg(target_os = "linux")'.dependencies]` | L47 |
| 实际执行者 | Cargo 按 target 过滤依赖 | — |
| 成功副作用与观察点 | 笔记本 test 不链接 GStreamer；板上链接 | L8–L11 注释 |
| 失败是否返回且被检查 | 笔记本上引用 GStreamer 类型→编译错误 | 编译期 |
| 不能覆盖的对象 | 跨平台代码（路由表/session pipe/upstream pool）仍在普通 [dependencies]，始终编译 | — |
| status | **confirmed** | — |

### 7.2 主张："mediad 通过 robotd-params 读取配置，不自己解析 TOML"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | /etc/robot/robotd.toml [media]/[detect] | L20–L23 |
| 触发者 | mediad 启动 | — |
| 当前装配/选择/开关 | robotd-params path 依赖 | L23 |
| 实际执行者 | robotd-params::load | — |
| 成功副作用与观察点 | mediad/robotd/robotctl 三方 schema 一致 | L21–L22 注释 |
| 失败是否返回且被检查 | 配置缺失→fallback to defaults | L66 dev-dep 注释 |
| 不能覆盖的对象 | 非 [media]/[detect] 段由其他 daemon 读取 | — |
| status | **confirmed**（注释明确说明三方共享） | — |

---

## 八、边界与反例（BREAK）

1. **没有 openssl/rustls**：axum 只开了 http1，没有 TLS。WebRTC 本身有 DTLS/SRTP，但 HTTP 控制台是明文的。
2. **没有硬件抽象层**：GStreamer v4l2src 直接绑定相机——如果换相机驱动，需要改 pipeline。
3. **GStreamer 版本下限 v1_22**：如果板上 GStreamer < 1.22（不太可能，板上 1.26.2），构建失败。
4. **没有 GStreamer 插件依赖声明**：Cargo.toml 只声明 Rust 绑定，不声明 GStreamer 插件（v4l2src、x264、openh264 等）。这些由系统包提供。
5. **axum 只有一个路由**：注释说"the console's one route"——这不是通用 Web 服务器。
6. **没有 serde 默认 feature**：serde_json.workspace=true，但没看到 serde 本身——可能通过 workspace 间接引入。
7. **tempfile 只在 dev-deps**：测试用临时目录放假配置。
8. **GStreamer 0.24 Rust binding**：这是 gst-plugins-rs 0.24 系列，与系统 GStreamer C 库版本独立但通过 v1_22 feature 门控。
9. **注释提到 webrtc-console.md §1.1**：有外部设计文档（不在本仓库中）记录 axum 选型理由。
10. **cross-sysroot.sh**：GStreamer 的交叉编译需要完整 sysroot——这是 workspace 级别的构建成本。
11. **duck-detect 依赖**：mediad 不仅做媒体流，还做检测——检测结果通过 WebRTC data channel 或 IPC 发送。
12. **没有 gstreamer-audio**：虽然 description 提到 mic，但 Cargo.toml 没有显式音频库。可能用 gstreamer-app 或 gstreamer 核心处理音频。

---

## 九、设计观察

### 9.1 这是"第二个 C 依赖"的里程碑

注释（L13–L16）承认这是一个重大决定：
> "This is the second C dependency to reach the board, after libudev for `gilrs` in `padd`, and it is much larger... `ci-cross-deps.sh` says of the first that 'it is the cost of that one exception, and it is worth reading before adding another' — this is that other, and the cost is a sysroot the whole workspace now builds against."

**含义**：第一个 C 依赖（libudev）是"一个例外"，GStreamer 是"另一个"，代价是整个 workspace 现在都要 against 一个 sysroot 构建。这是有意识的权衡——GStreamer 提供了完整的相机/编解码/WebRTC 栈，手写成本太高。

### 9.2 测试策略：真 socket 假 daemon

dev-dep 注释（L70–L71）：
> "Fake daemons on real unix sockets, which is how the pipe is tested without a WebRTC peer."

**方法**：不在测试中 mock WebRTC peer，而是启动假 daemon 监听真实 unix socket，验证 mediad 的 pipe 逻辑。这是集成测试风格——测真实 IPC 而非 mock 对象。

### 9.3 配置 schema 不漂移

四方共享 robotd-params：
1. mediad 读 [media]/[detect]
2. robotd 解析整个文件
3. robotctl configure 编辑
4. 测试用真实文件（tempfile）

这是"schema 作为库"的模式——配置格式不是散落在各 crate 的 TOML 字符串，而是一个类型安全的 Rust 结构体。

---

## 十、结论（按状态分级）

### confirmed
- C1：mediad 是媒体守护进程 crate，负责摄像头/麦克风/WebRTC/远程网关。
- C2：GStreamer 依赖仅 Linux 编译，按 target 门控而非 feature gate。
- C3：GStreamer 0.24 绑定，v1_22 feature（下限由插件声明，非板上版本碰巧）。
- C4：板上 GStreamer 版本 1.26.2。
- C5：axum 0.8.9 关默认特性，只服务一个 GET 路由。
- C6：复用 duck-detect（NPU 检测）、robotd-params（配置）、duck-ipc-proto（IPC）。
- C7：appsink 从 tee 取原始 NV12，避免编码后再解码。
- C8：GstVideoMeta 声明实现 v4l2src 零拷贝。
- C9：这是 workspace 第二个 C 依赖（第一个是 padd 的 libudev/gilrs）。
- C10：交叉编译需要 scripts/cross-sysroot.sh 建立的 sysroot。

### inferred
- I1：pipeline 结构约为 v4l2src → tee → [appsink (NV12), webrtcbin (encoded)]。
- I2：音频通过 GStreamer 核心处理，不需要 gstreamer-audio 显式依赖。
- I3：webrtc-console.md 是设计文档，记录 axum 选型和路由设计。
- I4：ci-cross-deps.sh 有审批流程控制新 C 依赖加入。

### unknown
- U1：完整的 GStreamer pipeline 字符串（在 .rs 源文件中）。
- U2：WebRTC 信令协议（SDP? WHIP? 自定义?）。
- U3：data channel 传输什么（检测结果？控制命令？）。
- U4：音频采样率/编码格式。
- U5：控制台 GET 路由服务什么文件（状态页？视频流？）。

---

## 十一、未知项与最小验证动作

| 未知项 | 最小验证动作 | 预期通过信号 |
|--------|-------------|-------------|
| U1 pipeline | 读取 mediad/src/pipeline.rs 或类似文件 | 看到 gst::parse_launch! 字符串 |
| U2 信令 | 读取 webrtc-console.md 或 signaling 模块 | 看到 SDP exchange |
| U3 data channel | grep -rn 'data.channel\|data_channel' src/ | 看到消息类型 |
| U4 音频 | grep -rn 'audio\|alsasrc\|pulsesrc' src/ | 看到音频源 |
| U5 控制台路由 | grep -rn 'get(\|axum::Route' src/ | 看到路由定义 |

---

## 附录 A　资料来源

1. 原文件：mediad/Cargo.toml（本地附件，73 行，全文已读）。
2. 关联文件：duck-detect Cargo.toml（NPU 检测 crate）、robotd-params、duck-ipc-proto。
3. 外部参考：gst-plugins-rs 文档、axum 文档。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-11 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
