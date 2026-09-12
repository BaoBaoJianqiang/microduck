# setup-npu.sh

## 文件位置

`d:\microduck\scripts\setup-npu.sh`

## 核心设计决策

该脚本安装 Rockchip NPU 运行时，使鸭子检测器能在板端 NPU 上运行。

- **两部分来源不同**：RK3566 有小型（0.8 TOPS）INT8 NPU。使用它需要驱动（vendor 内核的一部分，有或没有）和运行时 `librknnrt.so`（vendor blob，不在任何 Debian suite 中）。此脚本安装第二部分并报告第一部分。
- **NPU 节点默认启用**：Armbian 的 `rk3566-radxa-zero3.dtb` 在每个 Radxa Zero 3 上都将 `npu@fde40000` 设为 `status = "disabled"`，所以不启用节点的话安装的运行时永远无法运行任何东西。默认启用，下次启动生效，脚本永不重启。通过从 `/boot/armbianEnv.txt` 的 `overlays=` 中移除 `npu-enable` 撤销。
- **运行时版本必须 ≥ 转换模型的工具包版本**：否则 `rknn_init` 以数字失败且无解释。默认 `RUNTIME="v2.3.2"`，与 `Cargo.toml` 中 `[workspace.metadata.rknpu]` 保持同步（测试断言一致）。
- **每次更新由 `hooks/preinstall` 运行**：与 `setup-gstreamer.sh` 和 `setup-rkaiq.sh` 并列，永不致命，报告写入更新日志。NPU 存在前初始化的板端通过普通更新修复，而非靠人记住命令。

## 常量/参数分析

### 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `RUNTIME` | `v2.3.2` | rknn-toolkit2 标签，默认取 `librknnrt.so` 的版本 |
| `ENABLE_NODE` | `1` | 默认启用 NPU 设备树节点 |
| `SELF` | `/usr/local/sbin/robot-setup-npu` | 脚本自身副本位置 |
| `LIB` | `/usr/lib/librknnrt.so` | 运行时库安装位置 |
| `STAMP` | `/usr/lib/librknnrt.version` | 版本戳记文件 |
| `URL` | `https://raw.githubusercontent.com/airockchip/rknn-toolkit2/${RUNTIME}/rknpu2/runtime/Linux/librknn_api/aarch64/librknnrt.so` | 下载 URL |

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--runtime TAG` | 指定 rknn-toolkit2 标签，默认 v2.3.2 |
| `--enable-node` | 启用 NPU 节点（默认行为，显式声明） |
| `--no-enable-node` | 不修改设备树 |

## 核心逻辑

### 1. 驱动检测（不修改任何东西前读取）

依次尝试：
- `/sys/kernel/debug/rknpu/version`
- `/proc/rknpu/version`
- `dmesg` 中 `RKNPU driver: v...`

### 2. 设备树节点启用（仅当驱动未绑定且 `ENABLE_NODE` 为真）

1. 查找 overlay 源文件 `rk3568-npu-enable.dts`（按优先级：`OVERLAY_DTS` 环境变量、脚本旁、`../deploy/overlays/`、`/usr/local/lib/`）。
2. 确保 `dtc` 可用（尝试 `apt-get install device-tree-compiler`）。
3. 确认 Armbian 布局（`/boot/armbianEnv.txt` 和 `/boot/dtb/rockchip/overlay` 存在）。
4. 编译 overlay：`dtc -I dts -O dtb -o ${OVERLAY_DIR}/rk3568-npu-enable.dtbo`。
5. 将 `npu-enable` 追加到 `overlays=` 行（不替换，保留其他 overlay），备份原文件为 `.before-npu`。
6. 设置 `NEEDS_REBOOT=1`。

### 3. 运行时安装

幂等：仅当 `$LIB` 缺失或 `$STAMP` 版本不符时下载。

下载后验证：
- ELF 魔数检查（`head -c 20 | od` 前 4 字节为 `7f454c46`），防止代理错误页面。
- 大小 > 1MB 检查。

安装后 `ldconfig`，写入版本戳记。

### 4. 自拷贝

若脚本不在 `$SELF` 路径，拷贝自身到 `/usr/local/sbin/robot-setup-npu`。

## 关键要点总结

1. 必须以 root 运行：`sudo sh setup-npu.sh`。
2. 设备树变更只追加 `npu-enable` 到 overlay 列表，不替换（保留 camera/audio codec/uart overlay）。
3. 运行时从 rknn-toolkit2 仓库直接下载单个文件，不在机器人上留下第三方 apt 源。
4. 下载验证防止截断下载导致首次推理时段错误。
5. 脚本永不重启，NPU 节点在下次启动时绑定。
6. 所有失败（找不到 dts、dtc 装不上、非 Armbian 布局）都是警告而非致命，运行时仍会安装。
