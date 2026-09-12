# Cargo.toml 文件解析

## 文件位置

`d:\microduck\duck-control\Cargo.toml`

## 核心设计决策

`duck-control` 的 `Cargo.toml` 定义了 crate 的元信息与依赖。每个依赖都附带详细注释，说明**为何需要该依赖**以及**版本约束的实际理由**——而非简单罗列。

## [package] 段

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `duck-control` | crate 名 |
| `version` | `workspace = true` | 从工作区继承版本 |
| `edition` | `workspace = true` | 从工作区继承 Rust 版本 |
| `license` | `workspace = true` | 从工作区继承许可证 |
| `description` | "Robot control core: model, bus, sensing" | 机器人控制核心：模型、总线、传感 |

## 依赖分析

### `duck-ipc-proto = { path = "../duck-ipc-proto" }`
仅为获取 `JOINT_NAMES`。关节*顺序*是协议——状态流将 `joints` 和 `targets` 作为裸数组传递——所以该表放在每个客户端都链接的 crate 中，本 crate 再导出而非手工维护第二份副本。这不是通往 IPC 的门：此处没有任何代码与 socket 通信，`robotd` 已链接两个 crate，所以不会有二进制因此获得新依赖。

### `serde = { workspace = true }`
工作区级依赖（序列化）。

### `thiserror.workspace = true`
错误类型派生宏。

### `tracing.workspace = true`
日志/追踪。

### `ort = { version = "=2.0.0-rc.11", default-features = false, features = ["load-dynamic"] }`
ONNX Runtime 绑定。`load-dynamic` 在首次使用时 dlopen libonnxruntime 而非链接，两个所需后果：
- aarch64 交叉编译在构建时无需目标架构的 ONNX Runtime
- 没有 libonnxruntime 的笔记本仍能编译并运行所有不创建会话的测试

版本固定为 `=2.0.0-rc.11`（microduck_runtime 当前运行的版本），保持板上已验证的 ABI。

### `libloading = "0.8"`
在触碰 `ort` 之前探测 ONNX Runtime dylib。与 `ort` 内部使用同一加载器，因此成功的探测意味着 `ort` 自身的加载也会成功——见 `policy::ensure_runtime`。

### `rustypot = "1.6.0"`
**1.6.0 是下限而非偏好**：它解包协议 2.0 状态包，其 payload 含 `FF FF FD`。此前，此类读返回过大，固定大小 sync_read 解码出垃圾。电流 -1 就足以触发，因此这是总线的正确性约束，非无谓版本升级。

### `serialport = { version = "4.8", default-features = false }`
`default-features = false` 去掉 serialport 的 `libudev` 特性（仅用于枚举端口 `available_ports`）。本 crate 按路径从参数文件打开一个端口，从不枚举；且 `libudev-sys` 需要板上交叉编译没有的 pkg-config sysroot，保留会直接破坏 aarch64 构建。rustypot 同样禁用它；本 crate 也必须禁用，否则 cargo 的特性统一会把它重新打开。

## 关键摘要

`Cargo.toml` 的每个依赖版本约束都有硬件/构建层面的实际理由：`rustypot >=1.6.0` 修复协议 2.0 状态包解码；`serialport` 禁用 `libudev` 以通过 aarch64 交叉编译；`ort` 固定 `2.0.0-rc.11` 并启用 `load-dynamic` 以避免构建时链接目标架构 ONNX Runtime；`libloading` 用于 `ort` 调用前的 dylib 探测。`duck-ipc-proto` 仅为复用 `JOINT_NAMES` 协议表。
