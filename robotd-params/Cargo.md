# Cargo.toml 文件解析

**文件位置**：`d:\microduck\robotd-params\Cargo.toml`

## 核心设计决策

从 `robotd/src/params.rs` 原样抽出，使 `robotctl configure` 能用真实类型、真实默认值、真实验证编辑文件，而非一个会漂移的副本。`robotd` 将其重新导出为 `crate::params`，自身无需改动。

## 依赖

- `kinematics`（路径）——特雷门琴默认值取自 `kinematics::hand::Config`。
- `duck-ipc-proto`（路径）——特雷门琴 socket 默认值取协议 crate 的常量；各 section 镜像它们所配置的守护进程。
- `serde`（workspace）——序列化/反序列化。
- `toml = "0.9"`——TOML 解析，依赖 `toml::Table` 实现 `without_unknown_keys` 的宽松路径。
- `thiserror`（workspace）——`ParamsError`。
- `tracing`（workspace）——缺失文件警告、未知 key 警告。

两个路径依赖都是纯 Rust 库 crate；本 crate 不引入 C 工具链或网络栈。

## 开发依赖

- `serde_json`（workspace）——测试辅助。
- `tempfile = "3.27.0"`——写临时 `robotd.toml` 测加载逻辑。

## 关键摘要

依赖极简：2 个路径库（取默认值常量）+ serde/toml/thiserror/tracing。无运行时 .so、无 GStreamer、无 ONNX——纯配置 schema crate。
