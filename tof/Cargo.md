# Cargo.toml 文件解析

## 文件位置

`d:\microduck\tof\Cargo.toml`

## [package]

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `tof` | 头部 ToF 传感器驱动与帧形状 |
| `version` | workspace | |
| `edition` | workspace | |
| `rust-version` | workspace | |
| `license` | workspace | |

## [lib] / [[bin]]

- lib 名 `tof`
- bin `tofd`（`src/main.rs`）

## 依赖

| 依赖 | 版本 | 用途 |
|---|---|---|
| `anyhow` | 1 | 错误处理 |
| `clap` | workspace，features=["derive"] | CLI 参数解析 |
| `duck-ipc-proto` | path | 协议类型与 socket 路径常量 |
| `libc` | 0.2 | `getgrnam`/`chown` 改 socket 组 |
| `serde` | workspace | 序列化 |
| `serde_json` | workspace | JSON-RPC 行协议 |
| `tokio` | workspace，features 含 macros/net/rt/signal/sync/time/io-util | 异步 socket 服务器（current_thread） |
| `tracing` | workspace | 日志 |
| `tracing-subscriber` | workspace，features=["env-filter"] | 日志格式化 |

## [build-dependencies]

| 依赖 | 版本 | 用途 |
|---|---|---|
| `cc` | 1 | 编译 vendored C ULD |

## 关键摘要

Cargo.toml 驱动与守护进程双用途：lib 提供 `Frame`/`Zone`/`Sensor`，bin `tofd` 发布帧。唯一 C 相关依赖是 build-dependency `cc`（编译 ST ULD），无运行时系统库。`libc` 仅用于 socket 属组设置。
