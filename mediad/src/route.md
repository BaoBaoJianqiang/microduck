# route.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\route.rs`

## 核心设计决策

WebRTC 对等端可以发起哪些调用。是 `btd::route` 的兄弟，刻意同构："哪个服务应答一个调用、应答会占用连接多久"只住在 `proto::Call::destination` 一次，本文件只回答"这个传输上的对等端可不可以问它"。

**匹配是穷举的**，这正是每个传输各有一份的意义：给 `proto::Call` 加一个变体，在此处和 `btd` 同时编译失败，新方法不会因没人记得这个文件而漏到远程对等端。共享表加 `_` 通配会在两个传输上同时开洞。

### 为什么此子集比 BLE 宽

BLE 窄的两个原因在此不适用：无线电慢（20 字节通知预算）、几米内任何人都能说话。数据通道既不慢也不限房间，所以 BLE 因容量拒绝的调用正是本传输存在的目的：intents、telemetry、pad 输入、深度流。

### 被拒绝的三类

1. **授权另一传输**：配对 PIN。能改写它的对等端可把手机锁出 BLE（恢复路径）。
2. **会丢掉发起它的会话**：更新变更、重配 wifi。更新是延迟而非禁止（`remote-webrtc.md` §8 列了所需：客户端重连重订阅、`RobotRemoteSessionActive` 区分旁观者与请求者）。
3. **不是客户端的问题**：`updaterd` 对 `robotd` 的内部查询。

## 类型与函数

### `Route`
`To(Service, Lane)` 或 `Refused`。

### `permits(call) -> bool`
穷举匹配，每个 `false` 分支是一个"持有会话的对等端不得做此事"的决策，并说明属于三类中的哪类。

要点：
- 控制面（`robot.move/head/look/pose/mouth/do/sound/theremin/stop/enable/init/relax/shutdown`、telemetry、`pad.input`、`tof.stream`）全部允许——这是本传输存在的目的。
- `robot.stop` 在此允许而 BLE 拒绝：区别是诚实而非权限。BLE 的理由是"未配对、高延迟、时有时无的无线电上的停止按钮看起来像急停但不是"；`control` 通道可靠有序，且 deadman 已在 intent 停止时停机器人。
- `robot.set_mode` 拒绝：模式切换声称硬件变化（"这只鸭子现在有轮子"），只有房间内的人能断言，pad 才是它的归属。
- `chorale` 全部拒绝：合唱是机器人之间通过 BLE 的事，浏览器既不在房间也不是鸭子。
- `system.authenticate` 拒绝：本传输不认证，回答就是撒谎。

### `route_for(call) -> Route`
不允许直接 `Refused`；否则按 `destination()` 转发。`destination()` 返回 `None` 的（`system.authenticate`）也 `Refused`。

### `refusal(call) -> Error`
`METHOD_NOT_FOUND` + "method is not available over WebRTC"。

## 单元测试

| 测试 | 意图 |
|---|---|
| `everything_permitted_is_deliverable` | 允许的调用必须能路由到某服务（与 `btd` 同理：许可与目的地分两处决定） |
| `nothing_refused_is_deliverable` | 被拒绝的不能到达任何服务；固定组合顺序（先 `permits` 后 `destination`） |
| `the_pin_and_the_factory_reset_are_never_available` | 四个绝不允许网络传输的调用逐个点名 |
| `the_control_surface_this_transport_exists_for_is_available` | 本传输存在目的的控制面调用必须可用 |
| `reaches_padd_and_tofd_which_btd_cannot` | WebRTC 可达 BLE 故意不持有的 `padd`/`tofd` |

## 关键摘要

- 穷举匹配，新方法漏不掉。
- 控制面/telemetry/pad/depth 全开放（本传输的目的）。
- PIN、factory reset、pin、mode、chorale、update 变更、wifi 重配、gamepad 配对被拒。
- 不认证：`system.authenticate` 直接拒绝而非撒谎。
