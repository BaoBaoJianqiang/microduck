# `route.rs` 解读

## 概述

`route.rs` 是 mediad 的**路由权限表**，决定一个通过 WebRTC datachannel 到达的 JSON-RPC 调用是否允许被服务，以及如果允许，转发给哪个后端服务。

它是 `btd`（BLE 守护进程）中 `btd::route` 的兄弟模块，刻意采用相同结构：

> *The sibling of `btd::route`, and deliberately structured the same way: which service answers a call and how long answering holds a connection lives once, in `proto::Call::destination`, and this file answers only may a peer over this transport ask it.*
> —— `btd::route` 的兄弟模块，刻意采用相同结构：哪个服务应答一个调用、应答占用连接多久，这些只存在于 `proto::Call::destination` 一处；本文件只回答"这个传输上的 peer 能不能问"。

**核心设计理念**：match 是穷尽的（exhaustive），这是为每种传输单独写一份路由表的根本原因：

> *Adding a variant to `proto::Call` fails the build here as well as in `btd`, so a new method cannot reach a remote peer because nobody remembered this file. A shared table with a `_` wildcard would have been the hole in both transports at once.*
> —— 在 `proto::Call` 上加一个变体，这里和 `btd` 都会编译失败，所以新方法不会因为没人记得这个文件就意外到达远程 peer。一个共享的带 `_` 通配符的表会同时在两种传输上都开个洞。

## 为什么 WebRTC 比 BLE 允许的调用更多

BLE 的路由表很窄，原因有二：
1. 无线电慢——20 字节通知预算
2. 几米内任何人都能连

datachannel 既不慢也不限于房间内，所以 BLE 因**容量原因**拒绝的调用，恰恰是这个传输存在的理由：意图、遥测、触控垫输入、深度流。

被排除的调用分三类，原因各不相同：

### 第一类：授权了另一个传输

> *It authorises a different transport. The pairing PIN. A peer that can rewrite it can lock a phone out of BLE, which is the recovery path.*
> —— 它授权了另一个传输。配对 PIN。能改写它的 peer 可以把手机锁在 BLE 外面，而 BLE 是恢复路径。

`SystemPairingPin` / `SystemSetPairingPin` 被拒绝。PIN 在所有网络传输上都不可路由，理由和它对 BLE 自身不可路由一样。

### 第二类：会切断当前会话

> *It would drop the session it was asked over. The update mutations, and reconfiguring wifi.*
> —— 它会切断被请求的会话。更新操作，以及重新配置 WiFi。

`Apply` / `Rollback` / `Select` 被拒绝：应用更新会重启 mediad，而客户端正在观看更新进度。注释明确这是**延期而非规则**（*a deferral, not a rule*）——手机通过 WebRTC 更新机器人是想要的功能，只是需要客户端支持重连重订阅。

`NetConnect` / `NetForget` 被拒绝：重新配网会把机器人搬到另一个网络，把当前会话一起带走。和更新不同，这里没有"延期到"的去处——配网是 BLE 存在的意义。

### 第三类：不是客户端该问的

> *It is not a client's question. `updaterd`'s internal queries to `robotd`.*
> —— 不是客户端的问题。`updaterd` 对 `robotd` 的内部查询。

`RobotSafeToRestart`、`RobotModelApi`、`RobotRemoteSessionActive` 被拒绝。

## 关键结构体与函数

### `Route` 枚举

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Route {
    /// Forwarded verbatim to a service, on that service's connection for this lane.
    /// —— 逐字转发给某个服务，在该服务的这条 lane 连接上。
    To(proto::Service, proto::Lane),
    /// Not available over this transport.
    /// —— 在本传输上不可用。
    Refused,
}
```

路由结果只有两种：转发到某个服务，或拒绝。

### `permits` — 权限判定（核心）

```rust
fn permits(call: &proto::Call) -> bool
```

对每个 `proto::Call` 变体做穷尽 match，返回 true/false。这是整个文件的核心决策表。

#### 允许的调用分组

**版本握手**：
- `Hello(_)` —— 必须可达，否则客户端什么都建立不了。

**机器人运动控制**（BLE 因容量拒绝的，这里全放行）：
- `RobotMove` / `RobotHead` / `RobotLook` / `RobotPose` / `RobotMouth`
- `RobotDo` / `RobotSound` / `RobotTheremin` —— 特雷门琴随声音一起，浏览器能叫鸭子就能选乐器。

**合唱（chorale）被拒绝**：
- `RobotChorale` / `ChoraleSubscribe` / `ChoraleBeaconSet` / `ChoraleHeard`
> *The chorale is between robots, over BLE — a browser is neither in the room nor a duck.*
> —— 合唱是机器人之间通过 BLE 进行的——浏览器既不在房间里也不是鸭子。

**状态与遥测**：
- `RobotHealth` / `RobotMode` —— 读状态。
- `RobotSubscribe` —— BLE 说这是"往 20 字节管子里倒消防水带"，这里通道就是为流设计的。

**关节使能与站立**：
- `RobotEnable` / `RobotInit` / `RobotRelax`
> *A peer holding this session has the camera: it is looking at the robot, which is precisely what a phone in the room over Bluetooth was not.*
> —— 持有本会话的 peer 有摄像头：它正在看机器人，这正是房间里蓝牙手机做不到的。

**停止**：
- `RobotStop`
> *BLE's objection was that a stop button over "an unbonded, high-latency, sometimes-absent radio" is worse than no button because it looks like an e-stop and is not one. The control channel is reliable and ordered, and the deadman already stops the robot when intents stop arriving.*
> —— BLE 的反对理由是：在"未绑定、高延迟、时有时无的无线电"上放停止按钮比没有按钮更糟，因为它看起来像急停按钮但不是。control 通道可靠有序，deadman 机制在意图停止到达时已经会停机器人。

**关机**：
- `RobotShutdown` —— 和 `RobotInit` 同理：peer 能看到机器人坐下。

**模式切换被拒绝**：
- `RobotSetMode(_)`
> *A mode switch says "this duck now has wheels on it", which is a claim about hardware only somebody in the room can make.*
> —— 模式切换等于声明"这只鸭子现在装了轮子"，这是关于硬件的断言，只有在房间里的人能做。

**触控垫与深度流**：
- `PadInput` —— BLE 拒绝是因为"测量的是手机链路而非垫子链路"，datachannel 没这个问题。`mediad` 可以持有到 `padd` 的 socket。
- `TofStream` —— BLE 的拒绝理由明确点名这个传输是它的家："depth belongs next to the frame it annotates"（深度数据该待在它标注的帧旁边）。

**软件状态读取**：
- `Check` / `Status` / `Subscribe` / `Log` / `Show` / `ListInstalled`
> *Show is the largest reply in this list — a run's whole transcript — and a datachannel is the one transport here with the room for it.*
> —— `Show` 是这个列表里最大的回复——一次运行的完整记录——datachannel 是这里唯一有空间容纳它的传输。

**系统信息与网络状态**：
- `SystemInfo` / `SystemServices` / `SystemSetName`
- `SystemReboot` —— 重启后客户端重连即可，不会留下过渡态。
- `NetStatus` / `NetScan` —— 只读网络状态，帮远程操作者判断链路差的原因。
- `PadStatus` / `PadForget`

#### 拒绝的调用分组

**配对 PIN（授权其他传输）**：
- `SystemPairingPin` / `SystemSetPairingPin` → false

**更新操作（会切断会话）**：
- `Apply` / `Rollback` / `Select` → false

**网络配置（会切断会话，且无处延期）**：
- `NetConnect` / `NetForget` → false

**手柄配对（远程做不到）**：
- `PadPair(_)` → false
> *Bonding a gamepad needs a pad in the room, in pairing mode, in a fifteen-second window. A remote peer cannot satisfy any of that.*
> —— 绑定手柄需要房间里有个手柄在配对模式、十五秒窗口内。远程 peer 一样都满足不了。

**恢复出厂与锁定（永不允许）**：
- `ResetToGolden` → false
> *Factory reset in all but name: back to the golden image, discarding every release since.*
> —— 名义不同的恢复出厂：回到黄金镜像，丢弃之后所有版本。

- `Pin(_)` → false
> *A robot pinned by mistake refuses every later update and reports itself as up to date. That is the one failure here that looks exactly like correct behaviour.*
> —— 被错误锁定的机器人会拒绝之后所有更新并报告自己已是最新。这是这里唯一一种看起来和正确行为一模一样的失败。

**内部查询（不是客户端的问题）**：
- `RobotSafeToRestart` / `RobotModelApi` / `RobotRemoteSessionActive` → false

**认证（本传输不认证）**：
- `SystemAuthenticate(_)` → false
> *Answering would be a lie. §4: there is no gate here — a LAN peer may drive the robot, and a bridged one authenticated to the rendezvous service before arriving.*
> —— 回答了就是撒谎。§4：这里没有闸门——局域网 peer 可以驾驶机器人，桥接 peer 在到达前已向会合服务认证过。

### `route_for` — 路由入口

```rust
pub fn route_for(call: &proto::Call) -> Route {
    if !permits(call) {
        return Route::Refused;
    }
    match call.destination() {
        Some((service, lane)) => Route::To(service, lane),
        None => Route::Refused,
    }
}
```

**组合顺序很重要**：先查 `permits`，再查 `destination`。注释指出：

> *Permitted but owned by no service. Only `system.authenticate` is in that position, and `permits` refuses it — so this is unreachable.*
> —— 被允许但没有服务拥有它。只有 `system.authenticate` 处于这个位置，而 `permits` 拒绝了它——所以这里不可达。

如果反过来先查 `destination`，可能会把被拒绝的调用路由出去（见测试 `nothing_refused_is_deliverable`）。

### `refusal` — 拒绝错误

```rust
pub fn refusal(call: &proto::Call) -> proto::Error {
    proto::Error::new(
        proto::code::METHOD_NOT_FOUND,
        format!("{} is not available over WebRTC", call.method()),
    )
}
```

返回 `METHOD_NOT_FOUND` 错误，并在消息中点名具体方法，方便客户端报告哪个调用被拒绝了。

## 测试要点

### `everything_permitted_is_deliverable`

遍历所有 `proto::Call` 变体，凡是 `permits` 返回 true 的，`route_for` 必须返回 `Route::To(..)`。

> *Permission and destination are decided in two places, so permitting a call no service answers became writable.*
> —— 权限和目标在两个地方决定，所以"允许了但没服务应答"是可能写出来的 bug。

### `nothing_refused_is_deliverable`

反过来：凡是被拒绝的，`route_for` 必须返回 `Route::Refused`。

> *This pins the composition order: consulting `destination` before `permits` would pass every other test here and route the whole API to any peer that connected.*
> —— 这钉住了组合顺序：先查 `destination` 再查 `permits`，会让其他所有测试通过却把整个 API 路由给任何连上的 peer。

### `the_pin_and_the_factory_reset_are_never_available`

点名验证 PIN 和恢复出厂永远不可用。不写成"有些东西被拒绝"的组检查，而是逐一列出——因为这是未来放宽权限时不能悄悄扩大的清单。

### `the_control_surface_this_transport_exists_for_is_available`

验证核心控制面（移动、头部、注视、停止、订阅、ToF 流、触控垫输入）全部可达。如果重构不小心拒绝了其中任何一个，说明功能坏了。

### `reaches_padd_and_tofd_which_btd_cannot`

验证 WebRTC 能到达 `Updater`、`Robot`、`Config`、`Pad`、`Tof` 五个服务——这是与 BLE 传输的具体差异：`mediad` 持有五条连接，`btd` 只有三条。

## 与其他模块的关系

```
session.rs
   │ handle()
   ▼
route.rs ──route_for()──▶ route::Route
   │                          │
   │ permits()                │ To(Service, Lane) / Refused
   ▼                          ▼
proto::Call           upstream::Pool (连接各后端服务的 unix socket)
```

- **上游**：`session.rs` 的 `handle()` 调用 `route::route_for()` 决定每个请求的去向。
- **下游**：允许的调用经 `upstream::Pool` 转发到对应服务的 unix socket。
- **兄弟模块**：`btd::route`（BLE 路由表），结构相同但权限集不同。
- **依赖**：`duck_ipc_proto` 提供 `Call`、`Service`、`Lane`、`Error` 等类型定义和 `destination()` 方法。路由表本身不做 I/O，纯函数判定，因此极易测试。
#（注：内容由AI生成）
