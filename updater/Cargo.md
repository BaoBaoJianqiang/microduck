# Cargo.toml 文件解析

## 文件位置

`d:\microduck\updater\Cargo.toml`

## [package]

`name = "updater"`，描述"机器人守护进程的签名、健康门控、回滚安全更新"。

## [[bin]]

`updaterd`（`src/main.rs`）。注释说明 proto 模块曾自包含，当 btd 需要协议或本 crate 增重依赖时才提取到 `duck-ipc-proto`——两者都是事件触发非猜测。

## 依赖

| 依赖 | 用途 |
|---|---|
| `duck-ipc-proto` | IPC 契约 |
| `libc` | `ETXTBSY` 命名 |
| `serde`/`serde_json` | 序列化 |
| `semver` | 版本比较 |
| `thiserror` | Error 派生 |
| `tokio` | 异步服务器+超时（rt-multi-thread/process/time/sync/io-util/signal） |
| `tokio-util` | LinesCodec |
| `async-trait` | Source/RobotClient dyn trait |
| `clap` | CLI |
| `fs4` | 磁盘空间 |
| `humantime-serde` | Duration 解析 |
| `tracing`/`tracing-subscriber` | 日志 |
| `minisign-verify` | 仅验证签名 |
| `sha2` | 哈希 |
| `tar`/`zstd` | 解压 |
| `toml` | 配置 |
| `futures-util` | 异步工具 |
| `reqwest` | HTTP（rustls，非 OpenSSL） |

## [dev-dependencies]

`test-support`、`axum`（本地 HTTP 服务器测试下载）、`minisign`（签名测试，dev-only）、`tempfile` 等。

## 关键摘要

Cargo.toml 驱动 updater 库与 updaterd 守护进程：核心依赖为 minisign-verify（仅验证）、sha2、tar/zstd、reqwest（rustls）、tokio、clap；设计注释说明 proto 在需要时才独立，避免客户端继承重型依赖树。
