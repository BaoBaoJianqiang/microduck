# `route.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 659 行 |
| 角色 | BLE 可以进行哪些调用，以及服务的哪个连接承载它们——方法白名单与路由表 |
| 平台 | 跨平台（纯逻辑） |

## 二、两个问题，只有一个是 BLE 的

BLE 暴露机器人 API 的**子集**（architecture.md §4.1）：配置、状态、更新命令及其进度。

- *哪个服务回答一个调用，以及回答持有连接多长时间*是调用的属性，住在 `proto::Call::destination`，每个传输读取相同的答案。
- *BLE 是否可以进行它*是此文件。
- 它们曾经是一个表，直到第二个传输需要前一半而不需要后一半；`docs/design/remote-webrtc.md` §5 记录了拆分。

## 三、权限匹配是故意穷举的

- 向 `proto::Call` 添加变体使此文件无法编译，因此新方法不能因为有人忘记此文件存在而到达无线电。
- `_ => false` 通配符在当时是安全默认值，随着时间推移是错误的：它会静默拒绝新方法，第一个症状是手机应用看不到没人记得路由的功能。
- 每个传输都需要自己的这样的匹配，原因相同——带通配符的共享匹配会同时是所有传输中的洞。

## 四、`Upstream` 枚举

```rust
pub enum Upstream {
    Updater,  // updaterd
    Robot,    // robotd
    Config,   // configd — wifi 和机器人身份
}
```

- 比 `proto::Service` 窄，故意。
- `padd` 和 `tofd` 也回答调用，但 `btd` 与两者都没有连接：
  - `padd` 是非特权客户端，其全部价值是没有特殊访问，给 BLE 传输到它的 socket 会是使这不真实的第一件事。
  - 因此转换对它们*失败*，这把注释变成编译器强制执行的东西。

### `TryFrom<proto::Service> for Upstream`

- `Updater`/`Robot`/`Config` → `Ok`
- `Pad`/`Tof` → `Err(())`（实践中不可达，因为 `permits` 拒绝这些回答的每个调用；错误而非 panic 以便保持真实不需要此文件小心）

## 五、`Route` 枚举

```rust
pub enum Route {
    To(Upstream, Lane),  // 原样转发到服务，在该服务对此 lane 的连接上
    Local,                // 由 btd 自己回答。只有 system.authenticate
    Refused,              // 此传输不可用
}
```

- `Local` 只有 `system.authenticate`：PIN 检查属于传输，因为 BLE 无法表达固定打印的 passkey，检查因此必须上移一层（`docs/design/app-path-design.md` §5）。

## 六、`permits(call) -> bool`——安全边界

读取 `false` 分支作为安全边界：每个都是手机在房间里不被允许做这件事的故意决定。

### 允许的调用（true）

| 类别 | 调用 | 说明 |
|---|---|---|
| 握手 | `Hello` | 必须可达否则没有客户端能建立任何东西 |
| 认证 | `SystemAuthenticate` | 由 btd 自己回答，会话在进行任何其他调用之前必须能进行的一个调用 |
| 更新子集 | `Apply`, `Check`, `Status`, `Subscribe`, `Log`, `Show`, `ListInstalled` | §4.1 命名的更新子集。`Apply` 是故意的：BLE 意味着物理存在 + 配对，"从手机更新机器人"是 M6 的头条。必须通过 `updaterd` 自己的对等策略，确实通过——`deploy/updater.toml` 在 `allow_users` 中命名 `btd` |
| 回退 | `Rollback`, `Select` | 两者都允许，都不如上面的 `Apply` 后果严重：它们将机器人移动到已经在此板上运行过的版本，不下载任何东西，像任何其他转换一样被门控和自动回退。曾经被拒绝，直到更新路径从手机驱动——引擎自己回退失败健康门的版本，但不是安装了、通过了门、然后表现*更差*的版本 |
| 机器人健康 | `RobotHealth` | 应用有任何用处的一个 `robot.*` 调用 |
| 配置 | `NetStatus`, `NetScan`, `NetConnect`, `NetForget` | 整个传输存在的原因：从未见过网络的机器人无法通过该网络配置，因此 BLE 是唯一入口。`NetConnect` 携带 wifi 密码，§7 要求它在配对的加密链接上旅行——确实如此：characteristic 设置 `encrypt_authenticated_write`，PIN agent 使绑定成为认证的 |
| 身份 | `SystemInfo`, `SystemSetName` | 从应用重命名是 `system.setName` 存在的原因 |
| 服务 | `SystemServices` | 哪些守护进程在运行，每个运行什么版本 |
| 重启 | `SystemReboot` | 剧烈但可恢复，机器人困惑时应用提供的东西——替代是"拔电源"，对行走机器人更糟。不像 `resetToGolden` 它不丢弃任何东西 |
| 游戏手柄 | `PadStatus`, `PadPair`, `PadForget` | 从手机配对控制器，它属于那里：拿着机器人的人拿着手柄，替代是 ssh 会话。`pad.pair` 后果更严重，因为绑定的手柄之后可以启用策略——故意的：与站在机器人旁边拿着控制器相同的权限 |

### 拒绝的调用（false）

| 调用 | 原因 |
|---|---|
| `SystemPairingPin`, `SystemSetPairingPin` | **承重的拒绝**：未配对对等方可读的 PIN 什么都不授权。btd 通过 unix socket 读取它来回答 BlueZ 的 passkey 请求，BLE 永远不能 |
| `Pin` | 固定版本，保持拒绝而 `Select` 不。区别是错误之后的样子：错误的 `select` 离撤销一个版本，机器人说它在哪个版本上；而被误触固定的机器人拒绝每个后续更新并报告自己是最新的。这是这里唯一一个看起来完全像正确行为的失败 |
| `ResetToGolden` | 除了名字之外的工厂重置：回到黄金镜像，丢弃之后的每个版本。永远不通过无线电——`Rollback` 和 `Select` 被路由不会削弱此，因为两者都不丢弃任何东西 |
| `RobotSafeToRestart`, `RobotModelApi`, `RobotRemoteSessionActive` | `updaterd` 对 `robotd` 的私有问题——更新决定的内部管道，对客户端无用且暴露会误导 |
| `RobotMove`, `RobotHead`, `RobotLook`, `RobotEnable`, `RobotDo`, `RobotPose`, `RobotMouth` | **电机控制。永远不通过 BLE**——§4.1 子集的意思：BLE 太慢太受限，遥操作属于 WebRTC 的 datachannel。20 字节通知预算和启动前 ~73s 不存在的链接不是控制传输 |
| `RobotSound`, `RobotTheremin`, `RobotChorale` | 从手机看无害且相当迷人——但它与其余 `robot.*` 乘坐相同的拒绝，直到应用路径存在想要它 |
| `ChoraleSubscribe`, `ChoraleBeaconSet`, `ChoraleHeard` | 合唱自己的命名空间在 `btd` 和 `robotd` 之间——不是客户端表面 |
| `RobotShutdown` | 从房间里的手机关机是没有回来的 `system.reboot`。坐下然后关机的流程想要问的人看着机器人，那是 `robotctl` 或手柄的长按，故意 |
| `RobotMode` | 仅对 `padd` 等本地客户端的摇杆映射提示 |
| `RobotSetMode` | 切换模式意味着机器人回家、加载其他策略、不同驾驶——原因是有人刚给它装了轮子。那是在房间里、拿着手柄做的决定 |
| `RobotInit`, `RobotRelax` | 关节电源。把机器人掉在地板上的手机按钮不是要提供的，`robot.init` 是其对应：让机器人站起来同时移动每个关节，想要做的人看着机器人而非屏幕 |
| `RobotStop` | **值得单独一行**，因为拒绝它看起来错。应用中的紧急停止正是某人伸手要的，但在未绑定、高延迟、有时缺席的无线电上工作的停止按钮比没有按钮更糟，因为它*看起来*像 e-stop 而不是。`robotd` 中的 deadman 已经在意图停止到达时停止机器人，这是不依赖手机在范围内的机制。真正的 e-stop 是物理的 |
| `RobotSubscribe` | 高速率遥测。`robot.subscribe` 以控制速率流状态；通过 BLE 那是进入 20 字节管道的消防水带，客户端会得到无法推理的抽取、不可预测滞后的视图 |
| `PadInput` | 与 `robot.subscribe` 相同的反对，只有更甚：这是手柄发送的每个 evdev 事件，每秒超过一百个报告，它存在是为了*测量其自己传递的节奏*。也不是 `btd` 转发的：`padd` 故意不是 `btd` 持有的 socket 之一 |
| `TofStream` | 深度帧，与手柄点击相同的两个反对。64 区域帧每秒 15 次是进入 20 字节管道的消防水带；由 `tofd` 服务，不是 `btd` 持有的 socket。当手机有理由看机器人看到的东西时，将通过 `mediad` 的视频路径 |

## 七、路由函数

### `destination_for(call) -> Option<(Upstream, Lane)>`

- 这个调用去哪里、在哪个连接上，或 `None` 如果 BLE 不可以进行它——或者如果没有服务回答它（对 `system.authenticate` 是相同答案不同原因）。
- 先检查 `permits`，再查 `call.destination()`，再转换 `Upstream`。

### `upstream_for(call) -> Option<Upstream>`

- 回答调用的服务，忽略哪个连接承载它。
- 权限问题本身，大多数调用者和每个关于安全边界的测试问的。

### `route_for(call) -> Route`

- 完整路由决定，包括传输自己回答的一个调用。
- `SystemAuthenticate` → `Local`
- 其他 → `destination_for` 的结果或 `Refused`

### `refusal(call) -> proto::Error`

- 回答被拒绝调用的 JSON-RPC 错误。
- `PERMISSION_DENIED` 而非 `METHOD_NOT_FOUND`，因为两者对拿着手机的人意味着不同的东西：此方法存在且此传输不可以使用它——"试试 `robotctl`"，而非"升级你的应用"。

## 八、测试（12 个）

| 测试 | 验证内容 |
|---|---|
| `only_these_mutating_calls_are_reachable_over_ble` | 精确命名 BLE 可以进行的变更调用：`update.apply`, `update.rollback`, `update.select`, `net.connect`, `net.forget`, `system.setName`, `system.reboot`, `pad.pair`, `pad.forget` |
| `a_pad_can_be_paired_from_the_phone` | 手柄配对从手机到达 `configd`，btd 不能自己回答（它什么都不拥有） |
| `the_pairing_pin_is_not_reachable_over_ble` | PIN 永远不可通过无线电读或写——这是这里唯一一个不仅仅是谨慎的拒绝 |
| `provisioning_reaches_configd` | 配置必须可达且到达 `configd`——BLE 存在的原因 |
| `the_refused_calls_stay_refused` | 被拒绝的调用保持被拒绝，单独命名。如果未来变更使其中一个可达，应该必须在这里删除一行并在提交中说明原因 |
| `the_app_path_is_reachable` | 手机必须能建立会话、看机器人状态、开始更新并观看。没有全部四个传输对它存在的目的就没用 |
| `a_refusal_says_permission_denied_and_names_the_method` | 拒绝必须可与"没有这样的方法"区分 |
| `an_apply_shares_its_connection_with_nothing_a_client_does_during_one` | 更新期间手机做的任何东西都不能与更新共享连接——lane 存在的缺陷 |
| `nothing_else_travels_on_the_stream_lane` | 进度流必须单独在其 lane 上——比上面更强的主张：交给 `stream_progress` 的连接*永远*不读进一步请求，因此共享它的第二个调用不是延迟而是丢失 |
| `the_calls_that_take_their_time_are_off_the_prompt_lane` | 花时间的调用永远不在快速回答使用的 lane 上 |
| `going_back_is_reachable_from_the_phone` | 回退可达且到达 `updaterd` |
| `everything_permitted_is_deliverable` | 每个被允许的调用必须是 btd 实际能交付的——拆分使此测试必要。`system.authenticate` 是故意的例外 |
| `nothing_refused_is_deliverable` | 被拒绝的调用必须不可交付，无论共享表怎么说。固定组合顺序：`destination_for` 在权限检查*之前*查共享目标会通过此文件中的每个其他测试并静默将整个 API 路由到无线电 |
#（注：内容由AI生成）
