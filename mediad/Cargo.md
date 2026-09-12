# Cargo.toml 文件解析

**文件位置**：`d:\microduck\mediad\Cargo.toml`

## 核心设计决策

`mediad`：摄像头、麦克风、WebRTC 与远程网关。GStreamer 仅 Linux（与 `padd` 的 evdev tap 同理）；守护进程跑在机器人上，而路由表、会话管道、上游连接池可移植，其测试在笔记本上运行。用 target 而非 feature 门控，让 `cargo test` 在两端都诚实——feature 默认关闭的模块等于没人编译。

这是继 `padd` 的 libudev 之后第二个到达板上的 C 依赖，且大得多——`scripts/cross-sysroot.sh` 为此存在。

## 依赖

### 全平台

| 依赖 | 用途 |
|---|---|
| `duck-ipc-proto` | IPC 契约（方法、错误码、socket 路径） |
| `robotd-params` | `robotd.toml` 的 `[media]`/`[detect]`，与 `robotd`/`robotctl configure` 同 schema |
| `duck-detect` | 鸭子检测器（letterbox+转向、NPU 绑定、ONNX 回退、解码器） |
| `axum` 0.8.9（`default-features=false`，仅 `http1`+`tokio`） | 控制台一个路由，默认特性全是重量 |
| `clap`、`serde_json`、`tokio`、`tracing`、`tracing-subscriber`、`anyhow` | 标准基础设施 |

### 仅 Linux（GStreamer 管线）

| 依赖 | 用途 |
|---|---|
| `gstreamer` 0.24（`v1_22`） | 核心 |
| `gstreamer-app` 0.24 | `appsink` 取原始帧分支 |
| `gstreamer-webrtc` 0.24 | 数据通道（`GstWebRTCDataChannel`） |
| `gstreamer-video` 0.24 | `GstVideoMeta`，`v4l2src` 必须在 ALLOCATION 查询里公告，否则逐帧拷贝 |
| `glib` 0.21 | 枚举类、值类型 |

`v1_22` 不是猜的：是 `gst-plugins-rs` 自己 webrtc crate 声明的下限，也是 `WebRTCDataChannel` 类型存在所需（需 `v1_18`）。板上跑 1.26.2，按插件要求而非板上碰巧版本钉下限。

### dev-dependencies

- `duck-ipc-proto`（`test-support`）：`every_call()`，穷举匹配测试用同一份列表。
- `tempfile`：假守护进程在真实 unix socket 上。
- `tokio`（`rt-multi-thread`+`macros`）。

## 关键摘要

- GStreamer 按 target 门控（非 feature），保持测试诚实。
- 控制台用 axum 精简特性，不引入手写 HTTP/1.1 解析器。
- 运行时库（GStreamer）不链接时依赖，由 `setup-gstreamer.sh` 安装；`GST_PLUGIN_PATH` 在 unit 文件设置。
