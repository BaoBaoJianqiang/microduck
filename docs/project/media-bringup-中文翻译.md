# 硬件上的媒体启动

Radxa Zero 3W 对视频做了什么。以下所有内容都是在板上观察到的，而非推断——在某些东西仍然是假设的地方，它会说明。

## 这解决了什么

`mediad` 需要硬件 H.264：软件编码不是一个更慢的选项，它根本不是选项。`jpegenc` 单独在这个 SoC 上无法在 640x480 下保持 30 fps（`microduck_runtime/src/camera.rs:500`），而 H.264 每帧比 JPEG 成本更高，在四个已经被 `robotd` 的 50 Hz 控制循环共享的 Cortex-A55 上。

**VPU 编码 H.264，比特流有效，编码器通过 Rockchip 的 MPP 而非 V4L2 到达。** 必须从源码构建两个 GStreamer 插件才能使用其中任何一个，且两者都不被任何未知的东西阻塞。

| | |
|---|---|
| VPU 编码 720p H.264 | 是——60 帧，428 KB，通过 `mpi_enc_test` |
| 此内核上比特流有效 | 是——干净的 `avdec_h264` 解码，High profile level 4，4:2:0 8-bit |
| 通过什么到达 | `/dev/mpp_service`（Rockchip MPP）。**不是** V4L2 M2M |
| 跨 1.x 的 GStreamer 插件 ABI | 不是风险——1.14 构建的插件在 1.26.2 中干净注册 |
| `mpph264enc` 元素 | 必须构建（§ [必须构建什么](#what-has-to-be-built)） |
| `webrtcsink` / `webrtcsrc` | 必须构建，单独 |

## 板上有什么

在已配置的板上没有安装任何与媒体相关的东西。`scripts/setup-gstreamer.sh` 安装它并报告硬件能做什么；该脚本是本页的可执行形式，也是有人再次需要的命令应该最终出现的地方。

GStreamer 来自**普通 Debian trixie**——`apt-cache policy` 显示 `deb.debian.org` 和 `security.debian.org`，没有 Armbian 多媒体 overlay，因此归档的版本完全适用：

| 包 | 版本 |
|---|---|
| `gstreamer1.0-plugins-bad`（有 `webrtcbin`） | 1.26.2-3+deb13u3 |
| `libgstreamer-plugins-bad1.0-dev`（有 `gstreamer-webrtc-1.0.pc`） | 1.26.2-3+deb13u3 |
| `gstreamer1.0-nice` | 0.1.22-1 |
| `gstreamer1.0-plugins-rs` | **在任何 Debian 套件中都不存在** |

内核是 `6.1.115-vendor-rk35xx`。这在两个方面重要：摄像头的 MIPI-CSI ISP 捕获驱动只存在于 Armbian 的厂商分支上，VPU 节点也是。`setup-board.sh` 已经安装了那个内核——为了音频编解码器的 I²S 树，不是为了视频——因此先决条件在任何人要求之前就已满足。一个拉取 `current` 内核并重新指向 `/boot` 的意外 `apt upgrade` 会把两者都带走。

## 摄像头需要 overlay，在前缀下镜像

插入的 CSI 摄像头在其设备树 overlay 启用之前**不产生 `/dev/video*` 且 dmesg 中什么都没有**——这读起来完全像一个没有连接的摄像头。

启用它有一个值得单独说明的陷阱。Armbian 将 overlay 运送为 `radxa-zero3-rpi-camera-v2.dtbo`，**没有 `rk3568-` 前缀**，而板运行 `overlay_prefix=rk3568`。因此一个 `overlays=` 词解析为 `rk3568-radxa-zero3-rpi-camera-v2.dtbo`，加载器什么都找不到，板愉快地启动且没有摄像头——与 `configure_overlay` 为 `uart2-m0` 防止的相同静默失败。文件必须**首先在前缀名称下镜像**，然后才能在 `overlays=` 中命名。`microduck_runtime/install.sh` 遇到了这个并做了同样的事。

`setup-board.sh` 中的 `configure_camera` 两者都做，写入*厂商*内核的 overlay 目录——MIPI-CSI 捕获驱动只存在于那个分支上，这是厂商内核不是可选的第二个原因。`DUCK_CAMERA_OVERLAY` 选择另一个模块；Armbian 每个传感器运送一个，对于这块板是 `radxa-zero3-rpi-camera-v2`（Pi Cam v2 / IMX219）或 `radxa-zero3-rpi-camera-v1.3`（Pi Cam v1.3 / OV5647）。这里只使用过第一个。

## 编码器是 MPP，不是 V4L2

`v4l2h264enc` 不存在且 `/dev/video*` 为空。两者都不是故障：

- 在 Rockchip BSP 内核上，VPU 暴露为 `/dev/mpp_service`，而不是 V4L2 M2M 编码器。`v4l2h264enc` 只有在 `gstreamer1.0-plugins-good` 找到编码器节点时才注册，因此它的缺失是这里预期的形状，而不是缺少包。
- 完全没有 `/dev/video*` 也正是一个**未连接的摄像头**的样子——rkisp 捕获节点只有在传感器被探测后才出现。

这值得直白说明，因为它是分支点。如果内核暴露了 V4L2 编码器，硬件 H.264 根本不需要任何树外的东西。

### 权限陷阱

`/dev/mpp_service` 以 `crw------- root root`，模式 0600 到达。非 root 进程无法打开它——而且**针对它的 `mpi_enc_test` 写入一个空文件并以 0 退出**。没有错误，没有日志行。因此零退出状态什么都不能证明；文件大小才是证据。

`mediad` 将作为自己的用户运行，就像这里的每个其他守护进程一样——`tofd` 进入 `i2c`，`padd` 进入 `input`，`btd` 进入 `bluetooth`——因此 VPU 需要相同的处理：一个给节点分配组的 udev 规则，以及单元上的 `SupplementaryGroups=`。`scripts/setup-gstreamer.sh` 安装该规则（`99-robot-mpp.rules`，组 `video`，模式 0660），遵循 `setup-board.sh` 中 `configure_tof` 的 i2c 规则的形状。

用 `video` 而不是 `robot`：`robot` 门控我们*定义*的 IPC socket（[`app-path-design.md`](../design/app-path-design.md) §，socket-mode-plus-group 分层）。内核设备节点不是我们可以重新定义的，而 `video` 是这个设备类的发行版约定，因此用 `gst-launch` 的开发者以与 `mediad` 相同的方式进入。

## Radxa 的池提供什么

Rockchip MPP 不在 Debian 中。Radxa 将其作为 GitHub Pages apt 仓库发布，包被作为**直接 `.deb` 下载**获取，而不是通过将仓库添加到 `sources.list`——这是 `microduck_runtime/radxa_setup/setup_rkaiq.sh` 已经在这块板上为 `rkaiq_3A_server` 使用的路线。基础：`https://radxa-repo.github.io/bullseye/pool/main`。

| 包 | 版本 | 为什么 |
|---|---|---|
| `m/mpp/librockchip-mpp1` | 1.5.0-1 | MPP 用户空间库 |
| `m/mpp/librockchip-vpu0` | 1.5.0-1 | `rockchip-mpp-demos` 恰好依赖这个版本 |
| `m/mpp/rockchip-mpp-demos` | 1.5.0-1 | `mpi_enc_test`——在不涉及 GStreamer 的情况下证明 VPU |
| `m/mpp/librockchip-mpp-dev` | 1.5.0-1 | 头文件，用于构建编码器插件 |
| `libr/librga/librga2` | 2.2.0-1 | Rockchip 2D 加速器；rockchip 插件依赖它 |
| `libr/librga/librga-dev` | 2.2.0-1 | 头文件，相同构建 |
| `g/gstreamer1.0-rockchip/gstreamer1.0-rockchip1` | 1.14-4 | MPP GStreamer 插件——见下文 |

这些是 bullseye 构建，它们在 trixie 上针对 glibc 2.41 干净配置。

**`dpkg -i` 不解析任何东西**，因为这些不来自配置的 apt 源。每个缺失的依赖都是一个未配置的包，而不是一个自我修复的安装，因此每个集合都必须命名其完整闭包。这花了三次往返才学会。

### Radxa 的预构建插件*不是*仅解码的，本页曾说它是

`gstreamer1.0-rockchip1_1.14-4` 在 GStreamer 1.26.2 中安装并干净注册，恰好显示 `mppvideodec` 和 `mppjpegdec`。这读起来像"没有编码器"，本页曾这样声称——错了。对其 `.so` 运行 `strings` 列出了 `mpph264enc`、`mpph265enc`、`mppjpegenc` 和 `mppvp8enc`。它们都在那里。

**权限陷阱是全部解释，在弄清楚之前它产生了四个独立的误导性结果：**

| 看起来像什么 | 实际是什么 |
|---|---|
| `mpi_enc_test` 什么都没写且**以 0 退出** | 节点无法打开；零退出什么都不说明 |
| Radxa 的 deb 仅解码 | 它有每个编码器 |
| 第三方 1.14-8 deb 仍然没有显示 `mpph264enc` | 又是同样的原因 |
| 我们自己的 CI 构建只列出两个解码器 | 容器也没有 `/dev/mpp_service`——预期，不是失败 |

MPP 插件**无条件注册其解码器，并在注册编码器之前探测 MPP。** 当 `/dev/mpp_service` 为 `0600 root:root` 时，探测静默失败，因此编码器从一个完全包含它们的插件中被省略。

因此一个只列出解码器的插件是关于*设备节点*的证据，而不是关于插件的，在 udev 规则到位之前，`gst-inspect-1.0 mpph264enc` 什么都不意味。

那次安装确实证明了一件事，且它成立：**针对 GStreamer 1.14 构建的插件在 1.26.2 中毫无怨言地注册。** 插件 ABI 曾是害怕源码构建的陈述理由，而它不是风险。

## 必须构建什么

两个插件，出于两个不相关的原因。两者都不替代对方。

| 插件 | 来源 | 提供 | 为什么不能安装 |
|---|---|---|---|
| `gstreamer-rockchip` | [`JeffyCN/mirrors`](https://github.com/JeffyCN/mirrors) 分支 `gstreamer-rockchip`，meson | `mpph264enc`——硬件编码器 | Debian 根本没有 Rockchip 编码器。Radxa 的构建*确实*有它们，因此这个是关于我们控制的一个 pin，去掉 `libx11-6`，并与下面的插件一起 |
| `gst-plugin-webrtc` | [`gst-plugins-rs`](https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs) 0.15.3，cargo-c | `webrtcsink`、`webrtcsrc` | `gst-plugins-rs` 在**任何** Debian 套件中都没有被打包 |

用 0.15.3 而不是 `reachy_mini` SDK 记录的 0.14.5：0.14.5 是重要的下限——低于它缺少一个 `webrtcsink` 在远程描述和 ICE 处理之间的死锁修复，表现为客户端在"connecting"上永远旋转——而 0.15.3 只是更新。两个系列都声明 GStreamer `v1_22` 特性下限，机器人运行 1.26.2，因此更新的那个不花任何成本。

`webrtcbin` **已**安装，来自 `gstreamer1.0-plugins-bad`。因此今天无需第二个构建就可以到达 WebRTC 会话——代价是自己实现信令协议。`webrtcsink` 更受青睐，因为它的信令协议是中继代理的东西，这就是使中央信令服务器可复用的原因。

### 为什么不用预构建的

硬件、内核驱动和 MPP 的用户空间库都在**没有编译任何东西**的情况下工作——`mpi_enc_test` 来自一个 deb，第一次尝试就编码了 720p H.264。缺少的只是*GStreamer 绑定*：一个将 `librockchip-mpp` 包装为流水线可以使用的元素的插件。`mpi_enc_test` 是一个独立程序；GStreamer 不知道它存在。与这里的 ONNX Runtime 形状相同——`libonnxruntime.so` 从 tarball 安装，没有编译任何东西，而 `ort` 是使其可达的绑定。

预构建绑定：

| 来源 | 有什么 |
|---|---|
| Radxa `bullseye` 池 | `gstreamer1.0-rockchip1_1.14-4`——已安装并检查：只有 `mppvideodec` + `mppjpegdec` |
| Radxa `rk3588s2-bookworm` 池 | 相同的 `1.14-4`，字节相同 |
| [`numbqq/gstreamer-rockchip-debs`](https://github.com/numbqq/gstreamer-rockchip-debs) | `1.14-8`——**有每个编码器** |

最后一个值得在构建任何东西之前尝试。它的 `bookworm/arm64/<board>/` 条目是指向 `jammy/arm64/` 的符号链接，因此它是一个 Ubuntu 22.04 构建，来自 `rockchip-linux/gstreamer-rockchip`（现在 404），Jeffy Chen 作为维护者——与 Radxa 构建的相同上游，在一个启用了编码器的修订版。`mpph264enc`、`mpph265enc`、`mppjpegenc` 和 `mppvp8enc` 都存在于 `.so` 中。

它的 `DT_NEEDED` 被板在上面的 deb 之后已经拥有的东西满足：`librockchip_mpp.so.1`、`librga.so.2`、`libgstreamer-1.0.so.0`、`libgstvideo`、`libgstallocators`、`libgstpbutils`、`libdrm2`、`libglib2.0-0`、`libx11-6`、`libc6 >= 2.33`，针对 glibc 2.41。其中没有任何东西是 RK3588 特有的——SoC 差异存在于 MPP 内部，而不是插件中——且 `Depends` 只从下方限定 GStreamer（`>= 1.14`）。

我们的在 [`microduck-gst-plugins`](https://github.com/pollen-robotics/microduck-gst-plugins) 中构建——一个独立的仓库，故意如此：

- **不在板上。** RK3566 编译 Rust 部分太慢了，等不起。
- **也不交叉编译。** daemon 用 `cargo-zigbuild` 交叉构建，`scripts/ci-cross-deps.sh` 直白地说它的一个 C 依赖"是那个例外的成本，在添加另一个之前值得一读"。GStreamer 将是一个大得多的第二个，且两条路线——x86 multiarch，或带 meson cross 文件的 sysroot——都链接到目标的近似值。
- **原生地，在 `debian:trixie` 容器中的 arm64 运行器上**，这是机器人自己的用户空间。没有东西被近似。arm64 运行器在公开仓库上是免费的。
- **公开**还有第二个更重要的原因：下载发生在配置期间和更新器的 `preinstall` 钩子中，它以清空的环境和**没有 token**运行。与 daemon 已经依赖的 ONNX Runtime 相同的安排。

它构建两个插件，在一个 `pins.env` 中按提交或标签 pin 上游，禁用 `rkximage` 和 `kmssrc`（同一棵树中的 X11 和 KMS sink——无头机器人两者都不需要，且它们是预构建 Radxa deb 依赖 `libx11-6` 的原因），并发布一个 tarball 加其 sha256，带有一个命名每个插件确切上游 ref 的 `MANIFEST`。那个清单是第三方 deb 无法回答的东西。

它守卫的两个陷阱，都是通过阅读树而非失败发现的：`gst/rockchipmpp/meson.build` 以 `if not mpp_dep.found() → subdir_done()` 结尾，因此缺失的 `librockchip-mpp-dev` 使 meson **跳过插件并成功**；且对于直接 `.deb` 下载 `dpkg -i` 不解析任何东西，因此 Radxa 闭包在一次调用中安装。

**`mediad.service` 必须设置 `GST_PLUGIN_PATH`。** 插件安装到 `/usr/local/lib/gstreamer-1.0`，GStreamer 默认**不**搜索它——其内置路径是发行版的 `/usr/lib/aarch64-linux-gnu/gstreamer-1.0`，且该目录被故意避免，因此 `apt` 操作不能替换或移除它们。因此单元需要

```
Environment=GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0
```

以及 VPU 节点需要的 `SupplementaryGroups=video`。两者都容易忘记，且两者在运行时表现相同：编码器根本不存在，没有任何东西说明为什么。

`scripts/setup-gstreamer.sh` 以一个**固定的**版本消费它——永远不是"latest"。相隔一天的两次配置运行产生不同的插件，且没有任何东西记录是哪个，是一个等待发生的不可复现媒体 bug。pin 存在于 `Cargo.toml` 的 `[workspace.metadata.gst-plugins]` 中，脚本携带字面量因为它是用 `curl` 独立获取的，且一个 `xtask` 测试断言它们一致——与 `ONNX_VERSION` 相同的安排和相同的原因。

**尝试第三方 deb 与依赖它不同。** 它是一个人的每板转储，没有我们控制的来源，如果那个仓库消失它也消失。它廉价买到的是关于构建的唯一真实问题的答案——这个插件是否针对*我们的* MPP 和 GStreamer 版本工作——如果它工作，自己构建相同的源码就被去风险了，而不是不必要了。我们自己的固定构建仍然是这应该最终到达的地方。

**完全可以避免构建**，通过在 `mediad` 中通过 Rust FFI 调用 MPP 的 C API，`mpi_enc_test` 证明这可行。这用一个 meson 构建换取手写和手维护的厂商库绑定，这是交易中更差的一面——但如果插件被证明与 GStreamer 1.26 对抗，这是一个真实的选项，不是死胡同。

### 上游

`rockchip-linux/gstreamer-rockchip` 没了（404）。`JeffyCN/mirrors@gstreamer-rockchip` 是活跃镜像——最后提交 2026-05-21——且 `gst/rockchipmpp` 持有 `gstmpph264enc.c`、`gstmpph265enc.c`、`gstmppjpegenc.c`、`gstmppvp8enc.c`。周围存在 fork 汤；无论使用哪个 fork 和标签都必须被固定和记录，原因与 `gst-plugin-webrtc` 被固定到 ≥ 0.14.5 相同（见下文）。

### `gst-plugin-webrtc` 上的 pin

**0.14.5 或更新，不是 0.14.4。** 更早的标签缺少一个 `webrtcsink` 在远程描述和 ICE 处理之间的死锁修复，表现为客户端在"connecting"上永远旋转。`reachy_mini` 的 SDK 安装文档记录了它；注意 `reachy-mini-desktop-app`  vendored 0.14.4，因此在那条线的错误一侧。

Pollen 已经为 **x86_64** vendored 这个插件——用 `cargo cinstall` 原生构建，strip，按架构提交，并被 CI 固定到一个提交 ref 加 sha256 消费。在同一个地方加一个 `aarch64/` 兄弟可能比这里的第二个流水线总工作量更少。

### 构建的插件属于哪里

在 daemon 发布载荷中，`GST_PLUGIN_PATH` 指向 `current`——不在 apt 中，也不在每台机器的 `/opt` 中。

插件版本和 `mediad` 的代码是纠缠的：上面的 0.14.5 故事正是一个插件版本决定 daemon 是否需要变通方案的案例。偏差是一个 `mediad` bug，因此它想要 `mediad` 的生命周期——原子交换、回退、健康门。`librockchip-mpp` 相反：一个与*内核*配对的系统库，被任何触及 VPU 的东西需要，属于包管理器。

## 已测量，和未测量

**在板上测量：** GStreamer 1.26.2 及其来源；`webrtcbin` 存在；`webrtcsink`/`webrtcsrc` 不存在；`v4l2h264enc` 不存在且没有 `/dev/video*`；`/dev/mpp_service` 存在于 0600 root:root；`mpi_enc_test` 作为非 root 静默什么都不写，作为 root 写 428 KB；该比特流干净解码为 High/4.0；Radxa deb 的依赖闭包；rockchip 插件在 1.26.2 中加载，有两个解码元素。

**编码路径在硬件上端到端关闭。** 按顺序：

1. `v1` 从公开发布获取，sha256 验证，安装到 `/usr/local/lib/gstreamer-1.0`。
2. `gst-inspect-1.0 mpph264enc` 回答 `provided-by /usr/local/lib/gstreamer-1.0/libgstrockchipmpp.so`——我们的构建。第三方 deb 首先被移除，因此答案可以归因于某个东西。
3. 它**编码**：`videotestsrc ! mpph264enc profile=baseline header-mode=each-idr bps=2000000 ! h264parse ! filesink` 为 60 帧 720p 产生了 476 KB，耗时 **0.44 秒墙钟**，包括源生成和流水线设置——轻松快于实时。结果通过 `avdec_h264` 干净解码。
4. 它**无需 root** 工作：在 udev 规则将 `/dev/mpp_service` 置于 `660 root:video` 且用户加入 `video` 后，`gst-inspect` 作为该用户回答。这就是 `mediad` 所处的情况，且在此之前的每个检查都是在 `sudo` 下进行的。

**整个媒体链在硬件上关闭。** 传感器到 WebRTC 可协商的流：

| 步骤 | 证据 |
|---|---|
| overlay 已应用 | `csi2-dphy0` 被探测，`rkisp` 启动，十个 `/dev/videoN` |
| 传感器识别 | `imx219 2-0010: Model ID 0x0219, Lot ID 0x5a8e73, Chip ID 0x0773` |
| 捕获节点 | `/dev/video0`，卡名 `rkisp_mainpath`，格式到 3280x2464 |
| 帧 | 720p NV12 下 `--stream-count=10` 为 13,824,000 字节——精确 |
| 硬件编码 | `v4l2-ctl … --stream-to=-` 进入 `fdsrc ! rawvideoparse ! mpph264enc` |
| 流 | 干净解码；`h264parse` 报告 **1280x720 constrained-baseline** |

节点号在启动间不稳定，因此捕获节点通过匹配 `/sys/class/video4linux/*/name` 下的卡名 `rkisp_mainpath` 找到——如 `camera.rs:219` 所做。

### 三个设备节点需要 `video` 组，不是一个

这花了三轮独立调试，每轮都有一个命名其他东西的失败：

| 节点 | 仅 root 时的症状 |
|---|---|
| `/dev/mpp_service` | `mpi_enc_test` 什么都不写且**以 0 退出**；`mpph264enc` 根本不注册 |
| `/dev/rga` | 元素存在，流水线启动，然后 `Try to use uninit rgaCtx=(nil)` 和成页的 `rga call blit fail` |
| `/dev/video0` | 已经是 `root:video`，因此它是不咬人的那个 |

`setup-gstreamer.sh` 安装一个覆盖前两个的 udev 规则。**`mediad.service` 需要 `SupplementaryGroups=video`**——加上 `Environment=GST_PLUGIN_PATH`，这是站在工作流水线和四种不同的令人困惑的失败之间的两行。

### 花费 22 fps 的旋转

摄像头安装时偏了四分之一圈，显而易见的修复——在 tee 之前放 `videoflip`，这样每个消费者都得到正立的画面——是错误的，在机器人上测量：

| | RGA 失败 | `v4l2src` 丢失的帧 | fps | SoC |
|---|---|---|---|---|
| 没有翻转 | 0 | 0 | ~30 | 正常 |
| 有翻转 | 一个会话中 5522 次 | 1565 | 7–8 | 97 °C，CPU 在 408 MHz |

`mpph264enc` 将 UYVY→NV12 转换交给 SoC 的 2D 引擎且不为此付费。`videoflip` 的输出是 RGA 拒绝的缓冲区——`10000 is unsupport format`，然后是 `RGA_BLIT fail: Bad address` 在一个 `rect[0, 0, 720, 1280]` 上——因此 MPP 回退到在软件中转换*每一帧*。这使 CPU 饱和，SoC 达到其温度极限，一切都节流到 408 MHz，摄像头丢弃它无法交付的帧。旋转本身是账单中较小的一半。

因此流水线中没有任何东西旋转。安装被*报告*（`media.video`，每个控制通道一次），显示画面的人旋转它：控制台用 CSS 变换做，这是免费的，感知消费者将其折叠进它已经做的重采样。`--flip-in-pipeline` 为无法自己旋转且能负担得起的消费者恢复旧行为。

如果真正需要正立的流，正确的修复是插件集中的一个 RGA 元素：2D 引擎免费旋转，这就是编码器能用得起它的原因。

### 3A 引擎必须在流开始之前等待

`rkaiq_3A_server` 附加到 ISP，然后等待一个**流开始**事件——它会错过已经发生的一个。在 `mediad` 正在流式传输时重启它，它会停在

```
DBG: /dev/media0: wait stream start event...
```

上永远：没有统计循环，因此没有自动曝光和没有白平衡，画面是绿色的。重启"修复"它只是因为重启恰好将两者排序正确。

这使它成为一个没有明显原因的回归，因为**每次 `robotctl update apply` 都在运行的流下面重启引擎**：预安装钩子运行 `setup-rkaiq.sh`，而那个脚本重启 `rkaiq_3A`。它甚至打印了"restart the camera stream for it to take effect"——一条没人遵循且没有东西强制执行的建议。

因此该脚本安装的 drop-in 现在携带不变量：

```ini
ExecStartPost=-/bin/systemctl --no-block try-restart mediad.service
```

每当引擎启动时，流在它后面被弹跳，因此它等待的事件是一个尚未发生的事件。`try-restart` 让没有 `mediad` 的板独自待着，而 `--no-block` 是防止一个单元等待另一个单元的作业而死锁 systemd 的东西。手动救援一块板，顺序就是全部诀窍：

```bash
sudo systemctl stop mediad && sudo systemctl restart rkaiq_3A && sleep 2 && sudo systemctl start mediad
```

### rkaiq 的自动曝光触发一次，且仅在它捕获到流时

`scripts/setup-rkaiq.sh` 最初交付时保持 rkaiq 的 AE **启用**，理由是：原型禁用它只是为了阻止引擎与自己拥有曝光的运行时对抗，而 `mediad` 中没有东西拥有曝光，因此引擎应该拥有它。它确实写传感器。它不会持续写。

在机器人上测量，在一次单元以正确顺序启动的启动上——`rkaiq_3A` 在 17:24:01，`mediad` 在 17:24:11，`wait stream start event success` 在 17:24:17：

| 什么 | 值 |
|---|---|
| `mediad` 在 17:24:17 写入 | `exposure=600 analogue_gain=1024` |
| `/dev/v4l-subdev3`，几分钟后 | `exposure: 1589  analogue_gain: 1536` |
| 手写 `exposure=300 analogue_gain=256`，然后观察 25 秒 | `300 / 256`，未被纠正 |

因此引擎收敛了一次，到一个明显是它自己的而非我们的答案，然后停止了：在它下面把画面弄暗四倍没有引起任何反应。流开始时的一次收敛不是自动曝光——一个从窗户走进走廊的机器人会保持窗户的曝光。

**而在引擎错过流开始事件的启动上——上面的小节——甚至那一次都不会发生。** 那是先被测量的且被误读了：传感器在被观察的整个时间内恰好停在 `600 / 1024`，手动写入卡住了，这看起来像一个从未工作的 AE，而实际上是一个从未被给予流的 AE。从传感器的单次读取中这两种状态无法区分，这就是为什么两个修复属于一起：排序修复是使它们之间的差异可观察的东西。

`mediad::exposure` 是闭环的东西，从原型的 `ae_loop` 移植：每秒两次从 tee 的原始分支取平均亮度对照 90 的设定值，以及一个阻尼乘法步长，按噪声顺序拆分到三个控件——快门到 600 行（≈11 ms，短到不会使行走的机器人模糊），然后模拟增益到 11×，然后快门到 1200 行，然后 ISP 数字增益。硬快门上限是一个真实的天花板：驱动对长于帧长度的曝光的回答是*拉伸帧时间*而不是钳制，因此 3500 行静默地给出 15 fps。

它做两件不同的事，因为 `mediad` 是比原型更好的地方。亮度来自已经为鸭子检测器从 tee 分出的帧，因此没有 JPEG 解码且没有东西第二次打开摄像头——原型的第一个版本通过用并行的 `v4l2-ctl` 采样 ISP 自路径来测光，这在驱动级别与其自己的捕获争用并间歇性地杀死流水线。且第一次写入被读回：否则它可能失败的每种方式（没有控件的节点、拒绝、没有 `v4l2-ctl`）都会留下一个在一个曝光下的摄像头，这与它修复的 bug 无法区分。

`setup-rkaiq.sh` 现在断言 `CommCtrl.Enable = 0`。不是因为引擎的 AE 什么都不做，而是因为它做的事情落在流开始时——恰好当 mediad 的循环正在从其自己的起始值收敛时，这是两个写者为一个控件竞争。

**在测量这一切时 `v4l2-ctl` 会对你做的一件事：** 单个未知控件名称会使整个 `--set-ctrl` 或 `--get-ctrl` 失败。`--get-ctrl=exposure,analogue_gain,digital_gain` 返回 `unknown control 'digital_gain'` 且没有曝光，这读起来像一个两者都不携带的节点。`mediad::exposure` 因此在两次调用中写入传感器对和 ISP 的数字增益。

### 已知未完成的两件事

**比特率比目标低约 50 倍。** 3.3 秒捕获 15,553 字节，对照 `bps=2000000` 约为 37 kbps。要么场景足够静态使 CBR 崩溃——合理，ISP 在原始默认值上且在 `scripts/setup-rkaiq.sh` 运行（从原型移植；配置和预安装钩子现在都运行它）之前画面是绿色且有噪声的——要么捕获交付远低于 30 fps。帧数尚未测量。如果是后者，传感器模式是嫌疑人：IMX219 以 3280x2464 启动，原型在每次捕获前用 `media-ctl` 固定模式（`camera.rs:277`）。

**`rawvideoparse blocksize=1382400` 是一个启动捷径，不是设计。** 它有效因为 `v4l2-ctl` 发出我们计算的大小的紧密打包 NV12，而一旦在另一个分辨率出现步幅填充它就静默错误——`camera.rs` 恰好指出了这一点。`mediad` 做自己的 V4L2 mmap 循环进入 `appsrc` 从驱动获取真实步幅，而不是假设它。

## 流水线必须决定的两件事

**捕获不能使用 `v4l2src`。** rkisp 驱动给它一个 2 缓冲池，它重新排队太慢，丢弃每三帧——从 30 fps 传感器得到约 20 fps，伴随"lost frames detected"。`v4l2-ctl --stream-mmap` 维持全速率，因此 `microduck_runtime` 用它捕获并将原始帧管道到一个 `fdsrc` 流水线（`camera.rs:487`）。`mediad` 需要要么那个子进程形状，要么它自己的 V4L2 mmap 循环喂给 `appsrc`。

**四个 `mpph264enc` 属性是流水线决定，不是要继承的默认值。** 在板上从元素读取：

| 属性 | 默认 | `mediad` 应该设置 | 为什么 |
|---|---|---|---|
| `profile` | `high` | **`baseline`** | WebRTC 的可互操作下限是 Constrained Baseline（`profile-level-id 42e01f`）。当前浏览器协商 High；较旧的对等方不。设置 `baseline` 产生一个 `h264parse` 报告为 `constrained-baseline` 的流——已验证，不是假设，因为枚举只说"baseline" |
| `header-mode` | `first-frame` | **`each-idr`** | SPS/PPS 只在第一帧*意味着*一个稍后加入的对等方——或丢失那个包的对等方——永远无法解码任何东西。`reachy_mini` 的 Pi 流水线在 `v4l2h264enc` 上通过 `repeat_sequence_header=1` 设置恰好这个；相同要求，不同拼写 |
| `rotation` | `0` | 在 alpha 上 **`180`** | IMX219 倒置安装。`microduck_runtime` 用 `videoflip method=rotate-180` 修复它——对每一帧的完整 CPU 传递，在 `robotd` 共享的 SoC 上。编码器在硬件中免费做 |
| `bps` | `0`（自动） | 一个显式目标 | `rc-mode` 已经默认为 `cbr`，这是有损链路想要的；比特率不应该留给"自动计算" |

两件事结果不需要决定：

- **根本没有 B 帧旋钮**，因此 §5.5 的"无 B 帧"要求通过构造而非配置满足。
- **sink pad 接受 `NV12`**，这正是 rkisp 捕获路径发出的。捕获和编码之间没有 `videoconvert`，也没有 RGA 色彩转换。

关键帧应该来自 `min-force-key-unit-interval` 而不是周期性的 `gop`：WebRTC 从对等方的 PLI 驱动它们，而 `gop` 默认每秒一个 IDR，无论是否有人需要。

### 约束标志值得一读，而模板比标志更糟

本页曾经以要求某人验证一件事而非假设它结尾：`profile` 枚举说 `baseline`（66）而 WebRTC 协商*Constrained* Baseline，而一个避免 FMO、ASO 和冗余切片的 Baseline 流正是 constrained-baseline 解码器期望的。在板上读取：

```
gst-launch-1.0 -v videotestsrc num-buffers=60 ! video/x-raw,format=NV12,width=1280,height=720,framerate=30/1 ! mpph264enc profile=baseline ! h264parse ! fakesink
```

`h264parse` 在其 src pad 上协商 `profile=(string)constrained-baseline`。因此流是对的：`profile=baseline` 关闭 CABAC 和 8x8 变换，MPP 不发出 FMO、ASO 或冗余切片，且 SPS 携带 `profile_idc=66` 和 `constraint_set1_flag`。

**错的是 pad 模板，它花了一天。** `mpph264enc` 的 src 模板列出 `profile = { baseline, main, high }` 并省略了 `constrained-baseline`——WebRTC 要求的那个 profile。这只有在编码器在 `webrtcsink` 内部而不是在它前面时才重要，这就是为什么这里首先没有东西看到它：

1. `webrtcsink` 的编解码器发现构建其编码链时没有输出 caps，因此 `force_profile` 为 true，它插入一个要求 `profile=constrained-baseline` 的 capsfilter。
2. `h264parse` 从 caps 查询中剥离 `alignment`、`stream-format` 和 `parsed`，但**不剥离 `profile`**，因此需求到达编码器的 src pad。
3. 与模板的交集为空。`GstVideoEncoder` 的 sink getcaps 返回空，失败在四个元素上游表现为 `videorate` 报告它"could not transform NV12 … in anything we support"。
4. 发现丢弃 H.264 并伴随一个 `gst::warning!`，改为协商 VP8，会话在 `rtpvp8pay` 中死亡。**任何地方都没有错误命名 profile。**

两个教训而非一个。插件仓库现在携带一个一词补丁加宽那个模板，作为 `v3` 发布。且 `mediad` 必须在任何这可见之前将 GStreamer 的日志和流水线总线桥接到日志中——在此之前的每个媒体失败都是静默的，包括一个在协商中途结束的会话。

### 预编码不再是形状

本页曾经以指出 `webrtcsink` 在其 sink pad 上接受预编码的 H.264 结尾，因此 `appsrc ! mpph264enc ! h264parse ! webrtcsink` 使编码器完全脱离协商。确实如此，且它首先工作——但它付出了两个难以加回的代价。`webrtcsink` 无法到达它不拥有的编码器，因此拥塞控制无法使比特率适应链路，且对等方的 PLI 无法产生关键帧：丢失一个的观看者保持损坏直到下一个周期性 GOP。

因此 `mediad` 交给它原始 NV12 并让它构建编码器，通过 `encoder-setup` 信号配置每个——这就是上面的表格现在适用的地方。代价是编码器*确实*到达协商，这就是上面的模板差距如何被发现的。
#（注：内容由AI生成）
