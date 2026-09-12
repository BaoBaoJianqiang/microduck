# `route.rs` 文件解析 —— BLE 可发起哪些调用、走哪条连接

## 1. 文件定位

- 路径：`src/route.rs`
- 角色：BLE 传输的**权限边界 + 路由表**。决定一个 JSON-RPC 调用是否允许从 BLE 进入，以及允许时转发给哪个上游服务、占用哪条 lane（连接）。

## 2. 两个被刻意分开的问题

文档强调这里回答两个问题，且只有第二个属于 BLE：

1. **哪个服务应答一次调用、应答会占用连接多久**：这是调用本身的属性，定义在协议 crate 的 `proto::Call::destination()` 中，所有传输读到同一份答案；
2. **BLE 是否允许发起它**：这是本文件的职责。

两者曾合并为一张表，直到第二种传输（WebRTC，见 `docs/design/remote-webrtc.md` §5）只需要第一半、完全不需要第二半，才拆开。

`permits()` 的 match **刻意穷尽（exhaustive）而不用 `_ => false` 通配**：给 `proto::Call` 新增变体时本文件会编译失败，从而新方法不可能因为"有人忘了这个文件"而漏到无线电上。通配当下安全、长期却是错的（会静默拒绝，首个症状是手机 App 看不到某功能）。每种传输都需要自己的这样一份 match。

## 3. 类型定义

### 3.1 `enum Upstream`

`btd` 持有 socket 的三个上游：`Updater`（updaterd）、`Robot`（robotd）、`Config`（configd）。

- 故意比 `proto::Service` 窄：`padd` 和 `tofd` 也应答调用，但 `btd` 与二者都没有连接；
- `padd` 是无特权客户端，其全部价值就在于没有任何特殊访问权，给 BLE 一条通向它的 socket 会首先破坏这一点；
- `TryFrom<proto::Service>` 对 `Pad | Tof` 返回 `Err(())`，把注释变成编译器强制约束。

### 3.2 `enum Route`

一次从 BLE 到达的调用的处置：

- `To(Upstream, Lane)`：逐字转发给某服务的某条 lane 连接；
- `Local`：由 `btd` 自己应答。仅 `system.authenticate`——PIN 校验属于传输层（BLE 无法表达固定印刷 passkey）；
- `Refused`：本传输不可用。

### 3.3 `pub use proto::Lane`

一次调用占用连接的时长类型，定义在协议 crate，此处再导出。

## 4. 权限表 `fn permits(call) -> bool`

应把 `false` 分支读作安全边界。分类摘要：

**允许（true）：**

- `Hello`：版本握手，不允许就无法建立任何会话；
- `SystemAuthenticate`：唯一必须在其他调用之前就能发起的调用，由本地应答；
- 更新子集：`Apply`、`Check`、`Status`、`Subscribe`、`Log`、`Show`、`ListInstalled`；
- 回退类：`Rollback`、`Select`（移动到已在本板运行过的版本，不下载、不丢弃，经门控并可自动回退）；
- `RobotHealth`：App 唯一用得上的 `robot.*` 调用；
- 配网四件套：`NetStatus`、`NetScan`、`NetConnect`（携带 wifi 口令，要求配对加密链路）、`NetForget`；
- 名字与身份：`SystemInfo`、`SystemSetName`；
- `SystemServices`：各守护进程状态与版本；
- `SystemReboot`：激烈但可恢复，不丢弃任何东西（不同于 resetToGolden）；
- 手柄：`PadStatus`、`PadPair`、`PadForget`（由 configd 完成实际工作）。

关于 `Apply` 的注释还指出：它除了被 BLE 允许，还必须通过 `updaterd` 自身的对等策略——`deploy/updater.toml` 在 `allow_users` 中点名 `btd`，比直接授予 robot 组更窄。

**拒绝（false，均为有意决定）：**

- `SystemPairingPin | SystemSetPairingPin`：PIN 绝不能经无线电被读/写，这是承重性拒绝（否则配对机制沦为摆设）；`btd` 自己经 unix socket 读取；
- `Pin`：版本钉选。误操作的后果看起来与正确行为完全一样（拒绝后续更新却自称最新），需要 `robotctl` 与明确意图；
- `ResetToGolden`：等同恢复出厂，丢弃其后所有版本，绝不能走无线电；
- `RobotSafeToRestart | RobotModelApi | RobotRemoteSessionActive`：updaterd 对 robotd 的内部问询，对客户端无用且有误导；
- 全部电机控制：`RobotMove/Head/Look/Enable/Do/Pose/Mouth`。BLE 太慢太受限（20 字节通知预算、开机约 73 秒链路不存在），遥操作属于 WebRTC data channel；
- `RobotSound | RobotTheremin | RobotChorale`：在有 App 路径真正需要之前不开放，避免无谓扩大攻击面；
- `ChoraleSubscribe | ChoraleBeaconSet | ChoraleHeard`：btd 与 robotd 之间的内部命名空间，不是客户端接口；
- `RobotShutdown`：关机（不回来的 reboot）有意留给 `robotctl` 或手柄长按；
- `RobotMode`、`RobotSetMode`：模式切换意味着回家、换策略，需要在场持手柄；
- `RobotInit | RobotRelax`：上电解锁/卸力，需要人看着机器人，留给本机 `robotctl`；
- `RobotStop`：值得单列——一个在无绑定、高延迟、时断时续无线电上"看起来像急停却不是急停"的按钮比没有更糟；`robotd` 的死人开关（deadman）在意图停止到达时已会停机器人，真正的急停是物理的；
- `RobotSubscribe`：高速遥测，等于把消防水龙接进 20 字节管道，得到无法推理的抽样/滞后视图；
- `PadInput`：手柄每秒上百次 evdev 事件，且由 `padd`（btd 不持其 socket）应答；
- `TofStream`：深度帧同样是高速流，且由 `tofd`（不持 socket）应答；将来应走 `mediad` 视频路径。

## 5. 路由决策函数

- `destination_for(call) -> Option<(Upstream, Lane)>`：先查 `permits`，再调协议 crate 的 `call.destination()`，再用 `TryFrom` 收窄到三个上游。先权限、后目的，顺序至关重要；
- `upstream_for(call) -> Option<Upstream>`：只返回上游服务，供多数调用方与安全边界测试使用；
- `route_for(call) -> Route`：完整决策；`SystemAuthenticate` 特判为 `Local`，其余按 `destination_for` 决定 `To` 或 `Refused`；
- `refusal(call) -> proto::Error`：拒绝时返回 **PERMISSION_DENIED** 而非 METHOD_NOT_FOUND——对持手机者含义不同：方法存在但本传输不能用（"去用 robotctl"），而不是"升级你的 App"。错误消息会点名方法，例如 "… is not available over Bluetooth; use robotctl on the robot"。

## 6. 单元测试（约 13 个，安全边界的主要保障）

| 测试 | 要点 |
| ---- | ---- |
| `only_these_mutating_calls_are_reachable_over_ble` | 逐一列名允许的变更类调用白名单（apply/rollback/select/net.connect/net.forget/system.setName/system.reboot/pad.pair/pad.forget） |
| `a_pad_can_be_paired_from_the_phone` | pad 三个调用都路由到 `Config`，且 btd 不自己应答 |
| `the_pairing_pin_is_not_reachable_over_ble` | PIN 的读、写均不可从 BLE 到达 |
| `provisioning_reaches_configd` | 配网/名字/重启均到达 `Config` |
| `the_refused_calls_stay_refused` | 逐一钉死被拒调用（resetToGolden、pin、三个内部问询） |
| `the_app_path_is_reachable` | hello/status/subscribe/robot.health 路径可达，且目的地正确（Updater/Updater/Updater/Robot） |
| `a_refusal_says_permission_denied_and_names_the_method` | 拒绝码为 PERMISSION_DENIED 且消息含方法名 |
| `an_apply_shares_its_connection_with_nothing_a_client_does_during_one` | 更新进行中的 status/subscribe/check 与 apply 同服务但**不同 lane**，不会排队其后 |
| `nothing_else_travels_on_the_stream_lane` | 只有 `Subscribe` 占用 Stream lane（该连接一旦交出永不再读请求，第二条调用会丢失而非延迟） |
| `the_calls_that_take_their_time_are_off_the_prompt_lane` | apply/rollback/select/check/netScan/netConnect/padPair 均不在 Prompt lane |
| `going_back_is_reachable_from_the_phone` | rollback/select 可达且走 `(Updater, Operation)` |
| `everything_permitted_is_deliverable` | 允许的调用必须是 btd 真正能投递的（持其 socket）；`system.authenticate` 是唯一 `Local` 例外。防止权限表与目的地表分离后"允许了却送不到" |
| `nothing_refused_is_deliverable` | 逆否：被拒的绝不被投递；钉死组合顺序，防止先查目的地导致整个 API 漏到无线电 |

测试用 `duck_ipc_proto::test_support::every_call()` 遍历协议中**每一个**调用（共享清单而非本地复制——曾经两份清单漂移到 115 行对 82 行，导致 `pad.input` 在其中一份缺失）。

## 7. 本文件要点小结

1. 一份穷尽式 match 同时承担权限边界与路由，新增协议方法会编译失败强制复核；
2. "谁应答/占多久连接"归协议 crate 共享，"BLE 能否调用"归本文件；
3. `Upstream` 用类型系统把可达服务收窄到 btd 持有的三个 socket；
4. 允许面：版本、认证、更新子集与回退、配网、名字、重启、手柄配对；明确拒绝电机控制、出厂复位、PIN 读写、高速流等；
5. lane 机制保证长操作（apply/stream）不阻塞并发的状态轮询；
6. 拒绝返回 PERMISSION_DENIED 并点名方法；大量属性化测试用 `every_call()` 守住边界与可投递性。
