# Cargo.toml 文件解析

**文件位置**：`d:\microduck\padd\Cargo.toml`

## 核心设计决策

独立 crate 使手柄栈不进入 `robotctl`（后者须在坏机器人上工作，是恢复路径的一部分；此处无任何东西在该路径上）。

## 依赖

- `duck-ipc-proto`（路径）——意图 wire 类型。
- `clap`（workspace, derive）——CLI。
- `gilrs = "0.11"`——手柄抽象。**在 Linux 上无条件依赖 libudev-sys**（无 feature 可关），所以 CI 与板交叉编译都装 libudev。代价是为保留 gilrs 的 SDL 控制器数据库——没有它需手动映射每只 pad 的原始 evdev 码，而同一只 Xbox 手柄 USB 与蓝牙上报不同码。
- `serde`/`serde_json`（workspace）——仅序列化 tap 自己的行。
- `tracing`/`tracing-subscriber`（workspace, env-filter）。

### Linux 目标依赖
- `evdev = "0.13"`——原始事件流，`raw_stream::RawDevice` 不对 SYN_DROPPED resync（链路调查最需看到的事件）。
- `libc = "0.2"`——`getgrnam`/`chown`，把 tap socket 交给 robot 组。仅绑定无 C 构建。

`evdev` 是纯 Rust，是"在路径上优先纯 Rust crate"论点的实例——下一个必须到板的 C 依赖会再付一次 libudev 代价。

## 关键摘要

依赖分两层：通用层（gilrs+协议）与 Linux-only 层（evdev 原始 tap）。gilrs 引入 libudev 是本 crate 唯一 C 依赖，为控制器数据库付出。tap 的 evdev 选 raw_stream 变体以保留 SYN_DROPPED。
