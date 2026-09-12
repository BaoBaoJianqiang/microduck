# `Cargo.toml` 文件解析 —— btd crate 的清单

## 1. 文件定位

- 路径：`Cargo.toml`（btd 包根；workspace 成员之一）
- 角色：声明包元信息与全部依赖。注释本身承载了大量架构决策记录。

## 2. `[package]` 元信息

```toml
name = "btd"
version.workspace = true
edition.workspace = true
license.workspace = true
description = "BLE transport adapter — a GATT front door onto the robot's JSON-RPC API"
```

- `version` / `edition` / `license` 均继承自 workspace（`.workspace = true`），保证多包一致；
- 描述一句话定位：BLE 传输适配器——机器人 JSON-RPC API 之上的 GATT 前门。

顶部注释重申架构原则（architecture.md §4.1）：

- btd **只是传输适配器、不持状态**：每个接受的请求都逐字转发给拥有答案的服务（经其 unix socket）；
- 因此它只依赖"方法表"，不依赖任何实现行为的东西——没有更新引擎、没有控制循环、没有 NetworkManager 客户端；
- 它也是解析"无线电范围内任何人"所发字节的进程，所以保持非特权，而以 root 运行的是 `configd`。

## 3. 通用依赖 `[dependencies]`（全平台）

| 依赖 | 用途 |
| ---- | ---- |
| `duck-ipc-proto = { path = "../duck-ipc-proto" }` | 本地路径依赖的协议 crate：JSON-RPC 方法表、类型、socket 常量、API 版本 |
| `serde_json`（workspace） | JSON 序列化/反序列化 |
| `tokio`（workspace） | 异步运行时；启用特性 `rt-multi-thread`、`macros`、`net`、`io-util`、`time`、`sync`、`signal` |
| `clap`（workspace） | 命令行参数解析 |
| `tracing`（workspace） | 结构化日志/诊断的埋点面 |
| `tracing-subscriber`（workspace，`env-filter`） | 日志订阅者，支持用 `RUST_LOG` 过滤 |
| `uuid = "1"` | UUID 类型。GATT UUID 是每个客户端都需要的线路契约，而非 Linux 细节（见 `src/gatt.rs`） |

## 4. Linux 专用依赖

```toml
[target.'cfg(target_os = "linux")'.dependencies]
bluer = { version = "=0.17.4", features = ["bluetoothd"] }
dbus = { version = "0.9", features = ["vendored"] }
futures = "0.3"
```

注释解释了几个关键决策：

- **只在运行层面是 Linux/Radxa 专用，构建层面仍要支持 macOS 笔记本**：`cargo test` 在笔记本上是团队成员的完整入门路径（roadmap "Organisation"）。这是两个不同命题，故只有无线电部分放在 `cfg(target_os = "linux")` 之后，会话逻辑由 channel 驱动、无需无线电即可测试；
- **bluer 精确钉版 `=0.17.4`**：它还是 0.x，小版本之间就会破坏兼容，而其 GATT 服务端 API 又是最可能变动的部分，因此钉死而非浮动；
- **dbus 启用 `vendored`**：用 `cc` 从源码构建 libdbus，而不是链接目标系统的副本。这是交叉编译能成立的关键——`cargo board` 用 cargo-zigbuild，`zig cc` 提供交叉 C 编译器（与构建 `zstd-sys` 同一机制），但 sysroot 里有 libc、没有 libdbus。已验证完整 aarch64 release 构建链接干净；
- vendored 的代价：这份 libdbus 的更新责任在自己而非发行版；对一个只被我们自己写的守护进程经本地 socket 访问的库而言可以接受，但需要记得它的存在。

## 5. 开发依赖 `[dev-dependencies]`

```toml
duck-ipc-proto = { path = "../duck-ipc-proto", features = ["test-support"] }
tempfile = "3.27.0"
```

- 协议 crate 在测试中额外开启 `test-support` 特性，提供 `every_call()`——共享而非复制的全量调用清单（见 `src/route.rs` 测试，曾因两份复制清单漂移而漏方法）；
- `tempfile` 用于在测试中创建临时目录承载 unix socket（`session.rs`、`pairing.rs`、`upstream.rs` 均用到）。

## 6. 关于 examples 的说明（文件尾注释）

- 本包已不再放 examples；
- 笔记本侧客户端与广播观察器都需要 `btleplug`，二者已迁到 `duckctl/`；迁移带来的好处以及"绝不在机器人上运行"的保证如何在迁移后保持，见 `duckctl` 自己的清单。

## 7. 本文件要点小结

1. btd 是 workspace 成员，版本/版次/许可证统一继承；
2. 全平台依赖支撑"纯逻辑可在笔记本测试"，无线电三件套（bluer/dbus/futures）仅 Linux；
3. bluer 精确钉版防 0.x 破坏；dbus vendored 是 zigbuild 交叉编译到 aarch64 的前提；
4. dev-dependencies 提供全量调用清单与临时目录，支撑安全边界与 socket 测试；
5. 示例程序已迁出，保证机器人侧不引入 `btleplug`。
