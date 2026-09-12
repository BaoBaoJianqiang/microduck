# 硬件上的媒体启动

一块 Radxa Zero 3W 对视频做什么。以下所有内容都在一块板子上观察到，而非推断 —— 哪里仍是假设它会说明。

## 这解决了什么

`mediad` 需要硬件 H.264：软件编码不是一个更慢的选项，它根本不是一个选项。`jpegenc` 单独在这个 SoC 上无法以 640x480 维持 30 fps（`microduck_runtime/src/camera.rs:500`），而 H.264 每帧比 JPEG 更贵，在四个已经被 `robotd` 的 50 Hz 控制循环共享的 Cortex-A55 上。

**VPU 编码 H.264，比特流有效，且编码器通过 Rockchip 的 MPP 而非 V4L2 到达。** 必须从源码构建两个 GStreamer 插件才能使用其中任何东西，且两者都不被任何未知东西阻塞。

| | |
|---|---|
| VPU 编码 720p H.264 | 是 —— 60 帧，428 KB，通过 `mpi_enc_test` |
| 此内核上比特流有效 | 是 —— 干净的 `avdec_h264` 解码，High profile level 4，4:2:0 8-bit |
| 通过什么到达 | `/dev/mpp_service`（Rockchip MPP）。**不是** V4L2 M2M |
| 跨 1.x 的 GStreamer 插件 ABI | 不是风险 —— 一个 1.14 构建的插件在 1.26.2 中干净注册 |
| `mpph264enc` 元素 | 必须构建（§ [必须构建什么](#必须构建什么)） |
| `webrtcsink` / `webrtcsrc` | 必须构建，分开 |

## 板子有什么

在一块已初始化的板子上没有安装任何媒体相关的东西。`scripts/setup-gstreamer.sh` 安装它并报告硬件能做什么；那个脚本是本页的可执行形式，且是某人再次需要的命令该去的地方。

GStreamer 来自**普通 Debian trixie** —— `apt-cache policy` 显示 `deb.debian.org` 与 `security.debian.org`，无 Armbian 多媒体 overlay，因此归档的版本完全适用：

| 包 | 版本 |
|---|---|
| `gstreamer1.0-plugins-bad`（有 `webrtcbin`） | 1.26.2-3+deb13u3 |
| `libgstreamer-plugins-bad1.0-dev`（有 `gstreamer-webrtc-1.0.pc`） | 1.26.2-3+deb13u3 |
| `gstreamer1.0-nice` | 0.1.22-1 |
| `gstreamer1.0-plugins-rs` | **在任何 Debian 套件中都不存在** |

内核是 `6.1.115-vendor-rk35xx`。这关系到两次：摄像头的 MIPI-CSI ISP 捕获驱动只在 Armbian 的厂商分支上存在，VPU 节点也是。`setup-board.sh` 已经安装了那个内核 —— 为了音频编解码器的 I²S 树，不是为了视频 —— 因此先决条件在有人要求之前就满足了。一个拉取 `current` 内核并重指向 `/boot` 的 stray `apt upgrade` 会把两者都拿走。

## 摄像头需要一个 overlay，在前缀下镜像

一个插入的 CSI 摄像头产生**没有 `/dev/video*` 且 dmesg 中什么都没有**，直到它的设备树 overlay 被启用 —— 这读起来完全像一个没连接的摄像头。

启用它有一个值得单独陈述的陷阱。Armbian 把 overlay 作为 `radxa-zero3-rpi-camera-v2.dtbo` 交付，**没有 `rk3568-` 前缀**，而板子运行 `overlay_prefix=rk3568`。因此一个 `overlays=` 词解析为 `rk3568-radxa-zero3-rpi-camera-v2.dtbo`，加载器什么都找不到，板子开心地启动且没有摄像头 —— 与 `configure_overlay` 为 `uart2-m0` 防止的同样的静默失败。文件必须**先在前缀名下面镜像**，然后才能在 `overlays=` 中命名。`microduck_runtime/install.sh` 撞到了这个并做了同样的事。

`setup-board.sh` 中的 `configure_camera` 两者都做，放进*厂商*内核的 overlay 目录 —— MIPI-CSI 捕获驱动只在那个分支上存在，这是厂商内核不是可选的第二个原因。`DUCK_CAMERA_OVERLAY` 选择另一个模块；Armbian 每个传感器交付一个，对这块板子是 `radxa-zero3-rpi-camera-v2`（Pi Cam v2 / IMX219）或 `radxa-zero3-rpi-camera-v1.3`（Pi Cam v1.3 / OV5647）。这里只用过第一个。

## 编码器是 MPP，不是 V4L2

`v4l2h264enc` 缺席且 `/dev/video*` 为空。两者都不是故障：

- 在一个 Rockchip BSP 内核上，VPU 被暴露为 `/dev/mpp_service`，而非一个 V4L2 M2M 编码器。`v4l2h264enc` 仅当 `gstreamer1.0-plugins-good` 找到一个编码器节点时才被注册，因此它的缺席在这里是预期形态而非一个缺失的包。
- 完全没有 `/dev/video*` 也正好是一个**未连接摄像头**的样子 —— rkisp 捕获节点只在一个传感器被探测后出现。

这值得直白陈述，因为它是分支点。如果内核暴露了一个 V4L2 编码器，硬件 H.264 根本不需要任何树外的东西。

### 权限陷阱

`/dev/mpp_service` 以 `crw------- root root`、模式 0600 到达。一个非 root 进程无法打开它 —— 且 **`mpi_enc_test` 针对它写一个空文件并以 0 退出**。无错误，无日志行。因此零退出状态什么都证明不了；文件大小才是证据。

`mediad` 将作为它自己的用户运行，像这里的每个其他守护进程一样 —— `tofd` 进入 `i2c`，`padd` 进入 `input`，`btd` 进入 `bluetooth` —— 因此 VPU 需要同样的处理：一个给节点一个组的 udev 规则，以及 unit 上的 `SupplementaryGroups=`。`scripts/setup-gstreamer.sh` 安装那个规则（`99-robot-mpp.rules`，组 `video`，模式 0660），遵循 `setup-board.sh` 中 `configure_tof` 的 i2c 规则的形态。

用 `video` 而非 `robot`：`robot` 门控我们*定义*的 IPC socket（[`app-path-design.md`](../design/app-path-design.md) §，socket-mode-plus-group 分层）。一个内核设备节点不是我们能重新定义的，且 `video` 是这个设备类的发行版约定，因此一个用 `gst-launch` 的开发者以与 `mediad` 相同的方式进入。

## Radxa 的池提供什么

Rockchip MPP 不在 Debian 中。Radxa 把它作为一个 GitHub Pages apt 仓库发布，且这些包被作为**直接的 `.deb` 下载**而非把仓库加到 `sources.list` —— 这是 `microduck_runtime/radxa_setup/setup_rkaiq.sh` 已经在这块板子上为 `rkaiq_3A_server` 使用的路线。基础：`https://radxa-repo.github.io/bullseye/pool/main`。

| 包 | 版本 | 为什么 |
|---|---|---|
| `m/mpp/librockchip-mpp1` | 1.5.0-1 | MPP 用户空间库 |
| `m/mpp/librockchip-vpu0` | 1.5.0-1 | `rockchip-mpp-demos` 精确依赖此版本 |
| `m/mpp/rockchip-mpp-demos` | 1.5.0-1 | `mpi_enc_test` —— 不涉及 GStreamer 证明 VPU |
| `m/mpp/librockchip-mpp-dev` | 1.5.0-1 | 头文件，构建编码器插件 |
| `libr/librga/librga2` | 2.2.0-1 | Rockchip 2D 加速器；rockchip 插件依赖它 |
| `libr/librga/librga-dev` | 2.2.0-1 | 头文件，同一次构建 |
| `g/gstreamer1.0-rockchip/gstreamer1.0-rockchip1` | 1.14-4 | MPP GStreamer 插件 —— 见下 |

这些是 bullseye 构建，且它们在 trixie 上针对 glibc 2.41 干净配置。

**`dpkg -i` 不解析任何东西**，因为这些不来自一个配置好的 apt 源。每个缺失的依赖都是一个未配置的包而非一个自我修复的安装，因此每组都必须命名其完整闭包。那花了三次往返才学会。

### Radxa 的预构建插件*不是*仅解码的，本页曾说它是

`gstreamer1.0-rockchip1_1.14-4` 在 GStreamer 1.26.2 中干净安装并注册，恰好显示 `mppvideodec` 与 `mppjpegdec`。这读起来像"无编码器"，本页曾声称 —— 错了。对其 `.so` 的 `strings` 列出 `mpph264enc`、`mpph265enc`、`mppjpegenc` 与 `mppvp8enc`。它们都在。

**权限陷阱是全部解释，且在那清楚之前它产生了四个独立的误导结果：**

| 看起来像什么 | 实际真相 |
|---|---|
| `mpi_enc_test` 什么都没写且**以 0 退出** | 节点打不开；零退出什么都说明不了 |
| Radxa 的 deb 仅解码 | 它有每个编码器 |
| 一个第三方 1.14-8 deb 仍显示无 `mpph264enc` | 同样原因 |
| 我们自己的 CI 构建只列出两个解码器 | 容器也没有 `/dev/mpp_service` —— 预期，非失败 |

一个 MPP 插件**无条件注册其解码器，并在注册其编码器之前探测 MPP。** 当 `/dev/mpp_service` 在 `0600 root:root` 时探测静默失败，因此编码器被从一个完全包含它们的插件中省略。

因此一个只列出解码器的插件是关于*设备节点*的证据，而非关于插件，且在 udev 规则到位之前 `gst-inspect-1.0 mpph264enc` 毫无意义。

那次安装确实证明了一件事，且它成立：**一个针对 GStreamer 1.14 构建的插件在 1.26.2 中毫无怨言地注册。** 插件 ABI 曾是害怕源码构建的陈述理由，且它不是风险。

## 必须构建什么

两个插件，出于两个不相关的原因。两者都不替代对方。

| 插件 | 源 | 提供 | 为什么无法安装 |
|---|---|---|---|
| `gstreamer-rockchip` | [`JeffyCN/mirrors`](https://github.com/JeffyCN/mirrors) 分支 `gstreamer-rockchip`，meson | `mpph264enc` —— 硬件编码器 | Debian 完全没有 Rockchip 编码器。Radxa 的构建*确实*有它们，因此这个是关于一个我们控制的 pin，去掉 `libx11-6`，并与下面的插件同行 |
| `gst-plugin-webrtc` | [`gst-plugins-rs`](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs) 0.15.3，cargo-c | `webrtcsink`、`webrtcsrc` | `gst-plugins-rs` 在**没有** Debian 套件中被打包 |

用 0.15.3 而非 `reachy_mini` SDK 记录的 0.14.5：0.14.5 是重要的下限 —— 低于它缺少 `webrtcsink` 在远程描述与 ICE 处理之间的死锁修复，表现为客户端在"connecting"上永远旋转 —— 且 0.15.3 只是更新。两个系列都声明一个 GStreamer `v1_22` 特性下限，而机器人运行 1.26.2，因此更新的那个不花代价。

`webrtcbin` **已**安装，来自 `gstreamer1.0-plugins-bad`。因此一个 WebRTC 会话今天无需第二次构建即可到达 —— 代价是我们自己实现信令协议。`webrtcsink` 被偏好，因为它的信令协议是中继代理的东西，这正是让一个中心信令服务器可复用的原因。

### 为什么不用预构建的

硬件、内核驱动与 MPP 的用户空间库都在**什么都不编译**的情况下工作 —— `mpi_enc_test` 来自一个 deb 并在第一次尝试就编码了 720p H.264。缺失的只是*GStreamer 绑定*：一个把 `librockchip-mpp` 包装为一个流水线可以使用的元素的插件。`mpi_enc_test` 是一个独立程序；GStreamer 完全不知道它存在。与这里的 ONNX Runtime 形态相同 —— `libonnxruntime.so` 从一个 tarball 安装，什么都不编译，而 `ort` 是让它可达的绑定。

预构建绑定：

| 源 | 它有什么 |
|---|---|
| Radxa `bullseye` 池 | `gstreamer1.0-rockchip1_1.14-4` —— 已安装并检查：仅 `mppvideodec` + `mppjpegdec` |
| Radxa `rk3588s2-bookworm` 池 | 相同的 `1.14-4`，字节相同 |
| [`numbqq/gstreamer-rockchip-debs`](https://github.com/numbqq/gstreamer-rockchip-debs) | `1.14-8` —— **有每个编码器** |

最后一个在构建任何东西之前值得一试。它的 `bookworm/arm64/<board>/` 条目是到 `jammy/arm64/` 的符号链接，因此它是一个 Ubuntu 22.04 构建，来自 `rockchip-linux/gstreamer-rockchip`（现在 404），Jeffy Chen 为维护者 —— 与 Radxa 构建的同一个上游，在一个启用了编码器的修订版。`mpph264enc`、`mpph265enc`、`mppjpegenc` 与 `mppvp8enc` 都在 `.so` 中。

它的 `DT_NEEDED` 被板子在上面的 deb 之后已经有的东西满足：`librockchip_mpp.so.1`、`librga.so.2`、`libgstreamer-1.0.so.0`、`libgstvideo`、`libgstallocators`、`libgstpbutils`、`libdrm2`、`libglib2.0-0`、`libx11-6`、针对 glibc 2.41 的 `libc6 >= 2.33`。其中没有任何东西是 RK3588 特定的 —— SoC 差异生活在 MPP 内部，而非插件 —— 且 `Depends` 仅从下方界定 GStreamer（`>= 1.14`）。

我们的在 [`microduck-gst-plugins`](https://github.com/pollen-robotics/microduck-gst-plugins) 中构建 —— 一个自己的仓库，故意：

- **不在板子上。** 一个 RK3566 编译 Rust 那半太慢，等不了。
- **也不交叉编译。** 守护进程用 `cargo-zigbuild` 交叉构建，且 `scripts/ci-cross-deps.sh` 直白地说它的一个 C 依赖"是那一个例外的代价，且在添加另一个之前值得一读"。GStreamer 会是一个大得多的第二个，且两条路线 —— x86 multiarch，或一个带 meson cross 文件的 sysroot —— 都链接到目标的一个近似。
- **原生，在一个 `debian:trixie` 容器中的 arm64 runner 上**，那是机器人自己的用户空间。没有东西被近似。arm64 runner 在公共仓库上是免费的。
- **公开**有第二个更重要的原因：下载发生在初始化期间且来自更新器的 `preinstall` hook，那个 hook 以一个清空的环境且**无 token** 运行。与守护进程已经依赖 ONNX Runtime 的相同安排。

它构建两个插件，在一个 `pins.env` 中按 commit 或 tag pin 上游，禁用 `rkximage` 与 `kmssrc`（同一棵树中的 X11 与 KMS sink —— 一个无头机器人两者都不需要，且它们是预构建 Radxa deb 依赖 `libx11-6` 的原因），并发布一个 tarball 加上它的 sha256 与一个命名每个插件精确上游 ref 的 `MANIFEST`。那个清单是第三方 deb 无法回答的东西。

它防范的两个陷阱，都是通过读树而非失败发现的：`gst/rockchipmpp/meson.build` 以 `if not mpp_dep.found() → subdir_done()` 结尾，因此一个缺失的 `librockchip-mpp-dev` 让 meson **跳过插件并成功**；且 `dpkg -i` 对直接 `.deb` 下载不解析任何东西，因此 Radxa 闭包在一次调用中安装。

**`mediad.service` 必须设置 `GST_PLUGIN_PATH`。** 插件安装到 `/usr/local/lib/gstreamer-1.0`，GStreamer 默认**不**搜索 —— 它的内置路径是发行版的 `/usr/lib/aarch64-linux-gnu/gstreamer-1.0`，且那个目录被故意避开，以便一个 `apt` 操作无法替换或移除它们。因此 unit 需要

```
Environment=GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0
```

与 VPU 节点需要的 `SupplementaryGroups=video` 一起。两者都容易忘记，且两者在运行时以同样方式呈现：编码器就是不存在，没有东西说明为什么。

`scripts/setup-gstreamer.sh` 以一个**固定**版本消费它 —— 从不"latest"。相隔一天的两次初始化运行产生不同的插件，且没有东西记录是哪个，是一个等待发生的不可复现媒体 bug。pin 位于 `Cargo.toml` 的 `[workspace.metadata.gst-plugins]`，脚本携带字面量因为它用 `curl` 独立获取，且一个 `xtask` 测试断言它们一致 —— 与 `ONNX_VERSION` 相同的安排与相同的理由。

**尝试第三方 deb 与依赖它不是一回事。** 它是一个人的逐板转储，没有我们控制的出处，且如果那个仓库消失它也消失。它廉价买到的是关于构建的唯一真正问题的答案 —— 这个插件是否针对*我们的* MPP 与 GStreamer 版本工作 —— 且如果是，自己构建同一份源码就被去风险而非不必要。一个我们自己的固定构建仍是这该落脚的地方。

**构建可以完全避免**，通过从 `mediad` 经 Rust FFI 调用 MPP 的 C API，`mpi_enc_test` 证明这工作。那把一个 meson 构建换成手写且手维护的对一个厂商库的绑定，这是交易中较差的一面 —— 但如果插件结果与 GStreamer 1.26 打架，它是一个真实选项，不是死路。

### 上游

`rockchip-linux/gstreamer-rockchip` 没了（404）。`JeffyCN/mirrors@gstreamer-rockchip` 是活的镜像 —— 最后提交 2026-05-21 —— 且 `gst/rockchipmpp` 持有 `gstmpph264enc.c`、`gstmpph265enc.c`、`gstmppjpegenc.c`、`gstmppvp8enc.c`。周围存在 fork 汤；无论用哪个 fork 与 tag 都必须被固定并记录，出于与 `gst-plugin-webrtc` 被固定到 ≥ 0.14.5 相同的理由（下）。

### `gst-plugin-webrtc` 的 pin

**0.14.5 或更新，非 0.14.4。** 较早的 tag 缺少 `webrtcsink` 在远程描述与 ICE 处理之间的死锁修复，表现为客户端在"connecting"上永远旋转。`reachy_mini` 的 SDK 安装文档记录了它；注意 `reachy-mini-desktop-app` vendor 0.14.4 因此在那条线的错误一侧。

Pollen 已经为 **x86_64** vendor 这个插件 —— 用 `cargo cinstall` 原生构建、剥离、按架构提交，并被 CI 固定到一个 commit ref 加一个 sha256 消费。同一个地方的一个 `aarch64/` 兄弟可能比这里的第二个流水线总工作量更少。

### 一个构建的插件该属于哪里

在守护进程版本载荷中，`GST_PLUGIN_PATH` 指向 `current` 内部 —— 不在 apt 中，也不在每台机器的 `/opt` 中。

插件版本与 `mediad` 的代码是纠缠的：上面 0.14.5 的故事正好是一个插件版本决定守护进程是否需要一个变通方案的案例。一个偏差是一个 `mediad` bug，因此它想要 `mediad` 的生命周期 —— 原子交换、回滚、健康门。`librockchip-mpp` 正相反：一个与*内核*配对的系统库，被任何触碰 VPU 的东西想要，且属于包管理器。

## 测量过的，与未测量的

**在板子上测量：** GStreamer 1.26.2 及其来源；`webrtcbin` 存在；`webrtcsink`/`webrtcsrc` 缺席；`v4l2h264enc` 缺席且无 `/dev/video*`；`/dev/mpp_service` 在 0600 root:root 存在；`mpi_enc_test` 作为非 root 静默什么都不写、作为 root 写 428 KB；该比特流干净解码为 High/4.0；Radxa deb 的依赖闭包；rockchip 插件在 1.26.2 中加载带两个解码元素。

**编码路径已在硬件上端到端关闭。** 顺序：

1. `v1` 从公共发布获取，sha256 验证，安装到 `/usr/local/lib/gstreamer-1.0`。
2. `gst-inspect-1.0 mpph264enc` 应答 `provided-by /usr/local/lib/gstreamer-1.0/libgstrockchipmpp.so` —— 我们的构建。第三方 deb 先被移除以便答案可归因于某物。
3. 它**编码**：`videotestsrc ! mpph264enc profile=baseline header-mode=each-idr bps=2000000 ! h264parse ! filesink` 在 **0.44 秒墙钟**内为 720p 的 60 帧产生 476 KB，源生成与流水线设置包括在内 —— 轻松快于实时。结果通过 `avdec_h264` 干净解码。
4. 它**无需 root** 工作：在一个 udev 规则把 `/dev/mpp_service` 放到 `660 root:video` 且用户加入 `video` 后，`gst-inspect` 作为那个用户应答。那是 `mediad` 所在的情况，且此前每个检查都在 `sudo` 下。

**整个媒体链在硬件上关闭。** 传感器到一个 WebRTC 可协商的流：

| 步骤 | 证据 |
|---|---|
| overlay 已应用 | `csi2-dphy0` 探测，`rkisp` 起来，十个 `/dev/videoN` |
| 传感器识别 | `imx219 2-0010: Model ID 0x0219, Lot ID 0x5a8e73, Chip ID 0x0773` |
| 捕获节点 | `/dev/video0`，卡名 `rkisp_mainpath`，格式到 3280x2464 |
| 帧 | 720p NV12 下 `--stream-count=10` 的 13,824,000 字节 —— 精确 |
| 硬件编码 | `v4l2-ctl … --stream-to=-` 进入 `fdsrc ! rawvideoparse ! mpph264enc` |
| 流 | 干净解码；`h264parse` 报告 **1280x720 constrained-baseline** |

节点号在启动间不稳定，因此捕获节点通过匹配 `/sys/class/video4linux/*/name` 下的卡名 `rkisp_mainpath` 找到 —— 如 `camera.rs:219` 所做。

### 三个设备节点需要 `video` 组，不是一个

这花了三轮独立调试，每轮的失败都命名了别的东西：

| 节点 | 仅 root 时的症状 |
|---|---|
| `/dev/mpp_service` | `mpi_enc_test` 什么都不写且**以 0 退出**；`mpph264enc` 完全不被注册 |
| `/dev/rga` | 元素存在，流水线启动，然后 `Try to use uninit rgaCtx=(nil)` 与成页的 `rga call blit fail` |
| `/dev/video0` | 已经以 `root:video` 到达，因此它是不咬人的那个 |

`setup-gstreamer.sh` 安装一个覆盖前两个的 udev 规则。**`mediad.service` 需要 `SupplementaryGroups=video`** —— 与 `Environment=GST_PLUGIN_PATH` 一起，那是站在一个工作流水线与四种不同困惑失败之间的两行。

### 花了 22 fps 的旋转

摄像头安装偏了四分之一圈，而明显的修复 —— 在 tee 之前 `videoflip`，以便每个消费者得到一个正立的画面 —— 是错的，在一个机器人上测量：

| | RGA 失败 | `v4l2src` 丢帧 | fps | SoC |
|---|---|---|---|---|
| 无翻转 | 0 | 0 | ~30 | 正常 |
| 有翻转 | 一会话 5522 | 1565 | 7–8 | 97 °C，CPU 408 MHz |

`mpph264enc` 把 UYVY→NV12 转换交给 SoC 的 2D 引擎且不为它付出代价。`videoflip` 的输出是 RGA 拒绝的一个缓冲区 —— `10000 is unsupport format`，然后在一个 `rect[0, 0, 720, 1280]` 上 `RGA_BLIT fail: Bad address` —— 因此 MPP 回退到在软件中转换**每一帧**。这让 CPU 饱和，SoC 达到其热限，一切节流到 408 MHz，且摄像头丢掉它无法交付的帧。旋转本身是代价中较小的一半。

因此流水线中没有任何东西旋转。安装被*报告*（`media.video`，每个控制通道一次），且谁显示画面谁旋转它：控制台用一个 CSS 变换做，那是免费的，而一个感知消费者把它折进它已经做的重采样。`--flip-in-pipeline` 为一个无法自己旋转且负担得起的消费者把旧行为放回去。

如果一个正立的流曾经真正需要，合适的修复是插件集中的一个 RGA 元素：2D 引擎免费旋转，这正是编码器用得起它的原因。

### 3A 引擎必须在流开始之前等待

`rkaiq_3A_server` 附加到 ISP 然后等待一个**流开始**事件 —— 且它错过已经发生的一个。在 `mediad` 正在流式传输时重启它，它坐在

```
DBG: /dev/media0: wait stream start event...
```

永远：无统计循环，因此无自动曝光与无白平衡，且一个绿色画面。一次重启"修复"它只是因为重启碰巧把两者排对了顺序。

这让它成为一个无明显原因的回归，因为**每个 `robotctl update apply` 都在一个运行的流下面重启引擎**：安装前钩子运行 `setup-rkaiq.sh`，而那个脚本重启 `rkaiq_3A`。它甚至打印"restart the camera stream for it to take effect" —— 没人遵循且没有东西强制执行的建议。

因此那个脚本安装的 drop-in 现在承载不变量：

```ini
ExecStartPost=-/bin/systemctl --no-block try-restart mediad.service
```

每当引擎启动，流就在它后面被弹跳，因此它等待的事件是一个尚未发生的。`try-restart` 让一块没有 `mediad` 的板子独自待着，且 `--no-block` 正是让一个等待另一个 unit 作业的 unit 不死锁 systemd 的东西。手动救援一块板子，顺序就是全部技巧：

```bash
sudo systemctl stop mediad && sudo systemctl restart rkaiq_3A && sleep 2 && sudo systemctl start mediad
```

### rkaiq 的自动曝光触发一次，且仅当它抓住了流

`scripts/setup-rkaiq.sh` 首次交付时让 rkaiq 的 AE **启用**，基于这个推理：原型禁用它只是为了阻止引擎与一个自己拥有曝光的运行时打架，而 `mediad` 中没有东西拥有曝光，因此引擎应该拥有它。它确实写传感器。它不持续写。

在一个机器人上测量，在一次 unit 以正确顺序启动的启动上 —— `rkaiq_3A` 在 17:24:01，`mediad` 在 17:24:11，`wait stream start event success` 在 17:24:17：

| 什么 | 值 |
|---|---|
| `mediad` 在 17:24:17 写入 | `exposure=600 analogue_gain=1024` |
| `/dev/v4l-subdev3`，几分钟后 | `exposure: 1589  analogue_gain: 1536` |
| 一个手写的 `exposure=300 analogue_gain=256`，然后 25 秒观察 | `300 / 256`，未修正 |

因此引擎收敛了一次，到一个明显是它自己而非我们的答案，然后停了：一个在它下面变暗四倍的画面完全没有引起反应。在流开始时一次收敛不是自动曝光 —— 一个从窗户走进走廊的机器人保留着窗户的曝光。

**且在一次引擎错过流开始事件的启动上 —— 上面那节 —— 甚至那一次也不发生。** 那先被测量且被误读：传感器在被观察期间精确停在 `600 / 1024`，且一次手动写入粘住，这看起来像一个从未工作的 AE，而其实是一个从未被给予流的 AE。这两种状态从传感器的单次读数无法区分，这就是为什么两个修复属于一起：排序修复正是让它们之间的差异可观察的东西。

`mediad::exposure` 是闭合回路的东西，从原型的 `ae_loop` 移植：每秒两次从 tee 的原始分支取平均亮度对照 90 的设定点，以及一个阻尼的乘法步长按噪声顺序拆分到三个控制 —— 快门到 600 行（≈11 ms，短到不会让一个走路的机器人模糊），然后模拟增益到 11×，然后快门到 1200 行，然后 ISP 数字增益。硬快门上限是一个真正的天花板：驱动对一个长于帧长的曝光以*拉长帧时间*而非钳制来应答，因此 3500 行静默地给出 15 fps。

它做两件不同的事，因为 `mediad` 是比原型更好的地方。亮度来自已经为鸭子检测器从 tee 分接的帧，因此没有 JPEG 解码且没有东西第二次打开摄像头 —— 原型的第一版通过用一个并行 `v4l2-ctl` 采样 ISP 自身路径来测光，这在驱动层面与它自己的捕获竞争并无规律地杀死流水线。且第一次写入被读回：这能失败的每种方式（一个没有该控制的节点、一个拒绝、无 `v4l2-ctl`）否则会留下一个处于某一曝光的摄像头，与它修复的 bug 无法区分。

`setup-rkaiq.sh` 现在断言 `CommCtrl.Enable = 0`。不是因为引擎的 AE 什么都不做，而是因为它做的落在流开始 —— 正好是 mediad 的循环从它自己的起始值收敛的时候，那是两个写者为一个控制赛跑。

**在测量任何这些时 `v4l2-ctl` 会对你做的一件事：** 一个单一未知控制名会让整个 `--set-ctrl` 或 `--get-ctrl` 失败。`--get-ctrl=exposure,analogue_gain,digital_gain` 返回 `unknown control 'digital_gain'` 且无曝光，这读起来像一个两者都不携带的节点。`mediad::exposure` 因此在两次调用中写入传感器对与 ISP 的数字增益。

### 两个已知未完成的事

**比特率比目标低约 50 倍。** 针对 `bps=2000000` 的 3.3 秒捕获得到 15,553 字节，约 37 kbps。要么场景足够静态让 CBR 坍缩 —— 合理，ISP 在原始默认值上且图像在 `scripts/setup-rkaiq.sh` 运行（从原型移植；初始化与安装前钩子现在都运行它）之前是绿色且有噪点的 —— 要么捕获交付远低于 30 fps。帧数还没被测量。如果是后者，传感器模式是嫌疑：IMX219 在 3280x2464 启动，原型在每次捕获前用 `media-ctl` 固定模式（`camera.rs:277`）。

**`rawvideoparse blocksize=1382400` 是一个启动捷径，不是设计。** 它工作因为 `v4l2-ctl` 以我们计算的尺寸发射紧密打包的 NV12，且它在另一个分辨率出现步幅填充的那一刻静默出错 —— `camera.rs` 正好指出了那点。`mediad` 做它自己的 V4L2 mmap 循环进入 `appsrc` 会从驱动获得真实步幅而非假设它。

## 流水线将不得不决定的两件事

**捕获不能用 `v4l2src`。** rkisp 驱动给它一个 2 缓冲池且它重新入队太慢，丢掉每三帧 —— 一个 30 fps 传感器得到约 20 fps，带"lost frames detected"。`v4l2-ctl --stream-mmap` 维持满速率，因此 `microduck_runtime` 用它捕获并把原始帧管道进一个 `fdsrc` 流水线（`camera.rs:487`）。`mediad` 需要要么那个子进程形态，要么它自己的 V4L2 mmap 循环喂给 `appsrc`。

**四个 `mpph264enc` 属性是流水线决定，而非要继承的默认值。** 在板子上从元素读出：

| 属性 | 默认 | `mediad` 应该设什么 | 为什么 |
|---|---|---|---|
| `profile` | `high` | **`baseline`** | WebRTC 的可互操作下限是 Constrained Baseline（`profile-level-id 42e01f`）。当前浏览器协商 High；较旧对端不。设 `baseline` 产生一个 `h264parse` 报告为 `constrained-baseline` 的流 —— 已验证，非假设，因为枚举只说"baseline" |
| `header-mode` | `first-frame` | **`each-idr`** | SPS/PPS 仅在第一帧意味着一个稍后加入 —— 或丢了那个包 —— 的对端永远解不出任何东西。`reachy_mini` 的 Pi 流水线通过 `repeat_sequence_header=1` 在 `v4l2h264enc` 上正好设这个；同样要求，不同拼写 |
| `rotation` | `0` | **alpha 上 `180`** | IMX219 倒着安装。`microduck_runtime` 用 `videoflip method=rotate-180` 修复它 —— 对每一帧的一次完整 CPU 传递，在 `robotd` 共享的 SoC 上。编码器在硬件中免费做它 |
| `bps` | `0`（auto） | 一个显式目标 | `rc-mode` 已经默认为 `cbr`，那是一个有损链路想要的；比特率不该留给"auto calculate" |

结果不需要决定的两件事：

- **完全没有 B 帧旋钮**，因此 §5.5 的"无 B 帧"要求凭构造而非配置满足。
- **sink pad 接受 `NV12`**，正好是 rkisp 捕获路径发射的东西。捕获与编码之间没有 `videoconvert`，没有 RGA 色彩转换。

关键帧应该来自 `min-force-key-unit-interval` 而非一个周期性 `gop`：WebRTC 从对端的 PLI 驱动它们，且 `gop` 默认为每秒一个 IDR，无论有没有人需要。

### 约束标志值得一读，且模板比标志更糟

本页过去以要求某人验证一件事而非假设它结尾：`profile` 枚举说 `baseline`（66）而 WebRTC 协商*Constrained* Baseline，且一个避免 FMO、ASO 与冗余切片的 Baseline 流正是一个 constrained-baseline 解码器期望的。从板子读出：

```
gst-launch-1.0 -v videotestsrc num-buffers=60 ! video/x-raw,format=NV12,width=1280,height=720,framerate=30/1 ! mpph264enc profile=baseline ! h264parse ! fakesink
```

`h264parse` 在其 src pad 上协商 `profile=(string)constrained-baseline`。因此流是对的：`profile=baseline` 关掉 CABAC 与 8x8 变换，MPP 不发射 FMO、ASO 或冗余切片，且 SPS 携带 `profile_idc=66` 与 `constraint_set1_flag`。

**错的是 pad 模板，且它花了一天。** `mpph264enc` 的 src 模板列出 `profile = { baseline, main, high }` 且省略 `constrained-baseline` —— 正是 WebRTC 要求的那个 profile。那只在编码器位于 `webrtcsink` 内部而非其前面时才重要，这就是为什么这里没人先看到它：

1. `webrtcsink` 的编解码器发现以无输出 caps 构建其编码链，因此 `force_profile` 为真且它插入一个要求 `profile=constrained-baseline` 的 capsfilter。
2. `h264parse` 从 caps 查询中剥离 `alignment`、`stream-format` 与 `parsed` 但**不剥离 `profile`**，因此要求到达编码器的 src pad。
3. 与模板的交集为空。`GstVideoEncoder` 的 sink getcaps 返回空，且失败在四个元素上游浮出，为 `videorate` 报告它"could not transform NV12 … in anything we support"。
4. 发现以一个 `gst::warning!` 丢弃 H.264，VP8 被改为协商，且会话在 `rtpvp8pay` 中死亡。**没有任何错误命名 profile。**

两个教训而非一个。插件仓库现在携带一个加宽那个模板的单词补丁，作为 `v3` 发布。且 `mediad` 必须在任何这些可见之前把 GStreamer 的日志与流水线总线桥接到日志 —— 到那一点的每个媒体失败都是静默的，包括一个在协商中途结束的会话。

### 预编码不再是形态

本页过去以指出 `webrtcsink` 在其 sink pad 上接受预编码 H.264 结尾，因此 `appsrc ! mpph264enc ! h264parse ! webrtcsink` 让编码器完全不参与协商。为真，且它先工作了 —— 但它付出了两样难以加回的东西。`webrtcsink` 无法到达一个它不拥有的编码器，因此拥塞控制无法使比特率适应链路，且一个对端的 PLI 无法产生一个关键帧：一个丢了一帧的观看者会一直坏到下一个周期性 GOP。

因此 `mediad` 把原始 NV12 交给它并让它构建编码器，通过 `encoder-setup` 信号配置每一个 —— 那正是上面的表现在适用的地方。代价是编码器*确实*参与协商，这正是上面的模板缺口被发现的方式。
