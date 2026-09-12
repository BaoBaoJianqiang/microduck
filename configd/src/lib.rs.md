# lib.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/lib.rs`
- **角色**：`configd` crate 的根模块（库入口），只包含 crate 级文档注释与模块声明。它用一段总纲说明回答了「`configd` 为什么是一个独立服务」这一根本问题，并组织本 crate 的全部子模块。

## 2. 核心设计决策（crate 级文档）

文件开头的 `//!` 文档注释集中陈述了五条设计立场：

1. **配置必须在 `robotd` 死亡时仍然可达**（引 `architecture.md` §3.1）。当机器人出故障时，客户端最需要做的恰恰是配网，因此配网能力不能放在控制守护进程 `robotd` 里。
2. **不能放进 `btd`，因为 `btd` 不拥有任何状态**（§4.1）。如果改名逻辑活在 BLE 服务里，SDK 将不得不荒谬地绕一圈蓝牙才能设置机器人名字。
3. **因此它是刻意独立的第五个服务**：一个 socket，承载 `net.*`、`system.*`、`pad.*` 三组方法；BLE、`robotctl`、以及未来 `mediad` 的远程网关，全部是它的客户端，而不是各自重新实现一遍。
4. **`pad.*` 放在这里与 `net.*` 同理**：手柄配对是关于无线电配置的、仅 root 可做的事；而真正*读取*手柄的进程 `padd` 被刻意设计为没有任何特权（它每天走的是手机 App 将使用的同一套 intent API，以免该 API 悄悄腐化）。
5. **不存储任何凭据**。凭据归 NetworkManager 所有：由 NM 以 root-only 权限持久化并自行重连；`configd` 只是把口令递过去然后忘记。它自己拥有的只是一个很小的 [`store`] 文件：机器人名字，以及后续可能加入的身份信息。

关于**以 root 运行**的说明：这并非本仓库偏好的默认做法，理由很窄——logind 的 `Reboot` 受 polkit 门控，而板上没有 polkit，无会话的非 root 调用者会被直接拒绝。信任边界仍然处在正确的位置：`btd` 是解析无线电覆盖范围内任何人发来字节的进程，它是非特权的；`configd` 见到的永远只是来自带对端凭证的本地 socket 的、有类型的 JSON。配套沙箱见 `systemd/configd.service`。

## 3. 模块声明

| 声明 | 条件编译 | 说明 |
| --- | --- | --- |
| `pub mod bluez;` | `#[cfg(target_os = "linux")]` | 经 BlueZ D-Bus 做手柄配对，Linux 专有 |
| `pub mod identity;` | 无 | 每台设备的身份（SoC 序列号）与派生默认名，平台无关 |
| `pub mod net;` | 无 | `Net` trait、`UnavailableNet` 与内存假实现 `FakeNet`，平台无关 |
| `pub mod nm;` | `#[cfg(target_os = "linux")]` | NetworkManager 的生产实现，Linux 专有 |
| `pub mod pad;` | 无 | `Pads` trait、`FakePads` 与手柄识别启发式，平台无关 |
| `pub mod power;` | 无 | 经 logind 重启；非 Linux 下返回错误 |
| `pub mod store;` | 无 | 配置文件存储（名字、配对 PIN） |
| `pub mod units;` | 无 | 查询 systemd 单元状态并读取各守护进程自发布的身份 |

设计要点：**平台差异被隔离在 `bluez`/`nm` 两个 Linux 专有后端之后**，其余模块全部平台无关，使得整个 crate 在 macOS 笔记本上仍可 `cargo test`（这是仓库的新人上手路径）。

## 4. 要点小结

- 本文件几乎没有可执行代码，它是 `configd` 的「宪法」：声明服务独立性、信任边界、不存凭据三条底线。
- 一个 socket、三组方法（`net.*`/`system.*`/`pad.*`）、多种传输共用。
- root 身份是为 logind 重启这一个狭窄理由而设，并由 systemd 沙箱收紧。
- Linux 专有代码仅 `bluez` 与 `nm`，其余模块在任何平台可编译、可测试。
