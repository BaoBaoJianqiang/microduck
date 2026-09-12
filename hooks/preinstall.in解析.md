# 解析：`hooks/preinstall.in`

## 这是什么

**更新前置检查与运行时库安装脚本**（sh，232 行，约七成是注释）的**模板**。`xtask package` 从 `Cargo.toml` 的 `[workspace.metadata.onnxruntime]` 替换 `@ONNX_FLOOR@` 与 `@ONNX_TARGET@`，把结果作为 `hooks/preinstall` 随发布版发布。头注释强调：**编辑本文件，永不编辑生成的那个**——而且不存在"需要保持同步的副本"，因为 hook 与发布是从同一个常量构建出来的。

## 运行时机与失败哲学

在 artifact 下载、验证、解包**之后**、符号链接交换**之前**运行。头注释第 9-13 行："这个位置就是全部价值"——在这里失败会中止更新，**旧发布仍然活着**，无回滚、无 boot-counter 计数消耗。对比真实发生过的事故：完整下载 → 交换 → 重启 → 30 秒健康门 → 回滚，唯一的解释只有一行 `control loop has not completed a cycle yet`。

## 环境与两个依赖的不同失败方式

更新器清空环境、只留 `PATH` 与 hook 上下文，所以**这里没有 token**——没关系，本钩子抓取的两样东西都是公开资源：`microsoft/onnxruntime` 的 ONNX Runtime 与 `pollen-robotics/microduck-gst-plugins` 的 GStreamer 插件。

两个依赖**故意以不同方式失败**：

| 依赖 | 失败方式 | 理由 |
|---|---|---|
| ONNX Runtime | **致命**（exit 1） | 无法加载策略的发布是一台无法站立的机器人；在此中止，旧发布仍活着 |
| GStreamer 栈 | 仅警告 | 没有摄像头栈损失 `mediad`（视频与 WebRTC 控制台），但仍走路、仍配对、仍更新；为此拒绝更新 = 一块"无法被修复板子的机制所修复"的板子 |

## 脚本骨架

```sh
set -eu
ONNX_FLOOR="@ONNX_FLOOR@"
ONNX_TARGET="@ONNX_TARGET@"
ONNX_LIB_DIR=/usr/local/lib
CURL_MAX_TIME=90
```

`CURL_MAX_TIME=90` 低于更新器的 hook 120 秒上限（注释：慢网络产生的是**我们的、带修复指引的消息**，而不是一句裸的 `timed out after 120s`）。`say`/`die` 两个辅助函数，`die` 写 stderr 并 exit 1。

## 三个工具函数

### `installed_onnx()`（第 46-52 行）

读 `/usr/local/lib/libonnxruntime.so` 符号链接的解析目标，从 `libonnxruntime.so.<version>` 提取版本号。注释解释为何不运行库本身来问：tarball 同时铺设带版本的真文件与裸名符号链接，所以解析目标**无需运行任何东西**就给出版本；而询问库自身意味着 dlopen 它——"那正是我们试图避免不安全去做的事"。

### `at_least()`（第 59-84 行）

数值比较 major.minor，**而非字符串比较**：`"1.9"` 按字典序排在 `"1.23"` 之上，字符串比较会让一块跑不动发布的板子通过检查。忽略 patch 版本——`ort` 关心的是 API 版本，它随 minor 移动。还防御了完全非数字的版本串，不让 `test` 在其上嘈杂地失败。

### `install_onnx()`（第 92-123 行）

从 GitHub releases 下载 aarch64 tarball（`mktemp` + `trap` 清理临时目录），解包找到 `libonnxruntime.so.*`，`install -m 0644` 后重建裸名符号链接，再刷新动态链接缓存——`ldconfig` 住在 `/usr/sbin`，hook 的最小 PATH 包含它，但仍三重检查（`command -v` → 绝对路径 → 提示设 `ORT_DYLIB_PATH`），注释说因为"一个 dlopen 找不到的新拷贝的库，正是本钩子存在要防止的那个失败"。失败时 `die` 输出完整的修复指引（有网重试 / 手动装 / 跑 `setup-board.sh`）。

一个微妙点（注释第 88-91 行）：这**写在交换之前的 `/usr/local/lib`**，所以变更会幸存于一次被中止的更新——这是故意的且方向安全：`ort` 向动态库要求"至少其 `ORT_API_VERSION`"，而 ONNX Runtime 保持 C API 向后兼容，**被放弃的旧发布对新运行时仍然工作**。

## 四个安装函数（全部"永不致命"）

### `install_gstreamer()`（第 142-157 行）

运行发布自带的 `scripts/setup-gstreamer.sh`。注释加粗强调：**运行脚本本身，而不是复制它所做的事**——那个脚本拥有钉住的插件版本、Debian 包列表、Rockchip MPP 与 RGA 的 deb、`/dev/mpp_service` 的 udev 规则、编码器报告，"六样本会存在两份并漂移开的东西，正是本仓库反复写下的那类 bug"。幂等：已配置的板子只付一个 stamp 文件比较加两次 `dpkg -s`；从未有过该栈的板子在这里得到它。

这正是 provisioning 与 updating 分裂的目的：`provision.sh` 在新板上装；**早于它存在的板子、以及插件旧于本发布构建所对版本的板子，由一次普通更新修复**，而不是靠谁记住一条命令（§9.1 拥有这条规则与其被打破的四次记录）。

### `install_rkaiq()`（第 163-178 行）

摄像头的 3A 引擎（自动曝光与白平衡），与 GStreamer 相同的理由、相同的条款。装不上的板子仍是一台机器人——只是摄像头跑在 raw ISP 默认上："绿色、噪点多、固定曝光"。

### `install_npu()`（第 194-209 行）

`duck-detect` dlopen 的 NPU 运行时 + 它需要的设备树节点。同样条款：脚本拥有钉住的 runtime tag、overlay 与驱动报告，"一台机器人在本发布里的模型能跑之前，不应该需要谁记住一条命令"。

**它可以要求重启，而本钩子只说明、不执行**——注释加粗：Armbian 在每块 Radxa Zero 3 上出厂禁用 `npu@fde40000`，首个携带此设置的更新启用节点，NPU 在**下次启动**时绑定；在此之前检测器经 `.onnx` 在 CPU 上跑，"这正是 `DetectParams::models()` 是一个列表而非一个选择的原因"。无 NPU 的板子仍走路、仍推流、仍看——只是在 CPU 上。

## 主流程（第 211-231 行）

1. `installed_onnx` 取当前版本；
2. `at_least "$have" "$ONNX_FLOOR"` 满足 → 打印确认（幂等快路径）；
3. 否则打印"低于本发布要求的 floor"→ `install_onnx` → **验证而非假设**：重新读取版本并复核——注释说，一个"报告成功却把符号链接留错"的安装，否则会在 `robotd` 控制线程 panic 时才被发现，"这正是本钩子被写出来要移除的失败模式"——仍低于 floor 则 `die`；
4. 依次 `install_gstreamer`、`install_rkaiq`、`install_npu`（缺脚本则跳过并说明）。

## 在更新时序里的位置

`updaterd` 下载并验证 artifact → 解包 → **本钩子（preinstall）**：ONNX 致命检查在交换前，失败即中止且旧发布无损 → 符号链接交换 → [`postinstall`](postinstall解析.md) → `on_apply` 重启 → 健康门。
