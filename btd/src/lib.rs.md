# `lib.rs` 文件解析 —— btd crate 根

## 1. 文件定位

- 路径：`src/lib.rs`
- 角色：crate 根文件。只包含两部分内容：
  - crate 级文档注释（`//!`），说明 `btd` 是什么、为什么这样设计；
  - 全部子模块的声明（`pub mod ...`）。
- 本文件没有任何可执行逻辑，是整个 crate 的目录与架构说明。

## 2. 核心定位：只是一个传输适配器

文档第一句即点明：**A transport adapter and nothing else**（`architecture.md` §4.1）。`btd` 不持有任何状态，这不是风格偏好，而是承重设计：

- 如果配网（provisioning）或配置逻辑放在 `btd` 里，其他所有服务都要依赖 `btd`；
- 那样 SDK 想改机器人名字，居然必须经过蓝牙，这是荒谬的。

因此它的设计形态是一根**管道（pipe）**：

1. 对外暴露一个 GATT 服务，其中只有一个特征值（characteristic）——写它发请求，订阅它收响应；
2. 管道中承载的是与其他所有传输方式**完全相同的 NDJSON JSON-RPC 行**（每行一个 JSON 对象）；
3. `btd` 把分片重组成整行、对照 `route` 模块的方法表做权限检查、然后**逐字（verbatim）转发**给拥有答案的服务的 unix socket，再把回复分片发回；
4. 协议新增一个方法时，本 crate 只需在路由表中加一行。

## 3. 安全边界：解析不可信字节的进程不持权

`btd` 是直接解析"无线电范围内任何人"发来的字节的进程，所以它以**非特权用户**运行；而 `configd` 只接收来自本机、经过 peer 凭证校验的 socket 上的类型化 JSON，才是唯一以 root 运行的服务。

文档强调：把解析器放在这条信任边界的安全一侧，比加固分发器更重要。

## 4. 模块布局（Layout）

| 模块 | 性质 | 职责 |
| ---- | ---- | ---- |
| `framing` | 纯逻辑 | BLE 分片与 NDJSON 整行之间的重组/切分 |
| `route` | 纯逻辑 | BLE 允许哪些 JSON-RPC 方法、各转发给哪个上游服务 |
| `session` | 全部行为所在 | 一个中心设备一次连接的完整会话 |
| `link` | 接缝 | 无线电与可测试逻辑之间的接口：两个 channel，不是 trait |
| `upstream` | 连接管理 | 到 `updaterd` / `robotd` / `configd` 的 unix socket 连接 |
| `gatt` | 线路契约 | 服务与特征值的 UUID |
| `adv` | 线路契约 | 广播报文中 IPv4 地址字段的编码布局，与客户端共享 |
| `bluez` | Linux 专用 | 通过 bluetoothd 的 D-Bus API 操作 BlueZ 无线电 |
| `chorale` | 合唱信标 | 鸭子合唱（chorale）信标的广播与监听 |
| `pairing` | 配对 | 向 `configd` 索取配对 PIN |

可测试性的关键安排：`session` 只通过 `link::Link`（两个 channel + 一个普通结构体）接触无线电，因此测试可以在**真实的 unix socket** 上驱动完整会话，完全不涉及蓝牙。

## 5. 数据流上的两个"读回"特例

`btd` 对绝大多数请求只做转发，但有两样东西它会主动读回来：机器人的**名字**和 **IPv4 地址**。原因是两者都要放进广播报文：

- `net.*` 与 `system.*`（wifi、名字、重启）各占 `route` 表中的一支，发往 `configd`；
- `bluez` 向 `configd` 询问名字与地址，并让广播报文与之保持同步；
- `adv` 定义地址字段的字节布局，与解码该字段的客户端（`duckctl`）共享同一份代码。

## 6. 模块声明与条件编译

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

- 只有 `bluez` 用 `#[cfg(target_os = "linux")]` 门控，因为只有无线电部分依赖 Linux 上的 BlueZ；
- 其余模块全部平台无关，因此 crate 可以在 macOS 笔记本上编译并运行 `cargo test`——这是团队新人的入门路径（onboarding path）。

## 7. 本文件要点小结

1. `btd` 是无状态的传输管道：GATT ↔ NDJSON JSON-RPC ↔ 三个 unix socket；
2. 请求逐字转发，除路由表一行外不为新协议方法做任何改动；
3. 解析不可信无线电字节，故以非特权身份运行；
4. 纯逻辑（`framing`/`route`）、行为（`session`）、无线电（`bluez`）分层清晰，行为层用 channel 而非 trait 与无线电解耦；
5. 仅 `bluez` 为 Linux 专用，保证笔记本上可编译、可测试。
