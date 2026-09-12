# bluez.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/bluez.rs`
- **角色**：`Pads` trait 的生产后端，经 BlueZ 的 D-Bus API 完成**游戏手柄配对**（仅 Linux）。这是全 crate 注释最密集的文件，记录了配对顺序、异步绑定、agent 角色、板级设置与传输验证边界等大量真机经验。
- 选型：用 `zbus` 而非 `btd` 使用的 `bluer`——`bluer` 链 libdbus（vendored C、用 `cc` 构建），而本 crate 为 NetworkManager 已有一套纯 Rust D-Bus 栈；为四个方法调用再引入一个 C 依赖不划算。

## 2. 模块文档中的关键知识

### 2.1 配对顺序：尝试而非强制，以状态而非返回值为准

- 顺序是 **`connect` → `pair` → `trust`**。Xbox 手柄上先调 `Pair()` 会得到 `AuthenticationCanceled`。
- BlueZ 的返回值不描述实际发生了什么：
  - 对从未绑定的设备调 `Connect()` 可能返回 `br-connection-profile-unavailable`（还没有 profile 可连）。若就此拒绝，会错杀一个 `Pair()` 稍后就能绑定的手柄，故**软失败**。`br-` 前缀是 BlueZ 先尝试 BR/EDR 扑空，而该手柄走 LE 绑定。
  - 对*会*绑定的手柄，`Connect()` **在绑定完成前就返回**：HID profile 需要加密链路，连接触发绑定，绑定稍后才落地。
  - 对在途绑定调 `Pair()` **永远不应答**（不是 `AlreadyExists`，而是挂到超时）。
- 后两点组合出出厂时的 bug：`Connect()` 后立刻读 `Paired` 看到 false，于是调 `Pair()`，苦等 30 秒不应答，在手柄其实已绑定的情况下返回超时，且永远没走到 `set_trusted`。表现为「第一次配对超时、第二次秒成」。
- 结论：**`Paired` 变 true 是「成功」的唯一真相**。`Connect()` 获得 `BOND_SETTLE` 时间自行产生绑定；随后的 `Pair()` 与同一属性赛跑，而不是相信其返回值。
- **连接前先停发现**：活动扫描中 `Connect()` 会被 BlueZ 接受但间歇性失败，表现为「第二次才配上」的疑似硬件不稳。

### 2.2 Agent：为什么必须在配对窗口内申请 default 角色

- 配对需要 agent（回答「是否允许」）。`btd` 已为手机路径注册了一个**默认** agent；本文件再注册一个作用域限于目标手柄的 agent，并在配对窗口内**取得 default 角色**。
- 取角色不可选，两个原因叠加：① bluetoothd 只从默认 agent 向适配器下发 IO 能力，非默认的 `NoInputNoOutput` 会让适配器继续宣称有输入和显示，从而在配对请求中加入 MITM、使 SMP 选择数字比较而非 just-works；② bluetoothd 偏好发起 `Pair()` 的连接所属 agent，而在实际可行路径上 `Connect()` 自行完成绑定、`Pair()` 回退根本不执行，于是确认请求被抛给默认 agent——没抢到角色的 `configd` 永远看不到它，手柄等过链路监督超时后得到 `AuthenticationCanceled`。
- `unregister_agent` 会归还角色；`btd` 只在确实需要配对时持有它。agent 还**限定单一设备路径**，开窗期间来自无关设备的请求一律拒绝。手柄是 just-works，没有 passkey 可核对，「因人类要求、在这几秒内、仅接受这一台设备」就是授权的全部。

### 2.3 板级设置 `Privacy=device`

- `/etc/bluetooth/main.conf` 的 `Privacy` 由 `scripts/setup-board.sh` 设为 `device`。BlueZ 默认 `off`，在部分 Radxa Zero 3W 上可行、另一部分上手柄完全无法绑定；两类板子无任何可测量差异，故统一设 `device`。
- `device` 下，**`btd` 正在广播时无法形成新绑定**（已有绑定不受影响），因此 `robotctl pad pair` 会在配对窗口停掉 `btd`（见 robotctl 的 `BtdPaused` 与 `docs/project/pad-minimal-pairing.md`）。
- 易被误判为本文件之过的现象：SMP `DHKey check failed (0x0b)`。该校验基于双方地址计算，隐私使适配器从可解析私有地址发起配对、同时 `btd` 又用同一适配器广播。重试、`JustWorksRepairing`、清双方绑定都无效，`bluetoothctl` 同样失败，故问题在本文件之下。Xbox 手柄只保存**一个**主机绑定，半成品尝试会让它握着本适配器已不再拥有的密钥，下结论前应先在笔记本上重置手柄。私有地址是 LE 机制，也据此可判断受影响的只能是 LE 绑定。

### 2.4 传输与验证边界

- `StartDiscovery()` 不带过滤，BlueZ 默认 `auto` 同时扫 BR/EDR 与 LE；`Snapshot` 每个属性可选也部分因为两种传输呈现的属性不同。
- 实战对象是 **LE-only** Xbox 手柄（绑定信息只有长期密钥、无 `[LinkKey]`、BlueZ 不报 `Class`）。因此 BR/EDR 分支（含 class-of-device 判定、`br-connection-profile-unavailable` 软失败）来自规范而非实测。发现仍保持 `auto`，因为 DualShock/DualSense 等被启发式点名的手柄是 BR/EDR HID，只滤 LE 会让这些硬件不可达。
- **已在 Radxa Zero 3W + Xbox Wireless Controller 真机验证**：发现、识别（基于 LE appearance 派生的 `Icon`）、绑定、`Trusted` 保持、重启后自动回连（LE 下由机器人作为 central 主动回连，同时 `btd` 作为 peripheral 广播，两种角色在该无线电上共存）、`padd` 可驱动、`pad forget` 可删除。
- **未在硬件上演练**：BR/EDR 绑定、DualSense、两个手柄同时配对、按显式地址配对。
- 无法在此修复的情况：`pad forget` 只删机器人这一侧绑定，手柄仍握着它那一半时，需重新进入配对模式或绑到别处，否则报同样的 `AuthenticationFailed`。

## 3. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `AGENT_PATH` | `/com/pollenrobotics/configd/pad_agent` | agent 在总线上的对象路径 |
| `AGENT_CAPABILITY` | `NoInputNoOutput` | 机器人无键盘无显示（硬件事实），也使手柄配对走 just-works |
| `BOND_TIMEOUT` | 30 秒 | 找到手柄后给绑定的总时长；BlueZ 自身配对超时 60 秒，保持其内 |
| `BOND_SETTLE` | 5 秒 | `Connect()` 成功后等自行绑定落地的窗口（真机 1 秒内完成，5 秒是余量；过短会对在途绑定调 `Pair()` 而挂死） |
| `BOND_POLL` | 200 毫秒 | 等绑定期间重读 `Paired` 的频率 |
| `DISCOVERY_POLL` | 500 毫秒 | 找手柄时重读对象树的频率。**轮询而非 `InterfacesAdded`**：后者只对从未见过的设备发出，配对后又被 forget（重试时的精确情形）的手柄留在缓存中永不再通告 |

## 4. zbus proxy

- `Adapter`（`org.bluez.Adapter1`）：`StartDiscovery`/`StopDiscovery`/`RemoveDevice`，属性 `Powered`（含 setter）。
- `Device`（`org.bluez.Device1`）：`Connect`/`Pair`，属性 `Trusted` 的 setter。
- `AgentManager`（`/org/bluez`）：`RegisterAgent`/`UnregisterAgent`/`RequestDefaultAgent`。

## 5. `PairingAgent`（`org.bluez.Agent1` 服务端）

- 字段仅 `device: OwnedObjectPath`。
- `permit(device, what)`：路径匹配才放行并记 `info`；否则记 `warn` 并返回 `fdo::Error::AccessDenied`。
- Agent1 回调实现：
  - `release`：调试日志。
  - `request_authorization(device)`：just-works 绑定实际调用的那个，交给 `permit(..,"bond")`。
  - `authorize_service(device, uuid)`：绑定后逐 profile（手柄为 HID）授权，`permit(..,"service")`。
  - `request_confirmation(device, passkey)`：远端有显示时的数字比较；本机无法比较，接受是唯一能让绑定继续的答案，passkey 记入日志留痕。
  - `request_pin_code` / `request_passkey`：**拒绝而非猜一个**（返回 `NotSupported`）。声明了 `NoInputNoOutput` 后 BlueZ 不应问；真问了说明对方要本机没有的凭据，回 `0000` 是捏造凭据且近年设备必失败。
  - `display_passkey` / `display_pin_code`：无屏可显，仅入日志；实现而非省略，因为方法缺失会让 BlueZ 以语焉不详的 D-Bus 错误使绑定失败。
  - `cancel`：记 `warn`「远端取消配对」。

## 6. 数据结构

- `type Interfaces = HashMap<OwnedInterfaceName, HashMap<String, OwnedValue>>`：`GetManagedObjects` 报告的一个对象的全部接口。
- `fn interface(interfaces, name)`：按名扫描接口属性（`OwnedInterfaceName` 不能按 `&str` 直接 `get`，且对象只有三四个接口）。
- `struct Snapshot`：来自 `GetManagedObjects` 的一次性快照（一次往返、无缓存陈旧），字段 `path/mac/name/icon/class/appearance/paired/trusted/connected`，属性全部可选。`read` 中：无 `Address` 则放弃；名字取 **`Alias` 优先、回退 `Name`**（BlueZ 展示 Alias 且它回落到 Name）；布尔缺失按 false。方法 `is_gamepad()`（委托 `looks_like_a_gamepad`）与 `as_pad()`。
- `struct Found { matches, seen }`：一次搜索的「像手柄者」与「无线电看到的全部」。

## 7. `struct BlueZ` 与内部方法

- 字段 `bus` 与 `pairing: tokio::sync::Mutex<()>`（**一次只配一个**：两个并发配对会争用发现和 agent 路径，且只有一个适配器、一个人、一只手柄）。
- `new()`：连系统总线。
- `objects()`：ObjectManager `GetManagedObjects`，错误信息区分「连不上 bluetoothd」与「bluetoothd 拒绝列举」。
- `adapter()`：找第一个适配器（排序后取路径最小者，避免 HashMap 顺序不定）；不存在（开机约 73 秒内 `hci0` 可能尚未出现）返回 `Ok(None)`；存在但未上电则尝试上电，以免把「无线电关着」误报成「没有手柄」。
- `devices()`：从对象树读出全部 `Device1` 快照。
- `find(mac, timeout) -> Found`：
  - 无显式地址时仅在发现**未绑定**候选时提前结束，否则跑到 deadline——已绑定手柄每次扫描都在，首条命中就返回会让「再加一个手柄」不可能；代价是无新设备时重跑要等满窗口（可用 `--timeout` 缩短）。
  - 有显式 MAC 时一旦出现即结束（配对与否都行），显式地址完全绕过启发式，不被二次猜测。
  - 同时返回全部 `seen`，因为新发现设备常只有地址、没有 Name/Class/Icon，裸报「没找到手柄」会让人拿不到 `--mac` 所需地址。
- `wait_until_paired(mac, within) -> bool`：轮询重读该 MAC 的 `Paired`，变 true 即成功；超时返回 false。
- `bond(device) -> Result<(), (PadPairFailure, String)>`：
  1. 未绑定才动作，先带 `BOND_TIMEOUT` 调 `Connect()`；成功/失败/超时三分支，失败也只软处理。
  2. **仅当 connect 成功**才给 `BOND_SETTLE` 等待（connect 失败时等待是纯损耗，会白白耗损手柄有限的配对窗口导致 `AuthenticationFailed`）。
  3. 等待后仍未绑定，则 `tokio::select!` 让 `Pair()` 调用与 `wait_until_paired` **赛跑**；`Pair()` 返回 `AlreadyExists`（`is_already_paired` 判定）也算成功；其他错误 → `Rejected`；超时且属性未变 true → `Timeout`。
  4. 绑定后再次 `Connect()`（软失败：已绑定手柄会自行回连）。
  5. 最后无条件 `set_trusted(true)`：这是重启后无人登录仍能回连的关键（不受信设备的回连需要 agent 批准，开机时没有 agent）；缺了它就是「昨天配得好好的今天没反应」。
- `fn is_already_paired(error)`：匹配 D-Bus 错误名 `org.bluez.Error.AlreadyExists`。

## 8. `impl Pads for BlueZ`

- `status`：取全部设备，**只留已绑定且被识别为手柄者**（扫描中见过的其他一切都是噪声），排序为已连接优先、再按名字。
- `pair(mac, timeout)`：
  1. 取 `pairing` 互斥锁。
  2. 无适配器 → `Failed { NoAdapter, "…hci0 appears about 73s after power-on." }`。
  3. `StartDiscovery`（失败作为错误报告，否则只能搜到缓存）。
  4. `find` 后**无论成败都 `StopDiscovery`**（错误路径也停，避免适配器一直扫描）。
  5. 候选处理：0 个 → `NotFound`，detail 在空场景提示同步灯必须快闪；有 seen 设备则列出**最多 8 个**非手柄设备（未绑定优先、按 MAC 排序，无名显示 `(no name yet)`），给出可用 `--mac`/地址逃生的命令提示；1 个 → 选它；多个 → 优先未绑定者，唯一 fresh 选它，全无 fresh 则取 MAC 最小的已绑定者（幂等重跑），多个 fresh 则 `Ambiguous` 并列出名字与 MAC。
  6. 在对象服务器上安装作用域于该设备路径的 `PairingAgent`；`RegisterAgent` 失败非致命（just-works 可能根本不问，且 `btd` 默认 agent 还在）；注册成功才 `RequestDefaultAgent`（失败只 warn：需要确认的手柄会卡住）。
  7. 调 `bond`；随后尽最大努力 `UnregisterAgent`、从对象服务器移除 agent。
  8. `bond` 失败 → 记日志并返回 `Failed { reason, detail }`；成功则**重读设备实况**（而非假定）转成 `Pad` 返回，找不到对象时以 `paired/trusted=true` 兜底；以 `warn` 级记录配对成功（保证在默认日志级别可见）。
- `forget(mac)`：无适配器或无此设备均返回 `removed: false`（诚实且与忘记未知设备一致）；否则调适配器 `RemoveDevice`，成功返回 `removed: true`。

## 9. 要点小结

- 配对顺序 connect→pair→trust 只「尝试」不「强制」；**`Paired` 属性是唯一真相**，`Pair()` 与属性变化赛跑，根治「首次超时、二次秒成」。
- 连接前停发现；连接后（仅成功时）留 5 秒绑定沉降；最后必须置 `Trusted` 才能重启回连。
- 配对窗口内注册单设备作用域 agent 并抢默认角色，否则 IO 能力停留 DisplayYesNo、数字比较无人应答。
- 发现靠 500ms 轮询而非 `InterfacesAdded`；未绑定候选优先；找不到时列出在场设备供 `--mac` 逃生。
- 板级依赖 `Privacy=device` 且新绑定期间需停 `btd`；Xbox 只存一个主机绑定。
- 真机仅验证 LE（Xbox + Radxa Zero 3W），BR/EDR 路径来自规范、尚未过硬件。
