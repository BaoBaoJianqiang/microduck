# `bluez.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 779 行 |
| 角色 | 无线电——通过 `bluetoothd` 的 D-Bus API 使用 BlueZ，仅限 Linux |
| 平台 | Linux only（`cfg(target_os = "linux")`） |
| 状态 | **未针对硬件测试**。类型检查 aarch64 通过，但从未见过真实中心。在有人连接手机之前，以下内容视为意图 |

## 二、核心定位

- 这里的一切是 BlueZ 和 `session` 的两个 channel 之间的管道。
- **此文件中不做关于机器人的任何决定**，这是重点：可能错的逻辑是被测试的逻辑，而这是需要无线电的部分。

## 三、使用 `bluer` 的回调模型

### 替代方案在硬件上试过且不工作

- `bluer` 的 IO 模型用 `NotSupported` 回答 BlueZ 的 `WriteValue` 和 `StartNotify`——它只服务 `AcquireWrite`/`AcquireNotify` fd 路径——而 CoreBluetooth 中心驱动普通方法。
- 结果是机器人广告、接受连接、接受订阅、接受写入，但什么都没递送到此文件：没有 `central connected` 行，没有配对提示，客户端针对正在工作的服务超时。
- IO 模型被选择为了一个结果不存在的好处。它在两半上报告 `device_address()`，看起来对将订阅配对到应该喂它的会话是必要的——但 `bluer` 每个 characteristic 持有**一个** `CharacteristicNotifyState`，因此永远只有一个通知会话要配对。一次一个中心是栈的属性，不是这里采取的捷径。

### 因此

- 服务寿命内一个会话，一个通知泵，一个将字节推入其中的写入回调。

## 四、常量

| 常量 | 值 | 说明 |
|---|---|---|
| `FLOOR_MTU` | 20 | 出站块假设的通知有效载荷。写入侧学习协商的 MTU（BlueZ 每请求报告）；通知侧无法询问。因此块按 20 字节大小——每个 BLE 链路必须支持的有效载荷——在好的链接上比必要的慢但在每个链接上正确 |
| `ADV_INTERVAL_MIN` | 100ms | 广告间隔下限。见下方详细说明 |
| `ADV_INTERVAL_MAX` | 150ms | 广告间隔上限 |
| `ADAPTER_RETRY` | 5s | 尝试找到可用适配器之间的等待时间 |
| `ASK_TIMEOUT` | 5s | 等待 configd 说机器人叫什么或有什么地址的时间 |
| `ADV_POLL` | 5s | 广告与 configd 说的内容协调的频率 |

### 广告间隔——这是被找到和不被找到的区别

**默认 1.28s 的问题**（实测）：
- 不设置时，BlueZ 采用内核默认的 **1.28 秒**。
- 在此板上从 Mac 连续扫描两分钟测量：机器人平均每 7.5 秒到达一次，有 9s、14s、17s、一次 31s 的静默。
- 房间里每个其他无线电——智能插座 -66 dBm、信标 -91 dBm——在同一窗口到达 130 到 212 次，对机器人的 16 次，而机器人是那里*最强*的信号 -36 dBm。
- 所以不是距离、不是干扰、不是客户端：它只是说得太少被抓到。
- 中心以低占空比扫描，这就是把"慢 6 倍"变成"一次缺席数秒"的原因——落在那些静默之一中的八秒扫描什么都找不到，大约一半都是。
- 大间隙以 1.28s 的近整数倍出现，这就是从到达而非猜测识别间隔的方式。

**100-150ms 的选择**：
- 普通外设使用的范围，是默认的 8-12 倍。
- 不是规范允许的 20ms 下限：一根天线承载这个、游戏手柄的 LE 链接和 wifi，因此花在喊叫上的空中时间是从机器人用途的东西上拿的。
- *范围*而非一个值，因为固定间隔可能一直与同一个邻居的碰撞，控制器通过在窗口内抖动来避免。

**实测效果**：
- 安装后再次测量，同一 Mac 同一两分钟：**151 次到达，每 0.8 秒一次，最差静默 3.8 秒，没有一次 8 秒或以上的静默。**
- 它被诊断出的失败在那个间距不可能发生，这就是重点——对八秒扫描的余量现在是两倍而非抛硬币。

### 适配器重试——73 秒

- 在此板上测量：`hci0` 在通电后大约 **73 秒**才存在——`aic-bluetooth.service` 延迟附加 AIC8800 的 UART，`bluetooth.service` 本身在 `dbus` 后面阻塞 26 秒。
- 在"无适配器"时退出的守护进程会被 systemd 重启到同样的空虚超过一分钟，因此它等待。
- 与 `robotd` 等待电机总线而非放弃它同样的教训。

## 五、`AbortOnDrop` 结构体

```rust
struct AbortOnDrop(tokio::task::JoinHandle<()>);
```

- 不超过启动它的 bring-up 的任务。
- 广告、GATT 应用和 agent 都在 drop 时注销，合唱的无线电任务必须遵循同样的规则——针对已消失的适配器运行的任务会永远重连 `robotd` 并在什么都没有上广告。
- `Drop` 实现调用 `abort()`。

## 六、`serve()`——永远服务 BLE

```rust
pub async fn serve(sockets: Sockets, name: NameChoice, require_pairing: bool) -> bluer::Result<()> {
    loop {
        match serve_on_an_adapter(&sockets, &name, require_pairing).await {
            Ok(()) => tracing::warn!("the adapter is gone; waiting for it to come back"),
            Err(e) => tracing::warn!(error = %e, "BLE bring-up failed"),
        }
        tokio::time::sleep(ADAPTER_RETRY).await;
    }
}
```

### 等待适配器*出现*从来不够

- 之后的一切——给适配器通电、注册 agent、广告、发布 GATT 应用——曾经将其错误传播出此函数并退出进程，因此出现然后行为不当的适配器把 `btd` 拿下，而从未出现的适配器不会。
- 在没有网络的机器人上，这是"wifi 不可用"和"不可达"的区别。

### 因此整个 bring-up 原地重试

- 与它已经做的等待相同的 5 秒节奏。
- **无线电故障永远不离开此函数。** 因此 `failed` 的 `btd` 意味着损坏的二进制，这就是它被准入启动恢复网的原因。
- **且它自愈。** 非零退出从 `Restart=always` 得到同样的重试，但只是通过花一个进程死亡在上面，且只到单元获得启动限制的那天。

### `require_pairing`

- 控制写入请求是否需要已认证的加密链接。
- 默认开启，因为 §7 要求任何携带 wifi 凭据的东西，而 `net.connect` 现在确实携带。
- opt-out 存在用于针对无法配对的客户端的台架工作。

## 七、`serve_on_an_adapter()`——一次 bring-up

### 流程

1. **获取 BlueZ session**：`bluer::Session::new()`。
2. **等待适配器**（自己的循环，不折叠到调用者的）：
   - "还没有适配器"是板前 73 秒的普通状态，读为进度，而此点之后的失败是故障。合并它们会在正常启动期间每 5 秒记录一次故障。
   - `bt.default_adapter()`，失败则等待 `ADAPTER_RETRY`。
3. **通电**：`adapter.set_pairable(true)`。
4. **可配对**：如果 `require_pairing`，`adapter.set_pairable(true)`。
   - 只在广告时重要，板默认报告 `Pairable: no`。
   - 保持开放而非门控在窗口后面：PIN 携带窗口会添加的东西，只要它是每机器人的。
5. **注册 just-works agent**（如果 `require_pairing`）：
   - `Agent { request_default: true, ..Default::default() }`——所有 handler 为 `None`，bluer 发布为 `NoInputNoOutput`。
   - 因此绑定不需要交互，加密但*不*认证。
   - 这不是预期的设计。第一次版本用存储的 PIN 回答 BlueZ 的 passkey 请求，在无头机器人上无法工作（见 `pairing.rs` 详细说明）。
   - PIN 检查因此上移到链路层：`session` 在客户端通过 `system.authenticate` 之前什么都不服务。
   - 如果 `!require_pairing`，警告任何范围内的设备都可以到达 RPC characteristic（PIN 仍由会话强制执行）。
6. **记录服务状态**：适配器名、bd_addr、service UUID、pairing、max_adv_len（`crate::adv` 写入所依据的预算——报告更少的控制器是该假设失败的一个地方，是机器人广告名字但没有地址时首先要读的东西）。
7. **获取广告名称和地址**：
   - 名称：`name.pinned` 或 `ask_name()`。
   - 地址：`ask_address()`——在第一次广告之前询问而非留给第一次协调 tick：否则启动到已经知道的网络的机器人会在前几秒广播 `0.0.0.0`，列表无法将其与根本没有 wifi 的机器人区分。
8. **广告**：`advertise(&adapter, &advertised)`。
9. **启动合唱无线电任务**：
   - 独立任务而非会话循环的一部分：它不服务客户端，且不能阻塞此守护进程存在的唯一目的。
   - 它的失败是它自己的——还没起来的 `robotd` 是启动时的普通情况。
   - 包裹在 `AbortOnDrop` 中，因此 bring-up 结束时中止。
10. **创建当前会话槽**：
    - `Arc<StdMutex<Option<mpsc::Sender<Vec<u8>>>>>`——`std::sync::Mutex` 而非 tokio 的，故意：写入回调必须无需等待就读取此，因为那里的让步点让两个块交换位置。没有东西在 await 上持有。
11. **构建 GATT 应用**（见下方详细说明）。
12. **服务 GATT 应用**：`adapter.serve_gatt_application(app)`。
13. **等待**：`tokio::select!` 在 `watch_adapter()` 和 `reconcile_advertisement()` 上——任一先完成结束 bring-up。

### 一个会话 per 订阅，不是 per 守护进程

- 第一次版本为整个服务保持单个会话活着，更简单且错：
  - 在请求中间消失的客户端在重组器中留下部分行，在出站队列中留下未递送的块，*下一个*客户端被交给它们。
  - 表现为没有开头的回复到达——`":0,"result":{"authenticated":true}}`——这是前一次运行答案的尾部。
- 在中心订阅时创建，它离开时拆除。
- 先订阅是每个客户端使用的顺序，没有实时订阅的写入被拒绝：没有地方发送答案。

## 八、GATT 应用结构

一个 service（`SERVICE_UUID`，primary），一个 characteristic（`RPC_UUID`），三个属性：

### 1. Read（`CharacteristicRead`）

- 唯一工作是在写入任何东西之前强制绑定。
- §7 要求携带 wifi 凭据的 characteristic 配对并加密。读是被确认的，因此未配对的中心得到"insufficient authentication"并在那时那里开始配对，订阅做不到：`CharacteristicNotify` 完全不携带加密标志。
- **注意：这在实践中目前是*未加密*路径——见 `docs/design/app-path-design.md` §5.5。在这里要求加密会挂起 CoreBluetooth。**
- 值不如读它需要绑定这个事实重要；API 版本是可用的最有用字节，找到不认识的版本的客户端可以在写入任何东西之前说明。
- `encrypt_read: require_pairing`。
- 回调返回 `vec![API_VERSION as u8]`。

### 2. Write（`CharacteristicWrite`）

- `write: true`，`write_without_response: true`（分块请求不需要每块 ATT 确认。想要*拒绝*可见的客户端必须使用确认形式，这就是 `duckctl` 做的）。
- `encrypt_write: require_pairing`。
- **方法：`CharacteristicWriteMethod::Fun`**（回调）。

#### 写入回调的关键约束

- **接收块和入队之间没有 `.await`。** BlueZ 将每个 `WriteValue` 作为自己的任务分发，因此这里的让步点让两个块交换位置——重排的块静默损坏请求而非失败它。`main` 也将运行时固定到一个线程，原因相同。
- 锁定 `for_write` mutex，克隆 sender。
- 如果 `sender` 是 `None`（没有订阅）→ 拒绝（`GattError::Failed`）：接受请求会是谎言，因为没有地方发送答案。客户端先订阅；这是没订阅的客户端。
- 如果 `tx.try_send(value)`：
  - `Ok` → 成功。
  - `Full` → 拒绝（可恢复——客户端重发。丢弃块不是：行会重组为解析为错误内容的东西）。
  - `Closed` → 拒绝（会话已结束）。
- 记录 debug：peer、mtu、bytes、ok、head（块的前 8 字节，因此重排在日志中可见而非从三层之上的解析错误推断。截断因为请求可能携带 wifi 密码）。

### 3. Notify（`CharacteristicNotify`）

- `notify: true`。
- **方法：`CharacteristicNotifyMethod::Fun`**（回调）。

#### 通知回调

当中心订阅时调用：

1. 创建新的 `Link::pair(FLOOR_MTU, "central")`。
2. 锁定槽，设置 `*slot = Some(inbound)`：
   - 如果槽已经是 Some，警告"另一个中心已订阅；替换其会话"——bluer 每个 characteristic 保持一个通知状态，因此这是替换而非共享：两个客户端通过一个重组缓冲区会交错它们的请求。
3. 生成 `session::run(link, sockets)` 任务。
4. 循环 `select!`（**biased**）：
   - `notifier.stopped()` → break（中心走了。没有这个，泵只在通知失败时学习中心走了——这需要发送回复，因此空闲时断开的客户端会持有槽直到下一个请求到达给 nobody）。
   - `outbound.recv()` → 通知块。失败 → break。
5. 清理：
   - **只在槽仍然是*我们的*时清除。** 此任务可能比其订阅活得久——给消失的中心的通知花 BlueZ 放弃的时间——到那时重连的中心可能安装了更新的会话，盲目的 `take()` 会杀死它。
   - 检查 `slot.as_ref().is_some_and(|tx| tx.same_channel(&mine))`。
   - 如果是我们的：`slot.take()`，`session.abort()`，记录"中心取消订阅；会话被丢弃"（丢弃 sender 结束会话任务，丢弃其重组缓冲区和上游连接）。
   - 如果不是：记录"更新的会话持有槽；不管它"，`session.abort()`。

## 九、`Advertised` 结构体

```rust
struct Advertised {
    name: String,
    address: Option<Ipv4Addr>,  // None = 没有 IPv4 地址，出去为 0.0.0.0
}
```

- 广告关于机器人说的内容：它叫什么，它在网络上的哪里。
- 一个结构体而非两个参数穿过协调循环，因此"有任何东西移动了吗"是一次比较。
- `Display` 实现用于日志（有趣的行是说什么变了的那行）：
  - 有地址 → `"name at address"`
  - 无地址 → `"name with no address"`

## 十、`advertise()`——广告服务

### 两个名称，因为机器人有两个且只有一个曾经被设置

- 广告携带 Local Name；适配器单独服务 GAP Device Name（`0x2A00`），BlueZ 从 `Adapter.Alias` 取，默认为主机名。
- 因此重命名的机器人广告 `duck-5b21` 同时对任何读 characteristic 的人回答 `radxa-zero3`——而读了它的客户端保留答案：
  - **BlueZ 在广告名称之上缓存它。** `Device1.Name` 是 `btleplug` 报告的，因此在 Linux 上机器人在第一次连接前是 `duck-5b21`，之后是 `radxa-zero3`，`duckctl --name duck-5b21` 然后什么都找不到。相隔一分钟的两次扫描不一致。
  - **CoreBluetooth 保持两者**，`btleplug` 加入为 `radxa-zero3 [duck-5b21]`。
  - **手机的蓝牙设置显示 GAP 名称**，这是最重要的情况，也是此仓库中什么都看不到的情况。
- 因此设置别名是命名机器人的一部分而非好处，它属于这里以便没有路径可以发布名称而不带它。
- 别名持久在 BlueZ 自己的状态中，因此当它已经说正确的东西时跳过写入。
- 设置失败被记录且不传播：别名不如根本可见有价值，在这里返回错误会把广告也拿下。

### 地址字段被丢弃而非允许注册失败

- `crate::adv` 中的算术说有效载荷 fits，但溢出传统广告的字节是控制器数的，不是我们的——BlueZ 在它不 fits 时拒绝整个注册。
- 在唯一前门可能是 BLE 的机器人上，这个交易不接近：没有地址的广告是某人仍然可以到达的机器人，被拒绝的是已经变暗的机器人。
- 因此先尝试带地址，失败则回退到不带地址（记录警告，`duckctl scan` 将显示此机器人完全没有地址）。

### 广告内容

- `service_uuids: [SERVICE_UUID]`
- `manufacturer_data: [(COMPANY_ID, address_data)]`（如果有地址）
- `discoverable: Some(true)`
- `local_name: Some(name)`
- `min_interval: Some(ADV_INTERVAL_MIN)`
- `max_interval: Some(ADV_INTERVAL_MAX)`

## 十一、`reconcile_advertisement()`——保持广告与 configd 同步

- 永远不返回。
- 拥有广告句柄，因为改变任一意味着注销一个广告并注册另一个——在那发生时没有其他东西可以持有它。
- `pinned_name`：`--name`，名称是此进程自己的，没有人可以问，因此只有地址被协调。循环仍然运行，因为固定名称不固定 DHCP 租约。

### 循环

1. 睡眠 `ADV_POLL`（5 秒）。
2. 获取当前 `Advertised`：
   - 名称：如果 `pinned_name`，保持旧名称；否则 `ask_name()`。
   - 地址：`ask_address()`。
3. 如果 `current == advertised && handle.is_some()` → continue（没变化）。
   - `handle` 是 `None` 只在失败的重新广告之后，那时机器人不可见——因此无论是否有变化都重试。
4. **先注销再注册替换**：BlueZ 被要求改变一个广告，在交换时持有两个会邀请它拒绝第二个。间隙短暂，已经连接的中心不会注意到——连接不是广告。
5. `advertise(adapter, &current)`：
   - 成功 → 如果 `current == advertised`，记录"失败后再次广告"；否则记录从旧到新的变化。更新句柄和 advertised。
   - 失败 → 留给下一个 tick 而非致命，永远不传播：这在 bring-up 内部，其全部要点是无线电故障不结束进程。

### 为什么轮询而非事件驱动

- `btd` 将 `system.setName` 转发到 `configd` 而不读回复（`upstream::Pool` 为客户端合并行，在这里解释它们正是此守护进程避免的），因此它不通过观看学习重命名。
- 它可以在转发一个的那一刻重新询问，但它刚转发的写入可能还没被应用，第二个连接对第一个没有排序保证。
- 协调反而是更少的活动部件，覆盖每个重命名路径，包括通过 unix socket 的 `robotctl system set-name`，它根本不穿过此进程。
- 代价是每几秒一次 socket 连接和一行，永远，远低于已经为无线电等 73 秒的守护进程的噪声底。

### 地址在同一 tick、同一节奏被询问

- 比 DHCP 租约可能移动的速度快。
- 两个问题而非一个是第二次 socket 连接和 `configd` 从 NetworkManager 回答的 `net.status`，拆分节奏会以第二个计时器和地址滞后 `wifi connect` 半分钟为代价买回一些。
- 机器人在那一刻刚被给予网络，地址是做这件事的人等着读的东西。

## 十二、`ask_name()`

- 向 configd 询问 `system.info`，带 `ASK_TIMEOUT`。
- 成功 → 返回 `info.name`。
- 失败 → 返回 `fallback`（最后已知名称）。
- 失败是 `debug` 而非 `warn`：这每几秒运行，重启的 configd 会否则用自行解决的条件填满日志。启动调用是重要的那个，由调用者通过它最终广告的名称记录。

## 十三、`ask_address()`

### 两种失败不是同一个答案，合并它们使广告闪烁

- `configd` 报告无地址是不在 wifi 上的机器人，那清除字段。
- `configd` 不回答——重启，或 NetworkManager 比 `ASK_TIMEOUT` 花更长——对机器人的网络什么都没说，在它上面清除字段会在中断持续的每个 tick 注销并重新注册广告，客户端看着地址闪烁。
- 因此中断保持最后已知地址，与 `ask_name` 保持最后已知名称完全一样。

### 流程

1. 向 configd 询问 `net.status`，带 `ASK_TIMEOUT`。
2. 成功 → 解析 `status.ip4`：
   - `None` → 无地址。
   - `Some(address)` → 解析为 `Ipv4Addr`。
     - **解析而非信任**：`ip4` 是 NetworkManager 放在 `address-data` 中的任何东西，此无法解析的字符串不是要广播 4 字节的东西。解析失败 → 警告，返回 `None`。
3. 失败 → 返回 `last`（最后已知地址）。
- 只有 IPv4，因为只有 IPv4 fits（见 `adv.rs`）。
- 失败是 `debug` 而非 `warn`，与 `ask_name` 同样原因。

## 十四、`watch_adapter()`——适配器停止可用时返回

- 轮询，不是事件流。`bluer` 可以报告适配器移除，但这必须捕获的失败比移除更广——仍在总线上但什么都不回答的适配器是曾经杀死进程的情况——读一个属性覆盖两者，不依赖 BlueZ 为楔住而非缺席的无线电发射哪些事件。
- 间隔是 `ADAPTER_RETRY`，因为注意到晚的代价正是重试晚的代价：BLE 再暗几秒，在否则空闲的守护进程上。
- 循环：
  - 睡眠 `ADAPTER_RETRY`。
  - `adapter.is_powered()`：
    - `Ok(true)` → 继续。
    - `Ok(false)` → 警告"适配器不再通电"，返回（在我们下面被断电——被 `bluetoothctl power off`、驱动重置或挂起。下一次 bring-up 再次通电：在唯一前门可能是 BLE 的机器人上，未通电的适配器不是出于礼貌要保留的状态）。
    - `Err(e)` → 警告"适配器停止回答"，返回。
#（注：内容由AI生成）
