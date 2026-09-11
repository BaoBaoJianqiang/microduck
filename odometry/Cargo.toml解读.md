# `Cargo.toml` 解读 — odometry

> 文件路径：`odometry/Cargo.toml`
> 角色：基于接触的里程计（contact-based odometry），从腿和 IMU 计算机器人位置

---

## 一、包级文档注释（这个 crate 存在的理由）

```toml
# Contact-based odometry: where the robot is, from its own legs and IMU.
#
# A port of the prototype runtime's Rhoban-derived estimator (sole-corner
# anchoring, IMU re-projection), with one structural change: the foot chains
# come from the `kinematics` crate's MJCF model instead of segment tables
# transcribed by hand — the geometry has one source of truth. No dedicated
# service: it is a pure struct robotd ticks inside its control loop, because
# its inputs are exactly the sample the loop already holds.
```

这段注释是整个 crate 的设计宣言，逐句解读：

### 1. "基于接触的里程计：从腿和 IMU 计算机器人位置"

里程计（odometry）回答"机器人现在在哪里"。基于接触的里程计不用视觉（摄像头）或激光雷达，而是用：
- **腿**：足部触地时，足部相对于身体的位置由运动学确定，积分得到位移
- **IMU**：加速度计和陀螺仪，提供姿态和加速度

这是一个腿式机器人特有的里程计方法——脚踩地时，脚相对于世界不动，身体相对于脚移动，由此推算位移。

### 2. "移植自原型运行时的 Rhoban 派生估计器"

> A port of the prototype runtime's Rhoban-derived estimator (sole-corner anchoring, IMU re-projection)

这个估计器来自原型运行时（prototype runtime），源自 Rhoban（一个开源机器人竞赛项目）。核心算法：
- **足趾角锚定（sole-corner anchoring）**：足部触地时，用足趾角（而非整个足底）作为锚定点。这比用整个足底更精确——足趾角是足部与地面的实际接触点。
- **IMU 重投影（IMU re-projection）**：用 IMU 的姿态测量修正足部位置的估计，补偿运动学模型的误差。

### 3. "一个结构性变化：足部链来自 kinematics crate 的 MJCF 模型"

> with one structural change: the foot chains come from the `kinematics` crate's MJCF model instead of segment tables transcribed by hand — the geometry has one source of truth.

这是从原型移植时做的关键改进：

- **原来（原型）**：足部链（从髋到足的运动学链）是手动转录的分段表（segment tables）——在代码中硬编码关节位置、连杆长度、零位偏移。
- **现在**：足部链从 `kinematics` crate 的 MJCF 模型解析，与策略训练用的同一个 `robot_walk.xml`。

**"几何有一个单一事实来源"**：机械修订时只改一个 XML 文件，里程计自动更新，不需要手动同步代码中的分段表。

### 4. "没有专门的服务：纯 struct，robotd 在控制循环中 tick"

> No dedicated service: it is a pure struct robotd ticks inside its control loop, because its inputs are exactly the sample the loop already holds.

里程计不作为独立服务运行（不像 mediad/configd/btd 那样有自己的 systemd unit）。它是一个纯 struct，robotd 在控制循环中每个 tick 调用它。

**为什么不需要独立服务**：
- 里程计的输入（关节角度、IMU 数据）正是控制循环已经持有的样本
- 独立服务需要 IPC 传输数据，引入延迟和复杂度
- 纯 struct 直接在控制循环中运行，零拷贝、零延迟

---

## 二、[package] 元数据

```toml
[package]
name = "odometry"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `odometry` | crate 名 |
| `version` | workspace | 从工作区根继承 |
| `edition` | workspace | 从工作区根继承 |
| `rust-version` | workspace | 从工作区根继承 |
| `license` | workspace | 从工作区根继承 |

---

## 三、[dependencies] 运行时依赖

### 3.1 `duck-ipc-proto = { path = "../duck-ipc-proto" }`

```toml
duck-ipc-proto = { path = "../duck-ipc-proto" }
```

**IPC 协议类型定义。**

- 路径依赖（`path = "../duck-ipc-proto"`），同工作区内的相邻 crate
- 提供 IPC 消息的类型定义（结构体、枚举、序列化/反序列化）
- 里程计可能需要输出位置估计给其他服务（如 robotd 的状态报告、WebRTC 的遥测数据）
- 只依赖 proto（类型定义），不依赖具体传输层——保持 crate 可移植

### 3.2 `kinematics = { path = "../kinematics" }`

```toml
kinematics = { path = "../kinematics" }
```

**运动学模型。**

- 路径依赖，同工作区相邻 crate
- 提供 `Model::alpha()`、`site_pose()` 等前向运动学查询
- 里程计用它计算足部（`left_foot`/`right_foot` site）在身体坐标系中的位置
- 这是"几何单一事实来源"的实现——足部链来自 MJCF 解析的模型

---

## 四、依赖关系图

```
odometry (库)
├── duck-ipc-proto (path) — IPC 协议类型
└── kinematics (path)     — MJCF 运动学模型
```

**没有 dev-dependencies**：这个 crate 没有测试（或测试在外部 crate 中）。

---

## 五、设计要点总结

### 5.1 极简依赖

只有两个路径依赖，没有外部 crate：
- 没有 `nalgebra`/`glam`：运动学计算委托给 `kinematics` crate
- 没有 `serde`/`serde_json`：序列化委托给 `duck-ipc-proto`
- 没有 `thiserror`：错误类型可能在 `duck-ipc-proto` 中定义，或里程计不需要复杂错误处理
- 没有 `log`/`tracing`：纯计算，错误通过返回值传递

### 5.2 纯 struct，无独立服务

里程计不作为独立进程运行。它是 robotd 控制循环中的一个 struct，每个 tick 被调用。

**为什么这是正确的架构**：
- 输入（关节角度、IMU）正是控制循环已经持有的样本
- 独立进程需要 IPC 传输 → 延迟 + 序列化开销
- 纯 struct 直接在循环中运行 → 零拷贝、零延迟
- 里程计计算量小（足部位置查询），不需要独立线程

### 5.3 几何单一事实来源

足部链从 `kinematics` crate 的 MJCF 模型解析，而非手动转录的分段表。

**消除的风险**：
- 机械修订时忘记更新代码中的分段表 → 里程计漂移
- 分段表与 MJCF 不一致 → 里程计和仿真不匹配

### 5.4 从原型到生产的移植

这个 crate 是从原型运行时移植的，保留了核心算法（足趾角锚定、IMU 重投影），但：
- 足部链来源：手动分段表 → MJCF 模型
- 架构：原型中的独立模块 → robotd 控制循环中的纯 struct
- 几何事实来源：多处维护 → 单一 XML 文件

---

## 六、与其他 crate 的关系

- **`robotd`**：宿主进程。在控制循环中 tick 里程计 struct，传入关节角度和 IMU 样本，获取位置估计
- **`kinematics`**：提供足部运动学查询（`left_foot`/`right_foot` site 的世界/身体坐标位置）
- **`duck-ipc-proto`**：IPC 协议类型，里程计输出的位置估计可能通过 IPC 发给其他服务
- **`duck-control`**：控制循环的宿主，也使用 `kinematics` crate
- **原型运行时**：Rhoban 派生估计器的原始来源（已被吸收并改进）
#（注：内容由AI生成）
