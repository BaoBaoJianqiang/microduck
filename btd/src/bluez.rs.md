# `bluez.rs` 文件解析 —— BlueZ 无线电后端（Linux 专用）

## 1. 文件定位

- 路径：`src/bluez.rs`，仅在 `cfg(target_os = "linux")` 下编入；
- 角色：无线电。经 `bluetoothd` 的 D-Bus API 使用 BlueZ，是 BlueZ 与 `session` 两个 channel 之间的全部管道（plumbing）。
- 本文件**不做任何关于机器人的决策**：可能出错的逻辑才是被测试的逻辑，而需要真机无线电的部分集中在此。

## 2. 为什么使用 bluer 的回调（callback）模型

IO 模型曾在硬件上试过，不可行：

- bluer 的 IO 模型只用 `NotSupported` 应答 BlueZ 的 `WriteValue`/`StartNotify`，仅服务 `AcquireWrite`/`AcquireNotify` 的 fd 路径；而 CoreBluetooth 中心设备驱动的是普通方法；
- 结果是机器人能广播、接受连接、接受订阅、接受写，却没有任何一项送达本文件——无 "central connected" 日志、无配对提示、客户端对着一个正常工作的服务超时；
- 当初选 IO 模型是看中 `device_address()`（两半都有），以为配对订阅与会话需要它；但 bluer 对每个特征值只保持**一个** `CharacteristicNotifyState`，本来就只有一个通知会话可配对。"同一时刻一个中心设备"是协议栈属性，不是此处取的巧。

最终形态：服务生命周期内一个会话、一个通知泵、一个把字节推进去的写回调。文件开头明确标注：**尚未对真机硬件验证**（能为 aarch64 通过类型检查，但还没接过真实中心设备），在有手机连入前应把实现当作意图。

## 3. 关键常量

| 常量 | 值 | 含义 |
| ---- | -- | ---- |
| `FLOOR_MTU` | 20 | 出站分片按 20 字节估算。写侧能知道协商 MTU，通知侧无法询问，故取所有 BLE 链路必须支持的载荷，好链路上偏慢但任何链路都正确 |
| `ADV_INTERVAL_MIN/MAX` | 100ms / 150ms | 广播间隔。内核默认 1.28s 曾导致 Mac 连续扫描两分钟平均 7.5s 才看到一次、最长静默 31s；改为 100–150ms（普通外设区间，8–12 倍默认）后实测 0.8s 一次、最差静默 3.8s。不选规范下限 20ms 是为照顾同一根天线上的手柄链路与 wifi；取区间而非定值以抖动规避与邻居持续碰撞 |
| `ADAPTER_RETRY` | 5s | 等待/重试适配器的节奏。板上 `hci0` 在上电约 73s 后才存在（aic8800 UART 挂载晚，bluetooth.service 还被 dbus 阻塞约 26s） |
| `ASK_TIMEOUT` | 5s | 向 configd 询问名字/地址的时限。无人阻塞等待答案（与 PIN 不同），故取值宽松；net.status 需 configd 与 NetworkManager 若干次 D-Bus 往返，扫描中较慢 |
| `ADV_POLL` | 5s | 广播与 configd 状态对账周期 |

`struct AbortOnDrop(JoinHandle)`：随 bring-up 生灭的任务句柄，Drop 时 `abort()`。广播、GATT 应用、agent 都在 drop 时注销，chorale 的无线电任务必须遵守同样规则，否则会对着已消失的适配器永远重连 robotd。

## 4. 顶层服务循环

### 4.1 `serve(sockets, name, require_pairing)`

- 无限循环调用 `serve_on_an_adapter()`；无论其返回 Ok 还是 Err，都按**无线电问题**处理：记 warn（"适配器消失/bring-up 失败，5s 后重试"），睡眠 `ADAPTER_RETRY` 后再来；
- 关键设计：**无线电故障永不离开本函数、不退出进程**。
  - 以前"上电之后"的每一步（加电、注册 agent、广播、发布 GATT 应用）出错都会传播并退出进程，结果是"出现后又异常的适配器"能拖垮 btd，而"从不出现的适配器"反而不会；在没有网络的机器人上，这就是"wifi 不可用"与"完全不可达"的差别；
  - 就地重试还能自愈，不必消耗一次进程死亡，也不怕将来 unit 加上启动次数限制；
  - 副作用：一个 `failed` 状态的 btd 只可能意味着二进制本身坏了，这使其能进入启动恢复网（boot recovery net）。

### 4.2 `serve_on_an_adapter()`（一次完整 bring-up）

1. 克隆 sockets/name，建立 bluer `Session`；
2. **等待适配器出现**：单独一个内层循环，把"还没有适配器"（开机前 73 秒的常态，记为进展）与"此后失败"（故障）区分开，避免正常开机时每 5s 刷一条故障日志；
3. `set_powered(true)`；
4. 若要求配对：`set_pairable(true)`。板子默认 Pairable=no；保持常开而非限时窗口，因为每机 PIN 已承载窗口所增加的属性（见 `pairing.rs`）；
5. 注册 **just-works agent**（`request_default: true`，其余 `None` → BlueZ 视为 NoInputNoOutput）。注释记录了 passkey 方案在无键盘机器人上的失败史，PIN 校验收此上移到会话层；不要求配对时不注册 agent 并高声 warn；
6. 记录 "serving BLE" 日志：适配器名、蓝牙地址（字段名 `bd_addr` 以区别广播里的 IPv4 `address`）、服务 UUID、是否配对、`max_adv_len`（adv 预算所依赖的控制器上报值，地址缺失时首先应看它）；
7. 首次广播前先问名字与地址（地址先问是为避免已入网机器人开机头几秒广播 `0.0.0.0`，被误判为无 wifi）；`--name` 钉住时直接用 pinned；
8. `advertise(...)` 取得广播句柄；
9. 派生 **chorale 无线电任务**（独立连接 robotd、独立广播实例），失败仅 debug，5s 重试；用 `AbortOnDrop` 保证 bring-up 结束即中止；
10. 构造 **GATT 应用**（见第 5 节）并 `serve_gatt_application`；
11. 进入 `tokio::select!`：`watch_adapter()` 返回 或 `reconcile_advertisement()` 返回（后者自身永不返回）即结束本次 bring-up，所有句柄 drop 注销，交还调用方进入下一轮。

## 5. GATT 应用：单会话、读/写/通知

维护 `current: Arc<StdMutex<Option<Sender<Vec<u8>>>>>`（当前订阅会话的入站发送端）。

- 刻意用 **std 的 Mutex 而非 tokio 的**：写回调必须在不 await 的情况下读取它，任何让出点都可能让两片交换位置；没有任何东西跨 await 持锁。

### 5.1 读特征值（version read）

- `encrypt_read = require_pairing`；唯一作用是在任何写入之前**强制一次绑定**（读会被应答，未配对中心收到"认证不足"随即开始配对；订阅不带任何加密标志，做不到）；
- 返回 `API_VERSION as u8`，让客户端在写入前就能识别版本不符；
- 注释标注当前实践中这其实是*非加密*路径（§5.5，强制加密会挂起 CoreBluetooth）。

### 5.2 写特征值

- `write_without_response = true`（分片请求无需每片 ATT 应答），`encrypt_write = require_pairing`；
- 收到值到入队之间**无 await**（防分片乱序，同 `main` 单线程理由）；记日志只取前 8 字节（可能含 wifi 口令，需截断）；
- 无当前订阅者 → 拒绝（`GattError::Failed`）：客户端应先订阅，无处送答案还接受写入就是撒谎；
- 队列满 → 拒绝（客户端会重发；丢片不可恢复）；
- 通道关闭 → 拒绝（会话已结束）。

### 5.3 通知回调（订阅生命周期）

- 每次订阅新建全新 `Link::pair(FLOOR_MTU, "central")` 与全新 `session::run` 任务，杜绝上一个中心设备的残片泄漏；
- bluer 每特征值只有一个通知态：新订阅**替换**旧会话（而非共享，否则两个客户端共用重组缓冲会交错请求），记 warn；
- 泵循环用 `biased` select：先看 `notifier.stopped()`（中心设备离开），再从 `outbound` 取分片 `notify()`；
  - `stopped()` 很关键，否则空闲中断开的客户端只有在下次"无人可发的回复"通知失败时才被发现，会一直占着槽位；
- 收尾时**仅当槽位仍属于自己**才清空：通知一个已消失的中心可能比其订阅活得久，此时重连的新中心或许已装上更新的会话，盲目 `take()` 会误杀新会话。比较用 `same_channel`；
- 丢弃发送端即结束会话任务，释放重组缓冲与上游连接。

## 6. 广播与对账

### 6.1 `struct Advertised { name, address: Option<Ipv4Addr> }`

- 派生 PartialEq/Eq，用一个结构体而非两个参数，使"是否有变化"成为一次比较；`adv` 模块已解释没有第三字段的空间；
- 实现 `Display`：`"{name} at {address}"` 或 `"{name} with no address"`，供日志表达"什么变了"。

### 6.2 `advertise(adapter, advertised)`

同时设置**两个名字**：

- 广播里的 Local Name，以及适配器的 GAP Device Name（`0x2A00`，BlueZ 取自 `Adapter.Alias`，默认是主机名）；
- 历史 bug：改名后的机器人广播 `duck-5b21`，读 GAP 名却答 `radxa-zero3`；BlueZ 会在连接后缓存 GAP 名（Linux 上连接前后两个名字、`duckctl --name` 找不到），CoreBluetooth 拼接成 `radxa-zero3 [duck-5b21]`，而手机蓝牙设置显示 GAP 名（最重要却仓库里看不到的场景）；
- 因此设 alias 是"命名机器人"的一部分，所有重新广播路径都经过本函数；alias 已正确则跳过写（它持久保存在 BlueZ 状态中）；设置失败只记 warn 不传播——alias 不如"能被看见"重要。
- 广播内容：服务 UUID、manufacturer data（地址）、discoverable、local_name、100–150ms 间隔；
- **地址字段失败时降级而非整体失败**：先尝试带地址注册，若 BlueZ 拒绝（是否超 31 字节最终由控制器计数），再退回不带地址注册。在 BLE 可能是唯一前门的机器人上，"没有地址但能连上"远好于"整个被拒绝、彻底失联"。

### 6.3 `reconcile_advertisement(...)`（永不返回）

- 每 `ADV_POLL` 询问一次当前名字（pinned 时不问）与地址，组成新的 `Advertised`；
- **刻意轮询而非事件驱动**：btd 转发 `system.setName` 时不读回复（避免去解释客户端的应答），无法靠观察得知改名；转发后立刻再问也不保证写已应用、两次连接无顺序保证。轮询零件更少，还能覆盖不经本进程的改名路径（`robotctl system set-name`）；代价只是几秒一次的本地 socket 一问，远低于噪声；
- 名字与地址在**同一拍**以相同节奏询问，比 DHCP 租约变化快得多；分开节奏会让 `wifi connect` 后的地址滞后半分钟，而那一刻地址正是操作者最想读到的东西；
- 无变化且句柄健在则跳过；否则**先注销旧广播再注册新广播**（避免同时持两个导致 BlueZ 拒第二个）；重新注册失败留给下一拍，绝不传播（处于"无线电故障不杀进程"的 bring-up 之内）；
- `--name` 只抑制"问名字"，不抑制循环——钉住名字不等于钉住 DHCP 租约。

### 6.4 `ask_name()` / `ask_address()`

- 都向 configd 发一次性 `ask()`（`SystemInfo` / `NetStatus`），超时 `ASK_TIMEOUT`；
- 失败级别用 `debug` 而非 `warn`：每几秒跑一次，configd 重启时 warn 会刷屏，而启动那次的结果由调用方通过实际广播名记录；
- `ask_name` 失败回退到给定 fallback；
- `ask_address` 严格区分两种失败：configd 明确报告无地址（未连 wifi）→ 清空字段；configd 不应答（重启/NetworkManager 超时）→ **保留上一次地址**。二者曾被混为一谈，导致每次 tick 注销/重注广播、地址在客户端眼前闪烁；
- 地址字符串先 `parse::<Ipv4Addr>()` 再广播，不信任 NetworkManager 给的原始串；
- 只取 IPv4——只有 IPv4 放得下。

### 6.5 `watch_adapter()`

- 轮询 `is_powered()`（而非依赖事件流）：要抓的故障比"适配器被移除"更广——总线上却不应答的僵死适配器正是以前会杀进程的情况，读一个属性可同时覆盖两类，且不依赖 BlueZ 对"僵死"发什么事件；
- `Ok(true)` 继续；`Ok(false)`（被 power off、驱动重置、挂起）或出错都返回，交由下一轮 bring-up 重新加电。在 BLE 可能是唯一前门的机器人上，不主动"礼貌地"保留关机态。

## 7. 本文件要点小结

1. 只做 BlueZ↔session channel 的管道，不含机器人决策；采用 bluer 回调模型（IO 模型硬件验证失败）；
2. `serve` 无限就地重试，无线电故障永不杀进程；适配器出现前后分两段日志；
3. 广播间隔 100–150ms 是实测得来的可发现性关键；MTU 出站按 20 字节保底；
4. 每次订阅一个全新会话；std Mutex 无 await 入队防乱序；biased select 先察觉离开；清槽前确认归属；
5. 读特征值用于强制绑定与版本握手；写无订阅/队列满/关闭均显式拒绝；
6. 同时设 Local Name 与适配器 Alias；地址注册失败降级为无地址广播；
7. 名字/地址每 5s 轮询对账；configd 无应答保留旧值，明确无地址才清空；watch 轮询加电状态。
