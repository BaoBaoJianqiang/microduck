# io.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\io.rs`

## 核心设计决策

`io.rs` 定义了**控制循环与物理世界之间的接缝**。`RobotIo::read` 与 `RobotIo::write` 之间的一切都是对纯数据的计算，这正是循环无需真实机器人即可测试的原因。

`Sensors` 将关节与 IMU 打包在一起，因为硬件就是如此：IMU 板位于 Dynamixel 总线上，与舵机在同一事务中获取。将它们拆成两个 trait 会发明总线并不存在的区分，并为遵守该区分而加倍总线流量。

## 类型分析

### `Sensors`
机器人的一次原子采样：
- `positions: [f64; 15]` — 关节角度（弧度）
- `velocities: [f64; 15]` — 关节速度（rad/s）
- `currents_ma: [f64; 15]` — 电流幅值（mA，丢弃符号，方向可由速度推断）
- `imu: ImuData` — IMU 数据

### `JointTargets`
循环输出的命令，仅位置控制（alpha 无速度模式关节）。

### `IoError`
- `Port` — 串口打开失败（含路径与源错误）
- `Bus` — 总线事务失败
- `ShortRead` — `sync_read` 返回块数或长度不符（设备未应答），宁可报错也不掩盖（短读会在半个关节数组中留下陈旧值）
- `Simulated` — 模拟失败

### `SlowSensors`
非每 tick 采样（约每秒一次），电池不会在 20ms 内耗尽、电机也不会在 20ms 内升温，因此无需每 tick 多花一次总线事务：
- `volts: f64` — 舵机平均供电电压（唯一的电池测量，板上无电量计）
- `temps_c: [f64; 15]` — 每关节外壳温度（不取平均，因为单个负重关节过热才是值得关注的情况）

### `ImuStale`
IMU 陈旧读数统计：
- `total: u64` — 自启动累计陈旧读数（永不重置；偶发命中属正常）
- `run: u64` — 当前连续陈旧读数长度（任何新块都重置为 0）

两者一起报告，避免后端只提供其一：没有 total 做参照的 run 会被误读为告警。

## Trait 分析：`RobotIo`

| 方法 | 作用 |
|---|---|
| `read() -> Sensors` | 一次事务同时获取关节与 IMU |
| `write(targets)` | 写入目标位置 |
| `set_gain(kp)` | 设置所有关节位置 P 增益（"变瘫软"的含义由此实现：跌落后降增益而非仅停止命令） |
| `set_torque(on)` | 开关所有关节扭矩。**启动时不调用**——由更新重启的 `robotd` 必须让站立的机器人保持站立，扭矩仅在有人启用策略时打开 |
| `slow_sensors() -> SlowSensors` | 供电电压与温度（独立事务，约 1ms，不在 tick 关键路径上） |
| `imu_stale() -> ImuStale` | 总线自诊断（默认无报告） |
| `imu_ready() -> bool` | 姿态滤波器是否收敛（启动前几秒为 false） |

## `FakeIo`（测试用空机器人）

始终编译，使 `cargo test` 无需硬件、网络、Docker。位置回显最后写入值，行为像完美跟踪的舵机。支持：
- `fail_next_read` — 下一次读失败（一次性）
- `fail_reads` — 接下来 n 次读失败（模拟舵机尚未上电）
- `frozen()` — 冻结位置（模拟瘫软或被手动推动的机器人）
- `track_targets` — 是否跟踪写入目标
- `torque: Option<bool>` — 扭矩状态（`None` 断言"重启未移动机器人"）
- `torque_writes` — 扭矩写入次数
- `last_gain` — 最后增益
- `slow: Option<SlowSensors>` — 慢传感器数据

## 单元测试描述

- `fake_io_tracks_what_was_written`：写入后读回相同值（slice 1 循环的核心契约）。
- `frozen_fake_io_ignores_writes`：frozen 模式下写入不影响读数。
- `simulated_read_failure_clears_itself`：`fail_next_read` 是一次性的，失败后自动恢复。

## 关键摘要

`io.rs` 通过 `RobotIo` trait 抽象了所有硬件交互，`Sensors`/`SlowSensors`/`ImuStale` 定义了数据契约，`FakeIo` 使整个控制循环可在无硬件下测试。扭矩和增益操作通过 trait 暴露而非仅在总线层，因为控制循环现在负责在人类请求时启动机器人，且一切到达电机的指令都经过 `safety::Safety`（它持有唯一句柄）。
