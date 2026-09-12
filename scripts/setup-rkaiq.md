# setup-rkaiq.sh

## 文件位置

`d:\microduck\scripts\setup-rkaiq.sh`

## 核心设计决策

该脚本安装 Rockchip 的 rkaiq 3A 引擎和 IMX219 调优，使相机有白平衡、色彩和降噪而非原始 ISP 默认值。

- **为什么用 vendor 引擎**：此平台没有其他 3A。libcamera 的 rkisp1 IPA 驱动 mainline rkisp1；此板在 vendor 内核上运行 Rockchip 的 vendor rkisp，硬件编码器也在那里。选 mainline 获得开源 3A 会失去 VPU，所以 vendor 引擎（Radxa 池的预构建 deb）是路线。
- **关闭 rkaiq 自动曝光**：引擎的 AE 在流启动时收敛一次然后停止，`mediad` 需要它持续工作。`mediad::exposure` 测量帧并驱动传感器，此脚本只留给引擎连续做的部分（白平衡、色彩矩阵、gamma、降噪）。关闭 AE 是因为它恰在 mediad 循环收敛时写入传感器，两个写入者竞争一个控制。
- **由 `hooks/preinstall` 每次更新运行**：与 `setup-gstreamer.sh` 同契约：永不提示、永不致命、说明发生了什么。
- **幂等**：deb 只在缺失时获取，IQ 补丁应用后是 no-op，shim 因廉价且内核可能变化而每次重建（仅当源码变化时）。

## 常量/参数分析

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `SENSOR` | `imx219` | 默认传感器，Radxa IQ 包只提供 imx219 和 ov5647 |
| `SELF` | `/usr/local/sbin/robot-setup-rkaiq` | 脚本自拷贝位置 |
| `SHIM_SO` | `/usr/local/lib/rkaiq_modinfo_shim.so` | ioctl shim 编译产物 |
| `IQ_DIR` | `/etc/iqfiles` | IQ 调优文件目录 |
| `DROP_IN_DIR` | `/etc/systemd/system/rkaiq_3A.service.d` | systemd drop-in 目录 |
| `PIN` | `/usr/local/bin/rkaiq-pin-sensor-mode` | 传感器模式固定脚本 |
| `SENSOR_W`/`SENSOR_H` | `1920`/`1080` | 传感器模式分辨率，与 `mediad/src/pipeline.rs` 的 `pin_sensor_mode` 同步 |

### deb 来源

- `RKAIQ_DEB`：`camera_engine_rkaiq_rk3568_arm64-fixed.deb`（Radxa 池）
- `IQ_DEB`：`rockchip-iqfiles-rk356x_0.1.16_all.deb`

直接下载 .deb 而非添加 sources.list，不留第三方源。

## 核心逻辑

### 1. 安装引擎和调优

- `fetch_deb`：dpkg -s 检查后下载安装。
- 将 `/usr/share/rockchip-iqfiles-rk356x/<sensor>_*` 复制到 `/etc/iqfiles`。
- 若传感器无调优，警告（引擎用原始 ISP 默认值，画面绿且噪）。

### 2. IQ 枚举名重命名 + 关闭 AE（仅 imx219）

用 Python 二进制模式处理（文件有 CRLF 行尾，文本模式会重写）：
- 重命名两个解析器不认识的枚举值：`AECV2_STRATEGY_MODE_LOWLIGHT_PRIOR` → `AECV2_STRATEGY_MODE_LOWLIGHT`，`CALIB_AWB_HDR_FRAME_CHOOSE_MODE_AUTO` → `CALIB_AWB_HDR_FR_CH_AUTO`。
- 将 `ae_calib CommCtrl.Enable` 设为 0（mediad 拥有曝光）。

### 3. 构建 ioctl shim

- 要求 `rkaiq-modinfo-shim.c` 在脚本旁。
- 需要 gcc（自动安装）。
- 仅当源码变化时重建（与 `/usr/local/lib/rkaiq-modinfo-shim.c` 比较）。
- shim 运行时探测内核结构体大小，不依赖编译时内核。

### 4. 传感器模式固定

`rkaiq_3A_server` 启动时读一次传感器分辨率并编程 ISP 输入。IMX219 启动为 3280x2464，mediad 用 1920x1080 模式（也是获得 30fps 而非 21fps 的模式）。引擎远早于 mediad 启动，不固定的话读取启动模式导致 CIF_ISP_PIC_SIZE_ERROR。

`${PIN}` 脚本在 sysinit 时等待 media 实体出现（最多 10 秒），用 `media-ctl --set-v4l2` 固定为 1920x1080。

### 5. systemd drop-in

`/etc/systemd/system/rkaiq_3A.service.d/robot.conf` 三行：
- `Environment=LD_PRELOAD=${SHIM_SO}`：防止引擎段错误。
- `ExecStartPre=${PIN}`：固定传感器模式。
- `ExecStartPost=-/bin/systemctl --no-block try-restart mediad.service`：引擎启动后弹跳 mediad 流，使引擎看到流启动事件（否则引擎永远等"wait stream start event"）。

### 6. 启动和报告

- ISP 节点（/dev/video8, /dev/video9）存在时重启 rkaiq_3A，检查进程是否存活。
- 不存在时说明正常（需重启到 vendor 内核 + camera overlay）。

## 关键要点总结

1. 必须 root 运行。
2. 关闭 rkaiq AE，mediad 拥有曝光控制。
3. ioctl shim 解决 Radxa engine deb 与 Armbian vendor 内核的结构体大小漂移（5203 vs 5207 字节）。
4. 传感器模式 1920x1080 必须与 mediad 一致，否则相机无帧。
5. ExecStartPost 弹跳 mediad 是关键排序不变量，避免引擎永远等流启动事件。
6. IQ 枚举名重命名仅对 imx219 做过表征，其他传感器不重写。
