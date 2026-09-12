# lib.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\lib.rs`

## 核心设计决策

`lib.rs` 是 `duck-control` crate 的入口模块，其核心设计是**将机器人控制核心刻意设计为非守护进程**：这里没有 tokio、没有 socket、没有 systemd——这些进程级的关注点全部由 `robotd` 拥有。边界由编译器强制，而非靠开发者自律，从而阻止进程层面的事务泄漏到驱动电机的控制代码中。

控制路径（model、bus、io、observations、policy、safety）的整体设计见 `docs/design/robotd-design.md` 第 2 节。

## 模块导出

### 子模块声明
- `bus` — Dynamixel 总线交互
- `fall` — 跌倒预测（早期检测正在发生的跌倒）
- `imu` — IMU 板数据解码
- `io` — 控制循环与物理世界的接缝（`RobotIo` trait）
- `model` — 机器人的数据定义（关节、电池、位姿等）
- `obs` — 策略观测向量
- `policy` — ONNX 策略加载与推理
- `safety` — 安全权威（唯一持有写句柄）

### 公开再导出（`pub use`）
- `imu::ImuData`
- `io::{FakeIo, IoError, JointTargets, RobotIo, Sensors, SlowSensors}`
- `model::{BATTERY_EMPTY_V, BATTERY_FULL_V, DEFAULT_POSITION, JOINT_IDS, JOINT_NAMES, NUM_JOINTS, battery_percent}`
- `obs::{ACTION_LEN, Command, OBS_LEN, Observation}`

## 关键摘要

`lib.rs` 本身不包含可执行逻辑，仅作为模块组织与 API 门面。其最重要的设计声明是：控制核心是**纯计算层**，一切 I/O、异步、进程生命周期都被隔离在 `robotd` 中，确保驱动电机的代码可测试、可推理且不被运行时设施污染。
