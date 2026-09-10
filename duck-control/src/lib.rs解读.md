# `lib.rs` 解读

## 概述

`lib.rs` 是 `duck-control` crate 的根模块，仅 26 行，承担三项职责：

1. **声明 8 个子模块**：`bus`、`fall`、`imu`、`io`、`model`、`obs`、`policy`、`safety`，构成从硬件总线读取到安全写入电机的完整控制链路。
2. **重新导出关键类型**：将各模块中对外暴露的核心类型提升到 crate 根，使调用方（如 `robotd`）可以通过 `duck_control::RobotIo` 这样的简洁路径引用，而无需深入模块层级。
3. **文档注释声明架构边界**：明确本 crate **故意不是守护进程**——没有 `tokio`、没有 socket、没有 systemd，这些进程级关注点全部归 `robotd` 所有。边界由编译器强制，而非靠开发者自律，从而防止进程逻辑泄漏到驱动电机的代码中。

文件顶部的文档注释引用了 `docs/design/robotd-design.md` §2，说明控制路径（model → bus → io → observations → policy → safety）的设计在该文档中有完整论述。

## 模块声明

```rust
pub mod bus;      // Dynamixel 总线通信
pub mod fall;     // 跌倒预测器（检测跌倒开始）
pub mod imu;      // IMU 数据解码（LSM6DSV16X）
pub mod io;       // RobotIo trait 与 FakeIo 测试替身
pub mod model;    // 机器人模型常量（关节 ID、home pose、电池）
pub mod obs;      // 61 维观测向量与 14 维动作
pub mod policy;   // ONNX 神经网络策略
pub mod safety;   // 安全层（唯一电机写句柄所有者）
```

—— 8 个模块全部 `pub`，因为 `robotd` 需要直接访问各层类型进行组装和测试。

## 重新导出（re-export）

### IMU 与 IO 层

```rust
pub use imu::ImuData;
pub use io::{FakeIo, IoError, JointTargets, RobotIo, Sensors, SlowSensors};
```

—— `ImuData` 是 IMU 解码后的结构化数据；`RobotIo` 是控制循环与物理世界之间的核心 trait；`Sensors` 是一次原子采样（关节 + IMU 在一起）；`FakeIo` 是无硬件测试替身；`IoError` 统一了串口、总线、短读等错误类型。

### 模型常量

```rust
pub use model::{
    BATTERY_EMPTY_V, BATTERY_FULL_V, DEFAULT_POSITION, JOINT_IDS, JOINT_NAMES, NUM_JOINTS,
    battery_percent,
};
```

—— 电池电压上下限、默认 home 位姿、15 个关节的 ID/名称/数量，以及电池百分比换算函数。这些是整个 crate 共享的物理常量。

### 观测与动作

```rust
pub use obs::{ACTION_LEN, Command, OBS_LEN, Observation};
```

—— `OBS_LEN=61`（观测维度）、`ACTION_LEN=14`（动作维度，跳过嘴关节）、`Observation` 是策略网络的输入、`Command` 是高层速度/姿态命令。

## 设计意图与踩坑点

### 为什么 crate 根不做任何逻辑？

整个文件没有一行可执行代码（除了 `pub mod` 和 `pub use`）。这是刻意的：crate 根只做**模块编排和类型聚合**，所有功能下沉到子模块。这样做的好处是：

- 每个子模块可以独立编译、独立测试。
- `robotd` 只需 `use duck_control::*` 或按需导入，无需了解内部模块结构。
- 新增模块时只需加一行 `pub mod`，不会影响现有代码。

### 为什么重新导出而不是让调用方直接引用子模块？

如果 `robotd` 写 `use duck_control::io::RobotIo`，那么当内部模块重构（如将 `io` 拆分为 `io::bus` 和 `io::fake`）时，所有调用方都要改。通过 `pub use` 稳定了**公共 API 表面**，内部结构可以自由演化。

### "边界由编译器强制"是什么意思？

文档注释说 "The boundary is enforced by the compiler rather than by discipline"。具体来说：本 crate 的 `Cargo.toml` 不依赖 `tokio`、`nix`（systemd）或任何网络库，因此编译器根本不允许在驱动电机的代码中调用 `tokio::spawn` 或 `systemd` API。这比代码审查更可靠——**不可能写出跨越边界的代码**。

## 与其他模块的关系

`lib.rs` 是所有模块的父节点，它本身不被任何子模块依赖（子模块通过 `crate::` 路径互相引用）。它的角色是：

- **向上**：对 `robotd` 暴露统一的公共 API。
- **向下**：将 8 个子模块组织为一个内聚的控制核心。

控制数据流方向为：`bus`（读硬件）→ `io`（抽象为 trait）→ `obs`（构建观测）→ `policy`（推理动作）→ `safety`（安全过滤）→ `bus`（写硬件），`imu`、`fall`、`model` 为各层提供支撑。

## 测试要点

本文件无测试代码。其正确性由 Rust 编译器保证：模块声明错误会导致编译失败，重新导出的类型不存在也会导致编译失败。集成测试通过 `cargo test` 在 `FakeIo` 上运行整个控制循环，间接验证了模块编排的正确性。
#（注：内容由AI生成）
