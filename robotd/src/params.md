# params.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\params.rs`

## 核心设计决策

### 1. 参数类型的物理迁出

本文件**不定义任何类型**，仅通过 `pub use robotd_params::*;` 重导出 `robotd-params` crate 的全部内容。

设计理由：

- **避免副本漂移**：原型期参数定义散落在 `robotd` 内部，`robotctl configure` 编辑配置文件时只能维护一份手写副本（schema、默认值、校验逻辑），两者极易失步。
- **单一真相源**：类型、默认值、校验全部集中在 `robotd-params`，`robotd` 与 `robotctl` 共享同一套编译期 schema。
- **保留模块名**：使用 `mod params;` + 重导出，使 `robotd` 内部代码无需改动 import 路径。

## 内容

```rust
pub use robotd_params::*;
```

仅此一行。

## 关键摘要

`params.rs` 是 `robotd-params` 的薄适配层。它的全部价值在于「不定义任何东西」——把配置真相源锁定到一个独立 crate，使守护进程与配置工具不可能各说各话。
