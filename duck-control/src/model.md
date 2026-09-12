# model.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\model.rs`

## 核心设计决策

`model.rs` 将机器人定义为**数据**。仅存在一个变体 **alpha**（因为这是唯一真实存在的机器人），所有已发布的策略都是 `alpha_*`；v1/v1.5/v1.6 已是历史。所有数值直接取自 `microduck_runtime` 的 `motor.rs`——这些值是在真实硬件上测量得到的，而非从数据手册推导，因为"看起来对但实际走起来错"正是这类改动的典型陷阱。

## 常量分析

| 常量 | 值 | 含义 |
|---|---|---|
| `NUM_JOINTS` | 15 | 左腿(5) · 颈/头/嘴(5) · 右腿(5) |
| `JOINT_IDS` | `[20..24, 30..34, 10..14]` | Dynamixel ID，索引顺序与 `JOINT_NAMES` 一致 |
| `MOUTH_INDEX` | 9 | 嘴关节在所有 alpha 策略中被跳过（14 动作） |
| `DEFAULT_POSITION` | 15 个 f64 | 初始位姿；躯干比 v1.5 前移约 5mm 使重心落在踝关节轴上 |
| `MOUTH_CLOSED` / `MOUTH_OPEN` | -5° / +30° | 嘴部行程，复用 v1.6 范围 |
| `IMU_DXL_ID` | 200 | `imu_to_dxl` v2 板的 Dynamixel ID，与舵机共享总线 |
| `BAUD_RATE` | 1,000,000 | 总线波特率 |
| `BATTERY_FULL_V` / `BATTERY_EMPTY_V` | 8.2 / 6.6 | 负载下的可用电压范围（NP-F550 2S 锂电） |

`EXPECTED_REGISTERS`：启动时强制校正的 EEPROM 寄存器：
- `return_delay_time = 0`（最关键：XL330 出厂 250，每设备 500µs 周转，16 设备即 8ms/帧，占 20ms 预算的 40%）
- `baud_rate = 3`（1 Mbps）
- `pwm_slope = 255`
- `shutdown = 52`（过载、过热、输入电压故障的错误掩码）

## 函数分析

### `mouth_target(open: f64) -> f64`
根据开合比例（0=闭，1=全开）计算嘴部关节角度。对非有限值或越界值做钳制，避免向舵机发送超行程目标。

### `joint_index(name: &str) -> Option<usize>`
按名称线性查找关节索引（15 项，启动时和测试中使用）。

### `battery_percent(volts: f64) -> f64`
将总线电压映射为 0–100 的电量百分比（线性）。非有限或非正读数返回 0（表示"总线无应答"，调用方应报告为未知）。该映射集中放在此处而非 `robotd`，避免原型中"CLI 和 App 各自推导导致两个屏幕对同一电池显示不一致"的问题。

## 单元测试描述

- `tables_agree_on_length`：三张表（`JOINT_IDS`、`JOINT_NAMES`、`DEFAULT_POSITION`）长度一致，否则查找会静默读错关节。
- `ids_are_unique`：Dynamixel ID 不可重复，否则 `sync_read` 返回的块无法匹配回关节，`sync_write` 会同时驱动两个关节。
- `imu_id_does_not_collide_with_a_joint`：IMU 板 ID 不与任何关节 ID 冲突。
- `mouth_index_names_the_mouth`：`MOUTH_INDEX` 必须指向 "mouth"，否则跳过槽位时会整体错位。
- `battery_percent_spans_the_usable_range`：满电=100%、空电=0%、中间 7.4V≈50%。
- `battery_percent_clamps_rather_than_extrapolating`：超出范围（9.5V、5.0V）钳制在 0–100。
- `battery_percent_treats_no_reading_as_zero`：0.0、NaN、负值均返回 0。
- `mouth_target_spans_the_prototype_range`：开合范围正确，越界和 NaN 钳制。
- `home_pose_legs_are_mirrored`：双腿 roll/pitch/knee/ankle 镜像对称（和为 0），符号笔误肉眼不可见但会让机器人站歪。

## 关键摘要

`model.rs` 是机器人的"单一数据真相源"：关节表、位姿、电池、嘴部行程、EEPROM 期望寄存器全部集中定义并通过编译期断言保证一致性。所有数值来自硬件实测而非推导，是整个控制栈的基础。
