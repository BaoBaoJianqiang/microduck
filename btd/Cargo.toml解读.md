# `Cargo.toml`（btd）解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 包清单（Cargo.toml） |
| 包名 | `btd` |
| 角色 | BLE 传输适配器——机器人 JSON-RPC API 的 GATT 前门 |
| 架构定位 | 纯传输适配器，无状态（architecture.md §4.1）；所有请求原样转发到拥有答案的服务的 unix socket |
| 运行平台 | Radxa（Linux）；但 crate 必须能在 macOS 笔记本上构建（`cargo test` 是新成员入职路径） |
| 关联文件 | `src/gatt.rs`（GATT UUID 线协议）、`src/route.rs`（路由测试）、`../duck-ipc-proto`（方法表） |

---

## 二、`[package]` 包元数据

| 字段 | 值 | 说明 |
|---|---|---|
| `name` | `"btd"` | 包名，与二进制名一致 |
| `version.workspace` | `true` | 版本从 workspace 根继承，避免多包版本不一致 |
| `edition.workspace` | `true` | Rust edition 从 workspace 继承 |
| `license.workspace` | `true` | 许可证从 workspace 继承 |
| `description` | `"BLE transport adapter — a GATT front door onto the robot's JSON-RPC API"` | 一句话定位：BLE 传输适配器，机器人 JSON-RPC API 的 GATT 前门 |

**设计要点**：
- `btd` 是**纯传输适配器，不拥有任何状态**。每个请求原样转发到拥有答案的服务（通过该服务的 unix socket）。
- 因此它只依赖**方法表**（`duck-ipc-proto`），不依赖任何实现行为的东西——没有更新引擎、没有控制环、没有 NetworkManager 客户端。
- 它也是**解析来自无线电范围内任何人的字节**的进程，这就是为什么它保持**非特权**运行，而 `configd` 才是以 root 运行的那个。

---

## 三、`[dependencies]` 通用依赖

| 依赖 | 版本/来源 | features | 说明 |
|---|---|---|---|
| `duck-ipc-proto` | `path = "../duck-ipc-proto"` | — | 内部 crate，定义 JSON-RPC 方法表与线协议；btd 只依赖它来知道"有哪些方法、转发到哪" |
| `serde_json` | workspace | — | JSON 序列化/反序列化，用于解析请求与转发响应 |
| `tokio` | workspace | `rt-multi-thread`, `macros`, `net`, `io-util`, `time`, `sync`, `signal` | 异步运行时：多线程运行时、宏、网络、IO 工具、定时器、同步原语、信号处理 |
| `clap` | workspace | — | 命令行参数解析 |
| `tracing` | workspace | — | 结构化日志（事件/span） |
| `tracing-subscriber` | workspace | `env-filter` | 日志订阅者，`env-filter` 支持通过 `RUST_LOG` 环境变量动态过滤 |
| `uuid` | `"1"` | — | GATT UUID 生成与解析。注释强调：GATT UUID 是**每个客户端都需要的线协议契约**，不是 Linux 细节（见 `src/gatt.rs`） |

**tokio features 解读**：
- `rt-multi-thread`：多线程异步运行时（而非单线程 current-thread）。
- `macros`：`#[tokio::main]` 等过程宏。
- `net`：TCP/Unix socket 等网络类型（btd 通过 unix socket 转发请求）。
- `io-util`：`AsyncReadExt`/`AsyncWriteExt` 等 IO 扩展 trait。
- `time`：`sleep`/`timeout`/`interval` 等定时器。
- `sync`：`mpsc`/`watch`/`Mutex` 等同步原语（会话逻辑由 channel 驱动）。
- `signal`：`SIGTERM`/`SIGINT` 等信号处理（优雅退出）。

---

## 四、`[target.'cfg(target_os = "linux")'.dependencies]` Linux 特定依赖

这是整个文件最关键的部分，注释量最大。

| 依赖 | 版本 | features | 说明 |
|---|---|---|---|
| `bluer` | `=0.17.4`（精确锁定） | `bluetoothd` | BlueZ 的 Rust 绑定，通过 bluetoothd 的 D-Bus API 操作蓝牙。GATT server API 是最可能变动的部分 |
| `dbus` | `"0.9"` | `vendored` | D-Bus 客户端库。`vendored` feature 从源码构建 libdbus（用 `cc`），而非链接目标系统的副本 |
| `futures` | `"0.3"` | — | Future 工具库（`StreamExt`/`SinkExt` 等），配合 bluer 的异步 API |

### 关键设计决策

#### 1. 为什么蓝牙部分放在 `cfg(target_os = "linux")` 后面

- **运行与构建是两个不同的主张**：btd 只在 Radxa（Linux）上*运行*，但 crate 必须能在 macOS 笔记本上*构建*——因为 `cargo test` 是新成员入职的完整路径（roadmap "Organisation"）。
- 只有第一个主张（运行在 Linux）为真，所以无线电部分放在 `cfg(target_os = "linux")` 后面；**会话逻辑由 channel 驱动，不需要无线电即可测试**。

#### 2. 为什么 `bluer` 精确锁定到 `=0.17.4`

- `bluer` 是 0.x 版本，**跨次版本都会破坏性变更**。
- 它的 GATT-server API 是最可能变动的部分。
- 因此用 `=0.17.4` 精确锁定而非浮动（`^`/`~`），避免 `cargo update` 意外拉到不兼容版本。

#### 3. 为什么 `dbus` 用 `vendored` feature

- `vendored` 从源码构建 libdbus，而非链接目标系统的 libdbus。
- **这是交叉编译能工作的原因**：`cargo board` 使用 `cargo-zigbuild`，`zig cc` 提供交叉 C 编译器（与构建 `zstd-sys` 是同一机制），但 sysroot 里有 libc、没有 libdbus。
- 已验证：完整的 aarch64 release 构建能干净链接。
- **代价**：这个 libdbus 由我们自己维护更新，而非发行版维护。对于一个只被我们自己写的守护进程通过本地 socket 访问的库来说可以接受，但需要记住它的存在。

---

## 五、`[dev-dependencies]` 开发依赖

| 依赖 | 版本/来源 | features | 说明 |
|---|---|---|---|
| `duck-ipc-proto` | `path = "../duck-ipc-proto"` | `test-support` | 启用测试支持 feature，提供 `every_call()` 等测试辅助工具（共享而非复制，见 `src/route.rs` 的测试模块） |
| `tempfile` | `"3.27.0"` | — | 临时文件/目录创建，用于测试 |

**`test-support` feature 的意义**：
- `every_call()` 是测试辅助函数，用于断言路由覆盖了每个方法调用。
- 放在 `duck-ipc-proto` 的 `test-support` feature 中，**共享而非在每个 crate 复制**。
- 仅在 dev-dependencies 中启用，不进入生产构建。

---

## 六、已移除的内容：examples

文件末尾注释说明：

> 这里不再有 examples。笔记本端客户端和广告观察器都需要 `btleplug`，两者都移到了 `duckctl/`——见其 manifest 了解这带来了什么，以及"永不在机器人上"的保证如何在迁移后存续。

- `btleplug` 是跨平台 BLE 库（支持 macOS/Windows/Linux），适合笔记本端工具。
- 把需要 `btleplug` 的 examples 移到 `duckctl/`，保持 `btd` 只依赖 Linux 上的 `bluer`，避免在机器人侧引入跨平台 BLE 库。
- "never on the robot"（永不在机器人上）的保证通过目录分离维持。

---

## 七、关键设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **传输与行为分离** | btd 无状态，只依赖方法表，不依赖更新引擎/控制环/NetworkManager |
| **最小权限** | 解析不可信字节的 btd 非特权运行；root 权限留给 configd |
| **构建可移植性** | 蓝牙部分用 `cfg(target_os = "linux")` 隔离，macOS 上可 `cargo test` |
| **依赖稳定性** | `bluer` 精确锁定 `=0.17.4`，避免 0.x 破坏性变更 |
| **交叉编译友好** | `dbus` 用 `vendored` feature，配合 `cargo-zigbuild` + `zig cc` 实现 aarch64 交叉编译 |
| **测试可独立** | 会话逻辑由 channel 驱动，不需要无线电即可测试 |
| **测试工具共享** | `every_call()` 放在 `duck-ipc-proto` 的 `test-support` feature，跨 crate 共享 |
| **平台依赖隔离** | 笔记本端工具（需 `btleplug`）移到 `duckctl/`，不污染机器人侧 |

---

## 八、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `duck-ipc-proto` | 内部方法表 crate，btd 依赖它知道转发目标；dev 依赖启用 `test-support` |
| `updater.toml` `allow_users = ["btd"]` | btd 被授权可转发 App 的更新请求（窄权限） |
| `configd` | 以 root 运行的进程，btd 非特权；两者形成权限分层 |
| `robotd.toml` `[audio]` | btd 不直接涉及音频，但 JSON-RPC API 可能暴露音频控制 |
| `duckctl/` | 笔记本端客户端与广告观察器的新家，需 `btleplug` |
| `cargo-zigbuild` / `zig cc` | 交叉编译工具链，与 `dbus/vendored` 配合 |
| `architecture.md §4.1` | 定义 btd 作为纯传输适配器的架构定位 |

---

## 九、依赖速查表

| 类别 | 依赖 | 版本 | 平台 | 用途 |
|---|---|---|---|---|
| 内部 | `duck-ipc-proto` | path | 全部 | 方法表/线协议 |
| 序列化 | `serde_json` | workspace | 全部 | JSON |
| 异步 | `tokio` | workspace | 全部 | 运行时/网络/IO/同步/信号 |
| CLI | `clap` | workspace | 全部 | 参数解析 |
| 日志 | `tracing` / `tracing-subscriber` | workspace | 全部 | 结构化日志 + env 过滤 |
| 协议 | `uuid` | `1` | 全部 | GATT UUID |
| 蓝牙 | `bluer` | `=0.17.4` | Linux only | BlueZ D-Bus GATT |
| 总线 | `dbus` | `0.9` (vendored) | Linux only | D-Bus 客户端（源码构建） |
| 异步工具 | `futures` | `0.3` | Linux only | Stream/Sink 扩展 |
| 测试 | `duck-ipc-proto` (test-support) | path | dev | `every_call()` 等 |
| 测试 | `tempfile` | `3.27.0` | dev | 临时文件 |
#（注：内容由AI生成）
