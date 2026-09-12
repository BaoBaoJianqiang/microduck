# Cargo.toml 文件解析

## 文件位置

`d:\microduck\odometry\Cargo.toml`

## 包信息

| 字段 | 值 |
|---|---|
| `name` | `odometry` |
| `version` | 继承 workspace |
| `edition` | 继承 workspace |
| `rust-version` | 继承 workspace |
| `license` | 继承 workspace |

包顶部的多行 `#` 注释阐明定位：

> 基于接触的里程计：从自身腿部和 IMU 估计机器人位置。
> 移植自原型 runtime 的 Rhoban 派生估计器（足底角点锚定、IMU 重投影），一处结构性改动：足链来自 `kinematics` crate 的 MJCF 模型而非手抄节段表——几何只有一个真相源。
> 无专用服务：它是 `robotd` 在控制循环内 tick 的纯结构体，因为它的输入正是循环已持有的样本。

## 依赖分析

| 依赖 | 路径 | 用途 |
|---|---|---|
| `duck-ipc-proto` | `../duck-ipc-proto` | 仅取 `JOINT_NAMES`（关节顺序真相源） |
| `kinematics` | `../kinematics` | 提供 `Model`、`Pose`、`Quat`、`SiteId`，MJCF 加载 + 正运动学 |

**无其他依赖**——无 `serde`、无 `thiserror`、无第三方数值库。整个 crate 只用 `kinematics` 提供的 `Quat`/`Pose` 与 `duck-ipc-proto` 的关节名表，保持极简。

## 设计含义

- **无运行时依赖**：不 dlopen 任何 .so，不链接 ONNX/NPU 库，纯 Rust。
- **几何单一真相源**：通过 `kinematics` 的 MJCF 模型加载足部几何，避免与 `duck-control`/`kinematics` 的关节定义漂移。
- **协议类型复用**：`JOINT_NAMES` 来自 `duck-ipc-proto`，保证关节顺序与状态流一致。

## 关键摘要

- 仅 2 个依赖，均为 workspace 内路径依赖。
- 无运行时动态库加载，纯 Rust 纯计算。
- 复用 `kinematics` 的 MJCF 几何 + `duck-ipc-proto` 的关节顺序，杜绝多份几何/关节定义副本。
