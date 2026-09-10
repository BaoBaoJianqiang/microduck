# Cargo.toml 解读 — duck-control

> 文件路径：`duck-control/Cargo.toml`
> crate 名称：`duck-control`
> 描述：`Robot control core: model, bus, sensing`（机器人控制核心：模型、总线、传感）

## 一、[package] 段

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `duck-control` | crate 名称 |
| `version` | `workspace = true` | 版本从工作区继承 |
| `edition` | `workspace = true` | Rust edition 从工作区继承 |
| `license` | `workspace = true` | 许可证从工作区继承 |
| `description` | `Robot control core: model, bus, sensing` | 机器人控制核心：模型、总线、传感 |

**要点**：版本、edition、license 均从工作区根 `Cargo.toml` 继承，不在此文件中硬编码。这是多 crate 工作区的标准做法，确保所有 crate 保持版本一致。

## 二、[dependencies] 段详解

共 6 个依赖，每个都附有详细注释说明版本选择和特性配置的理由。

### 1. duck-ipc-proto（path 依赖）

```toml
duck-ipc-proto = { path = "../duck-ipc-proto" }
```

- **类型**：本地 path 依赖，指向同级目录 `../duck-ipc-proto`
- **用途**：**仅用于 `JOINT_NAMES`**（关节名称表）
- **关键设计决策**：
  - 关节*顺序*是协议的一部分——状态流以裸数组（bare arrays）携带 `joints` 和 `targets`
  - 因此名称表放在每个客户端已经链接的 crate（`duck-ipc-proto`）中，此 crate 重新导出它，而不是手动维护第二份副本
  - **不是 IPC 的入口**：这里没有任何东西与 socket 通信，`robotd` 已经链接了两个 crate，因此没有二进制文件从这个依赖边获得新的依赖

### 2. serde / thiserror / tracing（workspace 依赖）

```toml
serde = { workspace = true }
thiserror.workspace = true
tracing.workspace = true
```

- 三个常用 crate，版本和特性均从工作区继承
- `serde`：序列化/反序列化
- `thiserror`：错误类型派生宏
- `tracing`：结构化日志

### 3. libloading = "0.8"

```toml
libloading = "0.8"
```

- **用途**：在接触 `ort` 之前探测 ONNX Runtime 动态库（dylib）
- **关键细节**：
  - 使用与 `ort` 内部相同的加载器，因此成功的探测意味着 `ort` 自己的加载也会成功
  - 参见 `policy::ensure_runtime` 了解为什么这很重要
  - 这是一个防御性检查：在尝试创建 ONNX 会话之前，先确认运行时库存在且可加载

### 4. ort = "=2.0.0-rc.11"（固定版本 + load-dynamic）

```toml
ort = { version = "=2.0.0-rc.11", default-features = false, features = ["load-dynamic"] }
```

- **版本固定**：`=2.0.0-rc.11`（精确锁定，不允许自动升级）
- **`default-features = false`**：关闭默认特性
- **`features = ["load-dynamic"]`**：启用动态加载特性

**`load-dynamic` 的两个关键后果**（都是想要的）：

1. **aarch64 交叉构建在构建时不需要目标架构的 ONNX Runtime**——`load-dynamic` 在首次使用时通过 `dlopen` 加载 `libonnxruntime`，而不是链接它
2. **没有 libonnxruntime 的笔记本仍然可以编译并运行每个不创建会话的测试**

**版本固定的理由**：固定到 `microduck_runtime` 今天运行的版本，因此板保持它已经工作的 ABI。升级 ONNX Runtime 是一个有意识的行为，而不是 `cargo update` 的副作用。

### 5. rustypot = "1.6.0"（地板版本，不是偏好）

```toml
rustypot = "1.6.0"
```

- **Dynamixel 协议库**，用于与舵机总线通信
- **1.6.0 是地板（floor），不是偏好**：

**关键 bug 修复**：
- 1.6.0 之前的版本对 payload 包含 `FF FF FD` 的协议 2.0 状态包解包（de-stuff）有问题
- 在它之前，这样的读取回来尺寸过大，而下面的固定大小 sync 读取——`bus` 的每一个——解码出垃圾
- 一个 -1 的 present current 就足以触发它

**结论**：这是总线上的正确性边界，不是为了版本升级而升级。低于 1.6.0 的版本会产生错误的电机数据。

### 6. serialport = "4.8"（default-features = false）

```toml
serialport = { version = "4.8", default-features = false }
```

- **串口通信库**，用于打开与舵机总线的串口连接
- **`default-features = false`** 的关键理由：

**去掉 `libudev` 特性**：
- `serialport` 的 `libudev` 特性仅用于枚举端口（`available_ports`）
- 此 crate 从参数文件按路径打开一个端口，**从不枚举**
- `libudev-sys` 需要 pkg-config sysroot，而板的交叉构建没有它，因此保持开启会直接破坏 aarch64 构建

**cargo 特性统一（feature unification）的陷阱**：
- `rustypot` 出于同样的原因禁用了它
- **此 crate 也必须禁用，否则 cargo 的特性统一会把它重新打开**
- 这是 Rust 工作区中一个常见的踩坑点：依赖图中任何一个 crate 启用了某个特性，所有链接它的二进制文件都会获得该特性

## 三、总结

这个 Cargo.toml 是一个精心配置的控制核心 crate，每个依赖选择都有明确的工程理由：

| 依赖 | 版本/配置 | 核心理由 |
|---|---|---|
| `duck-ipc-proto` | path | 共享关节名称表，避免手动同步 |
| `ort` | `=2.0.0-rc.11` + `load-dynamic` | 动态加载避免交叉构建依赖，固定 ABI |
| `libloading` | `0.8` | 探测 ONNX Runtime 存在性 |
| `rustypot` | `>=1.6.0` | 修复协议 2.0 状态包解包 bug（正确性边界） |
| `serialport` | `4.8` + `default-features=false` | 去掉 libudev，避免交叉构建失败 |
| `serde`/`thiserror`/`tracing` | workspace | 标准工具链，工作区统一版本 |

**最值得注意的三个踩坑点**：

1. **cargo 特性统一**：`serialport` 必须在此 crate 中也禁用 `libudev`，否则特性统一会把它重新打开，破坏 aarch64 交叉构建
2. **rustypot 1.6.0 地板**：低于此版本会错误解码包含 `FF FF FD` 的电机状态包，导致垃圾数据
3. **ort load-dynamic**：动态加载使得没有 ONNX Runtime 的开发机器也能编译和运行大部分测试，同时避免交叉构建时需要目标架构的预编译库
#（注：内容由AI生成）
