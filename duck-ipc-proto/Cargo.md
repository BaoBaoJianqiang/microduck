# Cargo.toml 文件解析

**文件位置**：`d:\microduck\duck-ipc-proto\Cargo.toml`

## 一、包元信息

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `duck-ipc-proto` | crate 名 |
| `version` | `workspace = true` | 继承工作区版本（所有 workspace crate 共享一个版本线，因为打包在同一 artifact 中） |
| `edition` | `workspace = true` | 继承工作区 edition |
| `license` | `workspace = true` | 继承工作区许可证 |
| `description` | `IPC contracts between the robot's services and their clients` | 机器人服务与其客户端之间的 IPC 契约 |

## 二、核心设计决策：刻意近乎零依赖

注释明确说明：每个服务和客户端都使用这些类型，因此此处添加的任何依赖都会加到所有它们身上——包括 `btd`，而 `btd` 必须保持小巧，因为它是恢复路径的一部分（`architecture.md` §1.1）。

**明确禁止引入**：`http`、`tar`、`crypto`、`tokio`。若某类型需要这些，它属于拥有该行为的 crate，而不属于此处。

## 三、依赖分析

### 3.1 正式依赖（3 个）

| 依赖 | 来源 | 作用 |
|---|---|---|
| `serde` | `workspace = true` | 序列化/反序列化框架，所有 params/results 类型的 `Serialize`/`Deserialize` derive |
| `serde_json` | `workspace = true` | JSON 线格式编解码（`serde_json::to_value`/`from_value`/`to_string`/`from_str`） |
| `semver` | `workspace = true` | 语义化版本，`Target::Exact`、`SelectParams::version`、`HelloResult::daemon_version`、`release_from_path` 等使用；`pub use semver` 重导出以避免消费者出现两份不兼容的 `Version` |

### 3.2 开发依赖（1 个）

| 依赖 | 版本 | 作用 |
|---|---|---|
| `tempfile` | `3.27.0` | 仅用于 identity 往返测试（`an_identity_survives_being_published_and_read_back`），需要可写的 runtime root，而测试不能写 `/run`，故通过 `DUCK_RUNTIME_DIR` 指向临时目录 |

### 3.3 features

```toml
[features]
test-support = []
```

`test-support` 特性启用 `src/lib.rs` 中的 `test_support` 模块（`every_call()` 函数），供本 crate 及其消费者（如 `btd::route`、`mediad`）的测试使用。

**不添加任何依赖**——本 crate 处于恢复路径上，依赖列表刻意只有三个 crate。`#[cfg(any(test, feature = "test-support"))]` 使本 crate 自身的 `cargo test` 无需启用特性即可访问 `every_call`。

设计理由注释中说明：之前 `every_call` 的两份副本（此处和 `btd::route`）已经漂移（115 行 vs 82 行），第三份即将为 `mediad` 编写，因此集中到此处共享。

## 四、关键摘要

1. **三个依赖是硬约束**：serde、serde_json、semver，因为 `btd` 在恢复路径上不能更重
2. **`pub use semver`**：重导出 semver，防止消费者单独依赖产生两份不兼容的 `Version` 类型
3. **`tempfile` 仅 dev**：测试 runtime root 可写性，不进入生产二进制
4. **`test-support` 特性无依赖**：共享 `every_call()` 测试夹具，避免消费者各自维护已漂移的副本
