# setup-board.sh

## 文件位置

`d:\microduck\scripts\setup-board.sh`

## 核心设计决策

该脚本将刚刷好的板端准备好运行机器人，然后报告是否就绪。

- **为什么从 install.sh 拆出**：此脚本做 OS 级初始化（设备树 overlay、ONNX Runtime），变化少、需重启、属于板端。`install.sh` 安装签名守护进程发布，每次更新发生、属于软件。混淆两者意味着每次更新都重新争论启动配置。
- **幂等、安全可重跑、永不自行重启**：若改变需重启的东西，说明后停止，重跑继续。
- **首次运行拷贝自身到 `/usr/local/sbin/robot-setup-board`**：/tmp 不随重启存活，一个工作是"改启动配置、重启、确认"的脚本若在重启中删除自己是糟糕体验。

## 常量/参数分析

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `ONNX_VERSION` | `1.28.0` | ONNX Runtime 版本，必须满足 ort 最低要求（>= 1.23.x），与 Cargo.toml 中 ort 同步 |
| `MOTOR_PORT` | `/dev/ttyS2` | Dynamixel 总线串口 |
| `ENV_TXT` | `/boot/armbianEnv.txt` | Armbian 启动配置 |
| `REQUIRED_OVERLAY` | `uart2-m0` | robotd 唯一需要的 overlay（IMU 走 Dynamixel 总线） |
| `WEIRD_BLE_MARKER` | `/var/lib/robot/weird-ble` | robotctl 读取的标记，决定配对时是否暂停 btd |
| `SELF` | `/usr/local/sbin/robot-setup-board` | 脚本自拷贝位置 |

### 蓝牙标志

| 标志 | 说明 |
|---|---|
| `DUCK_WEIRD_BLE` | 设 `Privacy = device`（部分 Zero 3W 只能在此设置下配对） |
| `DUCK_PAUSE_BTD` | 配对时暂停 btd（aic8800 无线电上 btd 广播会阻止新配对）。`--weird-ble` 隐含此标志 |

## 核心函数

### configure_overlay

修复两个静默失败陷阱：
1. Armbian 带 `overlay_prefix=rk35xx`，但 RK3566 的 overlay 命名为 `rk3568-*.dtbo`，前缀错误导致找不到 overlay。
2. `armbian-config` 的 overlay 编辑器因同样原因崩溃。

直接修改 `/boot/armbianEnv.txt`：设 `overlay_prefix=rk3568`，追加 `overlays=uart2-m0`。

### install_onnxruntime

版本感知（非仅存在感知）：通过 `readlink -f libonnxruntime.so` 解析版本。从 microsoft/onnxruntime 下载 aarch64 tarball，安装到 `/usr/local/lib`，`ldconfig` 刷新。

### free_motor_port

UART2 是 RK3566 调试控制台，Armbian 默认在其上运行 `serial-getty@ttyS2`。getty 读取端口会消费 Dynamixel 回复。两部分处理：
1. mask `serial-getty@ttyS2.service`（非仅 disable，`getty.target` 会拉回）。
2. 将 `console=both`/`console=serial` 改为 `console=display`（内核 printk 也会在电机总线上破坏回复）。

### configure_bluetooth

两个独立标志修复不同故障：
- `btd` 广播破坏新配对 → 暂停 btd 修复。
- `Privacy = device` 在不需要的板上破坏重连（`PIN or Key Missing` 抖动）。

无标志时不动任何东西。写 `WEIRD_BLE_MARKER` 供 robotctl 读取。

### configure_audio（5 层）

1. 安装 alsa-utils、DKMS 工具链、dtc、i2c-tools（创建 i2c 组）。
2. 安装 Armbian vendor 内核（codec 的 I²S 时钟树只在那里），设为活动内核。
3. 编译并安装 device-tree overlay（i2c3-pihat、aic3104-i2c3）。
4. 通过 DKMS 构建 aic3x codec 驱动（vendor 内核不构建 SND_SOC_AIC3X）。
5. 安装 aic3104-init.service 设置混音器电平（robotd 之前运行）。

所有步骤软失败（无音频不影响行走）。

### configure_tof

udev 规则为 i2c3 总线创建稳定符号链接 `/dev/i2c-pihat`（按设备树地址 `fe5c0000.i2c` 或名称 `i2c-gpio-pihat` 匹配）。

### configure_camera

将 Armbian 的 `radxa-zero3-rpi-camera-v2.dtbo` 镜像为 `rk3568-radxa-zero3-rpi-camera-v2.dtbo`（因为 `overlay_prefix=rk3568` 会解析为带前缀的名称）。复制而非符号链接（Armbian 包更新会替换原文件）。

### report

报告板端状态：电机总线、btd 配对暂停、蓝牙 privacy、手柄、电机总线占用者、内核控制台、ONNX Runtime、失败 unit、wifi、networkd wait-online、时钟。

## 关键要点总结

1. 必须 root 且 aarch64 运行，需要 curl、tar、find、install。
2. ONNX 版本必须 >= ort 要求，否则 robotd 控制线程 panic。
3. 电机 UART 不能同时是控制台，需 mask getty 并设 console=display。
4. 蓝牙两个标志独立：暂停 btd 修复新配对，Privacy=device 修复无法配对但会破坏重连。
5. 音频 5 层全部软失败，vendor 内核是关键依赖。
6. 相机 overlay 需带 rk3568- 前缀镜像。
7. 脚本改变需重启的东西后停止并提示，幂等重跑继续。
8. `main "$@"` 在最后一行，防止 `curl | sh` 截断。
