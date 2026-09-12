# pipeline.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\pipeline.rs`

## 核心设计决策

GStreamer 管线与控制通道所骑的数据通道。仅 Linux。

### 管线形状

```text
                                               ┌─ queue ─ webrtcsink（内置 mpph264enc + 信令服务器）
videotestsrc | camera ─ UYVY ─ capsfilter ─ tee ┤
                                               └─ queue(leaky) ─ appsink（最新帧）
```

**tee 在原始 UYVY 上，在编码器之前**：`architecture.md` §5.3 要按需取帧、§2 要感知贴近传感器，两者都要像素，从编码分支取意味着解码刚编码的东西。

**UYVY 而非 NV12，是测量结果**：rkisp 提供两平面非连续的 `NM12`，请求 GStreamer `NV12` 会选它，`v4l2src` 在该驱动下无法满速推送。300 帧 720p：NV12 19.5 fps，UYVY 29.3 fps。`mpph264enc` 列 UYVY 为 sink pad 格式并用 SoC 2D 加速器转换，4:2:2→4:2:0 零 CPU 成本。

**不转换、不旋转**：头部摄像头安装偏四分之一圈，曾用 `videoflip` 在 tee 前旋转，破坏了 `mpph264enc` 到 2D 引擎的零拷贝路径，MPP 软件转换每帧：97 °C、CPU 节流 408 MHz、30 fps 摄像头只剩 8 fps。旋转现在是消费者的事（浏览器 CSS 变换、检测器重采样），都免费。

**每分支自有 queue，原始分支漏底（leaky downstream）**：tee 无 queue 时单线程推所有分支，慢消费者会卡住其他分支（感知消费者暂停视频轨）。原始分支丢旧帧而非背压，是 last-value-wins 语义。

**`webrtcsink` 在本进程内跑信令服务器**（`run-signalling-server=true`），无需单独的 `gst-webrtc-signalling-server` 二进制。

**`webrtcsink` 拥有编码器，喂原始视频**：曾用 `mpph264enc ! h264parse ! webrtcsink`，工作但悄悄放弃两样：预编码输入下 `webrtcsink` 碰不到编码器，拥塞控制无法随链路调码率，对等端 PLI 无法产生关键帧。代价是 `webrtcsink` 不认识 `mpph264enc` 时前面需软件 `videoconvert ! videoscale`——所以发布的插件带补丁加该分支。两者必须一起。

### 测试图案优先于摄像头

默认源是 `videotestsrc`。不是占位，而是无摄像头的板子（大多数）能让完整会话（信令、协商、数据通道、控制 API）可演练。

### 信号处理器不得 panic

这些闭包从 C 调用，panic 不会 unwind——会 abort 进程，journal 只显示 `thread caused non-unwinding panic` 穿过 `g_closure_invoke`，不说明真正问题。所以：runtime handle 在有 runtime 处捕获并显式 spawn；每个信号在连接或发射前检查存在（`emit_by_name`/`connect` 对不存在的名字会 panic）。

## 关键类型

- **`Rotation`**：`None/Cw90/Cw180/Cw270`。`from_degrees`、`video_direction`（对应 `videoflip` 的 `video-direction`）、`output`（90/270 交换宽高）。
- **`Settings`**：命名字段而非位置参数（`start` 曾达 8 个参数，width/height 同类型交换会编译通过却产生竖屏流）。
- **`Source`**：`Test` 或 `Camera{device, exposure, analogue_gain}`。
- **`Frame`**：`width/height/format/data`，format 显式携带（防格式变化时静默误读字节）。
- **`Frames`**：`Arc<Mutex<Option<Frame>>>`，last-value-wins。`inspect` 原地读（不拷贝，曝光环只需均值）、`latest` 克隆。

## 关键函数

- **`start(source, producer, settings)`**：构建并启动管线。含：`set_gstreamer_log_threshold`（`gst::init` 前设 `GST_DEBUG=*:WARNING`）、`bridge_gstreamer_log`（`init` 后把 GStreamer 日志桥接到 tracing）、捕获 tokio runtime handle（GStreamer 信号线程无 runtime）、`camera_source`、capsfilter（UYVY）、可选 `videoflip`、tee、video 分支（queue→webrtcsink）、raw 分支（leaky queue→appsink）、`wire_consumers`（数据通道）、`watch_bus`（独立线程 pop 总线消息转 tracing）、`meter_capture_rate`。
- **`camera_source`**：`pin_sensor_mode`（`media-ctl` 把 IMX219 设到 1920x1080 @30，否则启动模式 3280x2464 限 21 fps）+ `v4l2src`（`extra-controls` 写起始曝光/增益）+ `raise_capture_buffers`。
- **`raise_capture_buffers`**：核心技巧。`gst_v4l2_object_decide_allocation` 算池深三种方式，只有一种够用。需要：1) `GstVideoMeta`（否则 `can_share_own_pool=false`，走 else 分支 min=2 丢帧）；2) 第一个 pool 的 `min` 非零（`mpph264enc` 的 `propose_allocation` 会覆写为 0）。用 pad probe 在 allocation query 往返两次都重写 pool 0 的 min=4，保证最后说了算。`CAPTURE_BUFFERS=4`（3 是悬崖）。
- **`pin_sensor_mode`/`find_sensor`**：`media-ctl` 发现 IMX219 实体（名含 I2C 总线地址，按子串匹配），设 1920x1080 SRGGB10。失败只 warn（采集仍工作只是更慢）。
- **`set_congestion_control`**：防御性设置——查属性、通过枚举类解析 nick，任一失败留默认并 warn（不 panic）。
- **`wire_encoder_setup`**：`webrtcsink` 的 `encoder-setup` 信号，设 H.264 profile（constrained-baseline）与 header-mode，否则默认会回归。
- **`wire_consumers`**：`consumer-added` 信号为每个对等端建 `Channel`（inbound/outbound mpsc），数据通道消息转线路。
- **`wire_frames`**：appsink 回调把最新帧写入 `Frames`（替换而非排队）。

## 常量

- `CAPTURE_FORMAT = "UYVY"`
- `CAPTURE_BUFFERS = 4`

## 关键摘要

- tee 在编码器前的 UYVY 上，零转换、零旋转（消费者自己转）。
- `webrtcsink` 拥有编码器+内置信令服务器，发布的插件带 mpph264enc 补丁。
- `raise_capture_buffers` 通过 pad probe 强制 pool min=4 + VideoMeta，解决 v4l2src 丢三分之一帧。
- IMX219 用 `media-ctl` 切到 1920x1080 模式才能 30 fps。
- 所有 GStreamer 信号 handler 不得 panic（C 闭包边界 abort）。
