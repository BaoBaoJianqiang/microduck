# `Cargo.toml` 解读 — padd

> 文件路径：`padd/Cargo.toml`
> 角色：Gamepad → robot intents（手柄意图客户端）

---

## 一、为什么是独立 crate

```toml
# Its own crate so a gamepad stack stays out of `robotctl`, which has to work on a broken
# robot and is part of the recovery path. Nothing here is on that path.
```

padd 作为独立 crate 的核心理由：**让手柄栈不进 `robotctl`**。

`robotctl` 必须在坏机器人上工作，且是恢复路径的一部分。如果手柄依赖（gilrs、SDL 数据库、libudev）进入 `robotctl`：
- `robotctl` 的依赖树变大，在最小化恢复环境中可能无法安装
- gilrs 的 C 依赖（libudev-sys）可能在恢复环境中缺失
- padd 崩溃不会影响恢复工具

**这里没有东西在恢复路径上**——padd 只是一个驾驶工具，坏了不影响机器人恢复。

---

## 二、gilrs 的代价：libudev-sys

```toml
# gilrs depends on libudev-sys unconditionally on Linux — no feature disables it — so both
# CI and the board cross-build install libudev. That is a real cost, paid to keep gilrs's
# SDL controller database: without it we would hand-map each pad's raw evdev codes, and the
# same Xbox controller reports different codes over USB and Bluetooth.
```

### gilrs 无条件依赖 libudev-sys

- 没有 feature 可以禁用它
- CI 和板子交叉编译都必须安装 libudev
- 这是**真实代价**

### 为什么值得这个代价

换来 gilrs 的 **SDL 控制器数据库**：
- SDL 维护了一个巨大的手柄映射数据库
- 没有它，padd 必须手动映射每个手柄的原始 evdev 代码
- **同一个 Xbox 控制器在 USB 和蓝牙上报不同的按钮/轴代码**——SDL 数据库处理了这种差异
- 手动映射意味着每个新手柄型号都要写映射表

### 对未来的教训

```toml
# The same expense returns for the next C dependency that has to reach the board, which is
# an argument for preferring pure-Rust crates anywhere else on that path. `evdev` below is one:
# it reads the same device node gilrs reads, for `src/tap.rs`, and brings no C with it.
```

下一个必须到达板子的 C 依赖会带来相同代价。这是偏好纯 Rust crate 的理由。`evdev` 就是一个纯 Rust crate——它读 gilrs 读的同一个设备节点（为 `src/tap.rs`），不带 C。

---

## 三、[package] 元数据

```toml
[package]
name = "padd"
version.workspace = true
edition.workspace = true
license.workspace = true
description = "Gamepad → robot intents"
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `padd` | crate 名 |
| `version` | workspace | 从工作区根继承 |
| `edition` | workspace | 从工作区根继承 |
| `license` | workspace | 从工作区根继承 |
| `description` | `"Gamepad → robot intents"` | crate 描述 |

注意：没有 `rust-version.workspace = true`——padd 不声明最低 Rust 版本要求。

---

## 四、[dependencies] 运行时依赖

### 4.1 `duck-ipc-proto = { path = "../duck-ipc-proto" }`

IPC 协议类型定义。padd 通过 robotd 的 unix socket 发送意图，需要 `Call`、`Request`、`Response`、各种 `Params` 类型。

### 4.2 `clap = { workspace = true, features = ["derive"] }`

命令行参数解析。`derive` feature 用于 `#[derive(Parser)]`——Args 结构体用 derive 宏定义命令行参数。

### 4.3 `gilrs = "0.11"`

手柄库。提供：
- `Gilrs::new()` 初始化手柄子系统
- `gamepads()` 枚举连接的手柄
- `next_event()` 事件循环
- `Axis`/`Button` 枚举
- `LinuxGamepadExt::devpath()` 获取 evdev 节点路径（Linux-only）

**这是唯一带 C 依赖的 crate**（通过 libudev-sys）。

### 4.4 `serde.workspace = true` / `serde_json.workspace = true`

```toml
# Only to serialise the tap's own lines; the wire types themselves live in duck-ipc-proto.
```

仅用于序列化 tap 自己的行；线上类型本身住在 `duck-ipc-proto` 中。

### 4.5 `tracing.workspace = true` / `tracing-subscriber = { workspace = true, features = ["env-filter"] }`

结构化日志。`env-filter` feature 支持 `RUST_LOG` 环境变量控制日志级别。

---

## 五、[target.'cfg(target_os = "linux")'.dependencies] 平台特定依赖

```toml
# The raw input tap (`src/tap.rs`) is evdev, and evdev is Linux. Nothing else in `padd` is
# platform-specific, so a Mac still builds and still drives a pad — it serves no tap.
```

tap（`src/tap.rs`）是 evdev，evdev 是 Linux。padd 中没有其他东西是平台特定的，所以 Mac 仍能编译并驱动手柄——只是不服务 tap。

### 5.1 `evdev = "0.13"`

```toml
# The raw event stream, unfiltered: `raw_stream::RawDevice` is the one that does *not* resync on
# SYN_DROPPED, which is the event a link investigation most needs to see.
```

原始事件流，不过滤。`raw_stream::RawDevice` 是那个**不在 SYN_DROPPED 时 resync** 的变体——链路调查最需要看的事件。

**为什么不用 gilrs 读 tap**：gilrs 内部用 evdev 但会过滤掉 SYN_DROPPED/SYN_REPORT/MSC_SCAN，而这些正是调试不可靠链路需要看的。

### 5.2 `libc = "0.2"`

```toml
# `getgrnam`/`chown`, to hand the tap's socket to the `robot` group. Bindings only, no C to build.
```

`getgrnam`/`chown` 系统调用绑定，用于将 tap 的 socket 交给 `robot` 组。**只是绑定，不需要构建 C**。

---

## 六、依赖关系图

```
padd (二进制)
├── duck-ipc-proto (path)     — IPC 协议类型
├── clap (workspace, derive)   — 命令行参数
├── gilrs 0.11                — 手柄库（唯一带 C 依赖的）
│   └── libudev-sys (C)       — 无条件依赖
├── serde (workspace)         — tap 行序列化
├── serde_json (workspace)    — JSON
├── tracing (workspace)       — 日志
└── tracing-subscriber (workspace, env-filter)

Linux only:
├── evdev 0.13                — 原始 evdev 流（纯 Rust）
└── libc 0.2                  — getgrnam/chown 绑定
```

---

## 七、设计要点总结

### 7.1 手柄栈隔离在 padd crate 中

gilrs（及其 C 依赖 libudev-sys）被限制在 padd crate 中，不进入：
- `robotctl`（恢复路径，必须在坏机器人上工作）
- `robotd`（控制循环，不能引入不必要的依赖）

### 7.2 gilrs 的代价是有意识的权衡

- **代价**：CI 和板子交叉编译必须安装 libudev
- **收益**：SDL 控制器数据库——自动处理不同手柄型号和不同连接方式（USB/蓝牙）的代码差异
- **决策**：值得，因为手动映射每个手柄的 evdev 代码更不可维护

### 7.3 纯 Rust 优先

`evdev` 是纯 Rust crate，读同一个设备节点但不带 C。注释明确指出：未来优先选择纯 Rust crate，避免重复 gilrs 的 C 依赖代价。

### 7.4 平台条件编译

- `evdev` 和 `libc` 只在 Linux 上
- `tap` 模块用 `#[cfg(target_os = "linux")]` 门控
- 非 Linux 上有一个空的 `tap` 模块（`serve` 返回错误，`watch`/`idle` 为空）
- Mac 上 padd 仍能编译并驱动手柄，只是不服务原始输入 tap

### 7.5 没有 dev-dependencies

padd 是一个二进制 crate，没有测试依赖。tap.rs 中的测试使用标准库和已有的运行时依赖。

---

## 八、与其他 crate 的关系

- **`duck-ipc-proto`**：共享的 IPC 协议类型，padd 是其客户端之一
- **`robotd`**：padd 通过 unix socket 连接的服务端
- **`robotctl`**：明确不包含手柄依赖——这是 padd 独立成 crate 的理由
- **`evdev`**：Linux 特定的原始输入读取，用于 tap.rs
- **`gilrs`**：手柄抽象层，跨平台
#（注：内容由AI生成）
