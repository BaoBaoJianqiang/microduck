# bus.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\bus.rs`

## 核心设计决策

`bus.rs` 实现 Dynamixel 总线交互：每 tick 一次合并的 `sync_read`（覆盖 IMU 板和全部 15 个舵机），加一次 `sync_write` 目标位置。IMU 列在首位以便在舵机突发之前应答。

电池和温度是唯一不适合该形状的部分：它们位于 tick 读取块之外的寄存器，因此 `slow_sensors` 是独立事务，约每秒调用一次而非每 tick。

代码基于 `rustypot`，但数值（转换因子和启动时强制校正的 EEPROM 寄存器）来自 `microduck_runtime`（实测值）。

## 常量分析

| 常量 | 值 | 含义 |
|---|---|---|
| `READ_ADDR` / `READ_LEN` | 124 / 12 | tick 读取的连续块起始地址与长度：`present_pwm`、`present_current`、`present_velocity`、`present_position` |
| `RAD_PER_SEC_PER_COUNT` | 0.229 × (2π/60) | 速度转换：每计数 0.229 转/分 |
| `SLOW_READ_ADDR` / `SLOW_READ_LEN` | 144 / 3 | 慢读块：`present_input_voltage`(u16) + `present_temperature`(u8) |
| `VOLTS_PER_COUNT` | 0.1 | 电压每计数 0.1V |
| `READ_TIMEOUT` | 30ms | 读取超时，缺失设备只造成有限卡顿 |
| `STALE_RUN_WARN` | 25 | 连续陈旧读数达到此值时日志告警（50Hz 下半秒，与 `SflpDecoder::ready` 同跨度） |

## `StaleImuTracker`

检测 IMU 板"应答但未刷新"的故障（记住上一个块）。从读取路径拆出以便无串口测试。
- `last: Option<[u8; 12]>` — 首个块之前为 `None`（固定初始值不行，因为全零块正是 SFLP 表为空时板发送的内容）
- `stale: ImuStale` — `total`（累计）与 `run`（当前连续）
- `observe(block) -> u64` — 记录一个块并返回其所属连续陈旧长度（0=新鲜）

## `DynamixelIo`

`RobotIo` 的硬件实现。`ids` 顺序：IMU 在前，然后 `JOINT_IDS` 顺序（即块返回顺序）。

### 关键方法

- `open(port)` — 打开串口（`BAUD_RATE`、`READ_TIMEOUT`），创建 `Xl330Controller`（协议 v2），组装 ID 列表。
- `check_registers() -> Result<usize>` — 断言并校正 `EXPECTED_REGISTERS` 中的寄存器，返回修复数量。空 `Vec` 表示舵机未应答，不可当作"寄存器正常"。
- `present_positions() -> [f64; 15]` — 仅读取当前位置（启动时用于接纳机器人已在的姿态）。
- `set_torque(on)` — 每关节一次事务（非每 tick），仅在有人启用策略时调用。
- `interpolate_to(target, duration, step)` — 从当前位置线性插值到目标（仅 `init` 调用，控制循环绝不自行移动机器人）。阻塞执行，期间不应有其他总线通信。

### `RobotIo` 实现

- `read()`：对 16 个 ID 做 `sync_read_raw_data(READ_ADDR, READ_LEN)`。
  - 块 0 是 IMU：解码、陈旧检测（达 `STALE_RUN_WARN` 或其后每 500 次告警一次，避免刷屏）、`SflpDecoder::decode`。
  - 块 1.. 是舵机：`[0..2]` pwm 未用、`[2..4]` current（取绝对值）、`[4..8]` velocity（i32 × 转换因子）、`[8..12]` position（`2π×count/4096 − π`）。
- `write()`：`sync_write_goal_position`。
- `set_gain(kp)`：同时写 P=kp、I=0、D=0（原型默认；工厂 D 增益非零会让机器人在相同 kP 下明显变软，故固定而非暴露）。
- `slow_sensors()`：对 `JOINT_IDS` 读 144–146。电压取平均（15 舵机共用一组电池），温度不平均（单关节过热才是关注点）。零电压过滤（无意义值不参与平均）。
- `imu_stale()` / `imu_ready()` — 透传。

## 单元测试描述

- `read_block_is_long_enough_for_every_field`：`READ_LEN` 覆盖所有字段且等于 `IMU_BLOCK_LEN`。
- `position_conversion_round_trips_through_rustypot`：位置转换与 rustypot 的 `AnglePosition` 往返一致。
- `velocity_scale_matches_the_datasheet_figure`：速度比例 0.229 rpm/count 正确。
- `the_first_block_is_never_stale`：首块无前驱，不计陈旧。
- `fresh_blocks_count_for_nothing`：新鲜块不增加计数，且清除 run。
- `a_hiccup_is_remembered_in_the_total_but_not_the_run`：偶发重复计入 total 但 run 归零。
- `a_dead_board_runs_past_the_warning_threshold`：持续重复使 run 达到告警阈值。
- `separate_episodes_add_up`：多段陈旧在 total 中累加。

## 关键摘要

`bus.rs` 是 `RobotIo` 的真实硬件实现，基于 rustypot。每 tick 一次合并 sync_read（IMU+15 舵机）+ 一次 sync_write；电池/温度走独立慢读事务。关键设计：IMU 陈旧检测独立可测、EEPROM 寄存器启动校正（尤其 `return_delay_time`）、增益同时写 I=D=0、电压平均温度不平均。所有数值与 `microduck_runtime` 对齐。
