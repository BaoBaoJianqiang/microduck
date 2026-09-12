# setup-gstreamer.sh

## 文件位置

`d:\microduck\scripts\setup-gstreamer.sh`

## 核心设计决策

该脚本安装 `mediad` 所需的 GStreamer 栈，并报告板端实际能编码什么。

- **为什么从 setup-board.sh 拆出**：生命周期和风险不同。
  1. **不同生命周期**：setup-board.sh 中的 overlay 修复和 ONNX 安装使板端成为机器人（无它们就没有电机总线和策略）；GStreamer 使其成为相机。`mediad` 现在发布，调用者是 provisioning（默认开启）和 updater 的 pre-install hook（每次 apply 运行 release 中的此脚本），使 GStreamer 存在前初始化的板端通过更新获得栈。
  2. **不同问题**：setup-board.sh 中一切要么工作要么失败；此脚本一半价值是末尾报告，回答板上其他东西都不回答的问题：此内核能否硬件编码 H.264，通过哪个元素。
- **预构建插件钉版本**：`mpph264enc` 和 `webrtcsink`/`webrtcsrc` 不在任何 Debian suite，CI 原生构建发布到 `pollen-robotics/microduck-gst-plugins`。`PLUGINS_VERSION="v3"`，xtask 测试断言与 `Cargo.toml` 的 `[workspace.metadata.gst-plugins]` 一致。仓库公开，因为下载在 provisioning 和 preinstall hook（清空环境无 token）中发生。
- **不重启**：所有变更无需重启。
- **钩子约束**：`hooks/preinstall` 用 10 分钟上限、无 token 环境运行此脚本，因此插件仓库必须公开且脚本不能提示。

## 常量/参数分析

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `SELF` | `/usr/local/sbin/robot-setup-gstreamer` | 脚本自拷贝位置 |
| `WANT_DEV` | `0` | 默认不装 -dev 包 |
| `GST_EXTRA_PLUGIN_DIR` | `/usr/local/lib/gstreamer-1.0` | 手建插件目录，GStreamer 默认不扫描 |
| `PLUGINS_REPO` | `pollen-robotics/microduck-gst-plugins` | 预构建插件仓库 |
| `PLUGINS_VERSION` | `v3` | 插件版本，与 Cargo.toml workspace.metadata.gst-plugins 同步 |
| `MPP_VERSION` | `1.5.0-1` | Rockchip MPP 版本 |
| `RGA_VERSION` | `2.2.0-1` | Rockchip RGA 版本 |
| `RADXA_POOL` | `https://radxa-repo.github.io/bullseye/pool/main` | Radxa deb 池 |

### RUNTIME_PKGS

`gstreamer1.0-tools gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-nice libnice10 gstreamer1.0-plugins-ugly v4l-utils`

故意不装：`gstreamer1.0-libcamera`（mainline rkisp1 不驱动 vendor rkisp）、`gstreamer1.0-plugins-rs`（Debian 无）。

### DEV_PKGS（--dev 时）

`pkg-config build-essential libssl-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-bad1.0-dev`

## 核心函数

| 函数 | 功能 |
|---|---|
| `install_missing` | 用 `dpkg -s` 逐包检查，只装缺失的（apt 启动慢，重跑应无成本） |
| `have_element` | 检查 GStreamer 元素是否注册（搜索 distro 路径 + GST_EXTRA_PLUGIN_DIR） |
| `report_encoders` | 三层编码器判定：mpph264enc（最佳）→ v4l2h264enc → x264enc（软件，临时方案） |
| `install_rockchip_userspace` | 从 Radxa 池下载 librockchip-mpp1 和 librga2（不在 Debian），插件依赖它们 |
| `install_plugins` | 下载并验证 sha256，安装到 GST_EXTRA_PLUGIN_DIR，写版本戳记 |
| `configure_vpu_access` | udev 规则给 `/dev/mpp_service` 和 `/dev/rga` 设 video 组 0660，使非 root mediad 可打开 |
| `report_webrtc` | 报告 webrtcbin/webrtcsink/webrtcsrc，缺失时给出源码构建指引 |

### 编码器判定逻辑（report_encoders）

1. **mpph264enc 存在**：硬件 H.264 通过 MPP。提示验证命令和四个关键属性（profile=baseline, header-mode=each-idr, rotation=180, bps）。
2. **/dev/mpp_service 存在但 mpph264enc 未注册**：检查插件库（ldd not found）或节点权限（0600 root:root 时编码器静默不注册）。
3. **v4l2h264enc 存在且有 M2M 节点**：可能通过 V4L2 硬件编码，需验证。
4. **都没有**：无硬件 H.264，x264enc 软件编码是临时方案。

## 关键要点总结

1. 必须 root 且 aarch64 架构运行。
2. `/dev/mpp_service` 默认 0600 root:root，非 root mpi_enc_test 写空文件且 exit 0（静默失败），udev 规则修复为 video 组 0660。
3. 插件安装到 GST_EXTRA_PLUGIN_DIR 而非 distro 目录，apt 操作不会静默替换。
4. MPP/RGA 库不在 Debian，从 Radxa 池直接下载 .deb。
5. 报告是核心价值，可重复运行回答"此板能编码什么"。
6. `main "$@"` 在最后一行，防止 `curl | sh` 截断只定义函数不执行。
