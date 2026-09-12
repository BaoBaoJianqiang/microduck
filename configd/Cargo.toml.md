# Cargo.toml 文件解析

## 1. 文件定位

- **路径**：`configd/Cargo.toml`
- **角色**：`configd`（「Wifi 与机器人身份——配置服务」）的 crate 清单，声明包元数据、依赖与平台条件依赖。包内注释同样承载了架构决策说明。

## 2. `[package]` 段

| 字段 | 取值 | 说明 |
| --- | --- | --- |
| `name` | `configd` | 二进制/库名 |
| `version.workspace` | `true` | 版本继承工作区 |
| `edition.workspace` | `true` | Rust edition 继承工作区 |
| `license.workspace` | `true` | 许可证继承工作区 |
| `description` | `Wifi and robot identity — the config service` | 包描述 |

紧随其后的注释复述了独立成服务的理由（`architecture.md` §3.1、§4.1）：配置在 `robotd` 死亡时仍须可达；不属于 `btd`，否则 SDK 得绕蓝牙改名；并再次强调**不存凭据，凭据归 NetworkManager**。

## 3. 通用依赖 `[dependencies]`

| 依赖 | 版本/来源 | 启用特性与用途 |
| --- | --- | --- |
| `duck-ipc-proto` | `{ path = "../duck-ipc-proto" }` | 本地路径依赖，IPC 协议类型（`Request`/`Response`/`Call`、`Net*`、`Pad*`、socket 路径常量、`build_info!` 等） |
| `serde` | workspace | 序列化框架 |
| `serde_json` | workspace | 行式 JSON 报文编解码 |
| `tokio` | workspace | 异步运行时；特性 `rt-multi-thread`、`macros`、`net`、`io-util`、`time`、`sync`、`signal` |
| `clap` | workspace | 命令行参数解析（`--socket`、`--state-dir`、`--allow-user/group`、`--fake-net`、`--fake-pads`） |
| `tracing` | workspace | 结构化日志 |
| `tracing-subscriber` | workspace，特性 `env-filter` | 日志订阅者，支持 `RUST_LOG` 过滤 |
| `async-trait` | workspace | 为 `Net`/`Pads` trait 提供 `async fn` |
| `libc` | `0.2` | `getpwnam`/`getgrnam`/`getuid` 等，用于把用户名/组名解析为 uid/gid |
| `sha2` | `0.11.0` | 由 SoC 序列号派生默认名时做 SHA-256 |

关于 `sha2` 的注释要点：

- 用密码学摘要而不是 `std` 的 hasher，因为 **std 的哈希输出不保证跨 Rust 版本稳定**；一次工具链升级绝不能把现场每台机器人悄悄改名。
- 该依赖本已通过 updater 的发布校验进入依赖图，并非新增负担。

## 4. Linux 条件依赖 `[target.'cfg(target_os = "linux")'.dependencies]`

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| `zbus` | `5` | 纯 Rust D-Bus 客户端，同时对接 NetworkManager、logind、systemd、BlueZ |
| `futures` | `0.3` | 提供 `StreamExt`，用于消费 NM 的 `StateChanged` 信号流 |

注释解释了两个关键取舍：

1. **后端放在 trait 之后、配内存假实现**，使 crate 在 macOS 笔记本上也能构建和测试（`cargo test` 是新人上手路径），D-Bus 客户端在非 Linux 下根本不进入依赖图。
2. **选 `zbus` 而非 `dbus` crate**：纯 Rust、无需交叉编译 vendored C；NM 的设置是嵌套的 `a{sa{sv}}`，`zvariant` 表达起来毫不费力。代价是产物里带上两套 D-Bus 栈（`btd` 经 `bluer` 链接 libdbus）；若将来 `bluer` 长出 zbus 后端，值得回头重新评估。
3. `futures` 的存在原因：zbus 5 只再导出 `futures_core`（`Stream` trait），不导出 `futures_util`（组合子）；连接路径必须消费 NM 的 `StateChanged` 信号才能知道一次激活*为什么*失败。它本已通过 btd 进入依赖图。

## 5. 开发依赖 `[dev-dependencies]`

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| `tempfile` | `3.27.0` | 单元测试中创建临时目录（`identity`、`store` 的文件读写测试） |

## 6. 要点小结

- 依赖选择严格服务于架构：协议走本地 path crate；异步用多线程 tokio；授权所需的名字解析用 `libc`；默认名派生用跨版本稳定的 `sha2`。
- D-Bus 统一用纯 Rust 的 `zbus 5`，只在 Linux 下编入，保证笔记本可测、交叉编译简单。
- 唯一的 dev 依赖是 `tempfile`，用于文件类测试。
