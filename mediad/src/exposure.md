# exposure.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\exposure.rs`

## 核心设计决策

软件自动曝光。**根本原因是 Rockchip 的 3A 引擎只在流启动时收敛一次曝光，然后就停了**——实测在健康启动上 `mediad` 写入 `exposure=600/analogue_gain=1024`，传感器停在 `1589/1536`（引擎的答案），但之后手写 `exposure=300/analogue_gain=256`（更暗）能保持 25 秒无修正。一次收敛不是自动曝光，从窗边走进走廊的机器人会一直保持窗边的曝光。

而当引擎错过流启动事件（它等一个事件，已触发的注意不到），连那一次都不发生——这就是"3A 罢工，重启有时能好"的形态。修复顺序只能让那一次回来，不会变成循环，所以两个半都需要本模块。

与原型（`microduck_runtime` 的 `camera.rs`）两处不同：

1. **亮度来自 tee 的原始分支**，已为检测器和 `get_frame` 存在；帧是 UYVY，亮度在每隔一个字节，均值只需子采样遍历，无需解码。
2. **不再为测量而第二次打开摄像头**。原型早期用并行 `v4l2-ctl` 采 ISP self 路径，会在驱动层与采集竞争，间歇杀死管线。

## 常量

| 常量 | 值 | 含义 |
|---|---|---|
| `INTERVAL` | 500 ms | 测光频率，每秒两次 |
| `REASSERT_TICKS` | 20（10 秒） | 多久重写一次当前值，防止外部写入覆盖 |
| `REPORT_TICKS` | 20 | 多久打一次 info 心跳 |
| `TARGET_Y` | 90 | 目标均值亮度（8-bit，ISP gamma 后） |
| `DEADBAND` | 0.12 | 相对误差死区，防抖动 |
| `SOFT_LINES` | 600（≈11.4 ms） | 快门软上限，先花这里 |
| `HARD_LINES` | 1200（≈22.9 ms） | 快门硬上限；超过驱动会拉长帧时间导致降帧 |
| `MAX_ANALOGUE` | 11x | 模拟增益上限 |
| `MAX_DIGITAL` | 16x | ISP 数字增益上限 |

亮度按噪声顺序消耗：快门到软帽 → 模拟增益 → 快门到硬帽 → 数字增益（最噪）。

## 类型与函数

### `Controls`
`exposure`（行数）、`analogue_gain`（256=1x）、`gain`（ISP 数字增益，256=1x）。

### `Ae`
状态机：`exposure/analogue/digital` 三个 f64 预算 + `written: Option<Controls>`。

- **`starting_at`**：从 `mediad` 已写入的值起步，第一步相对画面而非凭空。
- **`step(mean_luma) -> Option<Controls>`**：死区内或与上次写入相同则 `None`。后者不是优化——暗到传感器极限的房间，比值永久在死区外（目标不可达），不跳过会每半秒写一次同样的三个数（每次 43 ms CPU）。步长对三控制的乘积做 `ratio^0.6` 阻尼（因 luma 经 gamma 压缩，全修正会过冲振荡）。
- **`current()`**：重断言用，返回当前值并标记为已写。

### `mean_luma(frame) -> Option<f64>`
UYVY 每第 8 个像素取亮度字节（步长 16），约 11k 样本足够三位精度；非 `CAPTURE_FORMAT` 返回 `None` 而非猜错。

### `spawn(device, frames, exposure_lines, analogue_gain_reg) -> Stop`
起 `auto-exposure` 线程。`device` 是采集节点（rkisp 经它代理传感器控制）。写传感器用 `v4l2-ctl` 子进程而非 ioctl（`VIDIOC_S_EXT_CTRLS` 三层嵌套结构手写偏移易错，写成功却什么都不改）。**分两次调用**：`exposure,analogue_gain` 属传感器，`gain` 属 ISP；一次调用会让拼写数字增益不同的板子连快门一起丢。

首次成功写且与启动 pin 不同时回读验证——否则写成功但传感器没接住会无声失败。

## 单元测试

| 测试 | 意图 |
|---|---|
| `luma_comes_from_the_luma_bytes` | UYVY 中从正确字节取亮度 |
| `a_format_we_cannot_read_is_not_guessed_at` | 非 UYVY 返回 None |
| `a_picture_at_the_setpoint_is_left_alone` | 目标点及死区内不写 |
| `a_dark_picture_gets_more_light_and_a_bright_one_less` | 方向正确 |
| `light_is_spent_on_the_shutter_before_the_gains` | 噪声顺序：先快门后增益 |
| `a_room_darker_than_the_sensor_can_reach_stops_being_written_to` | 到顶后停止重复写（实机 bug），亮回来能恢复 |
| `the_re_assert_writes_the_same_values_and_restarts_the_heartbeat` | 重断言写当前值并重启心跳 |
| `the_shutter_never_asks_for_a_longer_frame_than_the_sensor_has` | 不超硬帽，防降帧 |
| `the_sensor_is_only_believed_about_a_value_it_was_not_already_at` | 回读验证只在请求值≠启动 pin 时做一次 |
| `the_loop_converges_rather_than_ringing` | gamma 模型下收敛而非振荡 |

## 关键摘要

- 因 3A 引擎只收敛一次而做的软件曝光环。
- 三控制按噪声顺序分配亮度预算，阻尼步长防振荡。
- 到顶后停止重复写（省 CPU），每 10 秒重断言防外部覆盖。
- 用 `v4l2-ctl` 而非 ioctl，分两次写传感器/ISP 控制。
