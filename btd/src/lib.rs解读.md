# `lib.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust crate 根模块（lib.rs） |
| 行数 | 44 行 |
| 角色 | `btd` crate 的入口，定义模块结构与顶层架构文档 |
| 平台 | 跨平台（`bluez` 模块用 `cfg(target_os = "linux")` 隔离） |

## 二、crate 顶层文档

`btd` 是机器人 API 的 **BLE 前门**，核心定位：

### 1. 纯传输适配器，无状态（architecture.md §4.1）

- `btd` 不拥有任何状态，这是**承重设计**而非整洁：如果配置或配置信息住在这里，每个其他服务都要依赖 `btd`，SDK 就会荒谬地必须通过蓝牙来设置机器人名字。
- 设计是一根**管道**：一个 GATT 服务，一个 characteristic——写入请求、订阅响应——承载**与其他传输相同的 NDJSON JSON-RPC 行**。
- `btd` 重组这些行，对照 `route` 表检查方法，原样转发到拥有答案的服务的 unix socket，再把回复分块发回。
- 添加协议方法只需在路由表中加一行，此处无需改动。

### 2. 权限边界

- `btd` 是解析来自无线电范围内任何人的字节的进程，因此**非特权运行**。
- `configd` 只看到来自对等凭据本地 socket 的类型化 JSON，因此以 root 运行。
- 把解析器放在边界的安全侧，比加固分发器更重要。

## 三、模块布局

| 模块 | 角色 | 说明 |
|---|---|---|
| `framing` | 纯逻辑 | 字节分块与重组 |
| `route` | 纯逻辑 | 方法白名单与路由表 |
| `session` | 全部行为 | 会话逻辑，只通过 `link::Link`（两个 channel，非 trait）接触无线电 |
| `link` | 接缝 | 无线电与可测试逻辑之间的接口 |
| `upstream` | 上游连接 | 持有到拥有答案的服务的连接 |
| `bluez` | 无线电（Linux only） | BlueZ 实现，`cfg(target_os = "linux")` 隔离 |
| `gatt` | 线协议契约 | GATT UUID，平台无关 |
| `adv` | 广播布局 | 地址字段布局，与客户端共享 |
| `pairing` | 配对 | PIN 检查与配对逻辑 |
| `chorale` | 合唱无线电 | 鸭子合唱的信标广播与监听 |

### 测试可独立

- `session` 只通过 `link::Link` 接触无线电——两个 channel，不是 trait。
- 因此测试可以在**真实 unix socket** 上驱动完整会话，无需蓝牙参与。

## 四、特殊数据流

- `net.*` 和 `system.*`（wifi、名字、重启）→ `configd`，在 `route` 表中各占一个分支。
- **机器人名字和 IPv4 地址**是 `btd` 读回而非仅转发的两个东西：两者都进入广播，因此 `bluez` 向 `configd` 请求它们并保持广播同步。
- `adv` 是地址字段的布局，与解码它的客户端共享。

## 五、模块声明

```rust
pub mod adv;
#[cfg(target_os = "linux")]
pub mod bluez;
pub mod chorale;
pub mod framing;
pub mod gatt;
pub mod link;
pub mod pairing;
pub mod route;
pub mod session;
pub mod upstream;
```

- 仅 `bluez` 被 `cfg(target_os = "linux")` 隔离，因为它是唯一需要 BlueZ 的模块。
- 其余模块全部跨平台，确保 macOS 上可 `cargo test`。
#（注：内容由AI生成）
