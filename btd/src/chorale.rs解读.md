# `chorale.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 486 行 |
| 角色 | 鸭子合唱的无线电：信标发出，其他鸭子的信标进入 |
| 平台 | 跨平台逻辑 + Linux-only 无线电实现（`radio` 模块用 `cfg(target_os = "linux")` 隔离） |

## 二、仍然是传输适配器

- `btd` 不拥有任何合唱状态，不做任何合唱决定——它广播被交给的字节，报告听到的字节，这与它为 JSON-RPC 做的工作相同。
- 谁指挥、谁唱什么、何时开始都是 `robotd` 的，因为它们是行为。

### 为什么住在这里而非独立守护进程

- 它需要的一切已经在这里：适配器、`bluer`、D-Bus 权限、到 `robotd` 的连接。
- 第二个进程争用同一个适配器会是更多活动部件，而没有重要的分离。

## 三、出：第二个广告实例

- 板报告五个广告实例，一个在用，因此信标得到自己的，**现有广告不被触碰。**
- 这不是整洁。`adv` 记录了 31 字节预算，而这里的控制器报告 251 字节——因此 BlueZ 会愉快地在现有实例上接受更大的有效载荷，并且因为它按大小在传统和扩展 PDU 之间选择，会将其切换到扩展，使机器人对仅传统扫描器不可见。
- 手机发现会退化，看起来像蓝牙故障而非合唱故障。

## 四、按需注册，这是承重的

- 控制器交错其广告实例，因此注册第二个**将第一个的速率减半**。
- `bluez` 的间隔是针对测量调优的——默认 1.28 秒让机器人一次缺席最多 31 秒，100-150 ms 修复了它——因此永久减半会把来之不易的余量花在还没人要求的功能上。
- 信标因此只在需要合唱时注册，不需要时丢弃。

## 五、入：两种监听方式，因为好的不总在那里

### 想要的：BlueZ 的广告监视器（`bluer::monitor`）

- 在*控制器中*按字节模式过滤，因此主机只被已经是合唱信标的广告唤醒。
- 被动——鸭子监听时不传输任何东西，这在一根天线承载这个、游戏手柄链接和 wifi 时很重要。

### **它不在此板上**

- `AdvertisementMonitorManager1` 在 BlueZ 5.82 中仍在 `bluetoothd --experimental` 后面，而这里的守护进程没有它运行，因此 `RegisterMonitor` 返回 `UnknownMethod`，两只鸭子坐在房间里什么都听不到。
- 全局打开 `--experimental` 会在手机唯一入口下方启用一组其他未完成的接口，这对省电来说是糟糕的交易。

### 回退：普通发现

`watch` 尝试监视器并回退到普通发现（每个 BlueZ 都有）。回退的代价是真实的：

- **主动扫描**，因此鸭子传输。从游戏手柄拿走更多空中时间。
- **需要 `duplicate_data`**。BlueZ 默认抑制重复广告数据，而信标的*有效载荷*是整个信号——没有这个，第一次之后的每个节拍都不可见，跟随者什么都锁不到。
- 过滤发生在主机而非控制器，因此字节模式由 `beacon_in` 事后应用而非唤醒前。

### 区分器相同

- manufacturer-data，company id `0xFFFF`，以 `ChoraleBeacon::TAG` 开头。
- 那个标签就是*另一个*广告实例的 4 字节 IPv4 地址不被当作节拍送到这里的原因。

## 六、到达时间意味着什么

- 目击被加盖时间戳是在此进程看到它时，这在控制器、内核和 D-Bus 属性变化之后。
- 那条路径增加几毫秒和一些抖动——上游设计（`sounds::chorale::beat`，在 `sounds` crate 中）正是为此构建：它在许多节拍上平均相位，因此随机的东西平均掉，*恒定*的东西在相同硬件上的每只鸭子都相同因此听不见。
- 它不能吸收的是指挥对其节拍何时出去的想法与跟随者之间的系统差异——这就是为什么那个常数想要在硬件上测量而非假设。

## 七、跨平台部分（常量与函数）

| 项 | 说明 |
|---|---|
| `COMPANY_ID` | 与 `adv` 相同的 `0xFFFF` |
| `BEACON_INTERVAL_MIN = 20ms` | 比 `bluez` 的 100-150ms 快，故意：这个携带节拍，有效载荷变化多快到达空中*就是*同步误差。可负担因为实例只在合唱运行时存在 |
| `BEACON_INTERVAL_MAX = 40ms` | 同上 |
| `scan_pattern() -> Vec<u8>` | 扫描过滤器必须匹配的字节，从 manufacturer-data AD 字段开始：小端 company id + 信标签 |
| `beacon_data(beacon) -> Vec<u8>` | 信标作为广告的 manufacturer-data 有效载荷 |
| `beacon_in(manufacturer_data) -> Option<ChoraleBeacon>` | 设备 manufacturer data 中的信标（如果有）。`None` 用于同一 company id 上的任何其他东西——另一个实例广播的地址字段，或使用 `0xFFFF` 的其他供应商 |

## 八、Linux-only `radio` 模块

### `Sighting` 结构体

```rust
pub struct Sighting {
    pub beacon: ChoraleBeacon,
    pub from: bluer::Address,  // 仅用于去重的身份
    pub at: Instant,            // 此进程看到它的时间
}
```

### `broadcast(adapter, beacon) -> Result<AdvertisementHandle>`

- 在自己的广告实例上把信标放到空中。
- 丢弃返回的句柄停止它，这就是实例在合唱结束那一刻被释放的方式。
- 不携带 service UUID 或本地名称：扫描过滤在 manufacturer data 上，因此 18 字节 UUID 买不到什么，名称只会使有效载荷大到足以改变 PDU 类型。
- `discoverable: Some(false)`：此实例是信标，不是入口。机器人的前门是另一个广告，不变。

### `run(adapter, robot_socket) -> io::Result<()>`

- 只要适配器活着就服务合唱。
- 到 `robotd` 的一个连接，携带两个方向：
  - `chorale.subscribe` 流下来说要广告什么
  - `chorale.heard` 通知上去说到达了什么
- `robotd` 决定一切；此只持有无线电。
- 连接或适配器消失时返回，因此调用者以与 `bluez::serve` 在丢失适配器时重启相同的方式重启它。
- 不在那里的 `robotd`——启动时的普通情况——是重试，不是失败。

#### 核心循环

1. 连接 `robotd` socket。
2. 发送 `chorale.subscribe` 请求。
3. 循环 `select!`：
   - **来自 robotd 的行**：解析为 `ChoraleBeaconSet`，根据 `want.beacon` 广告/停止广告，根据 `want.listening` 启动/停止扫描任务。
   - **来自 heard channel 的 Sighting**：发送 `chorale.heard` 通知，带 `age_us`（*年龄*而非时间戳：两个守护进程共享机器但不共享纪元，年龄在 socket 下行中存活的方式是另一个进程的时钟读数做不到的）。

### `watch(adapter, tx) -> Result<()>`

- 永远监听其他鸭子的信标，将每个目击发送到 `tx`。
- 优先控制器卸载的监视器，BlueZ 没有监视器可提供时回退到普通发现。
- 任何*注册*失败都回退；已注册然后死亡的监视器是另一回事，作为自身报告。

### `monitor`（内部，好路径）

- 注册 `OrPatterns` 类型的监视器，模式匹配 manufacturer data 中的 `scan_pattern()`。
- `rssi_sampling_period: All`：每个包，不只是设备的第一次目击——有效载荷是变化的东西，被告知一次的节拍不是节拍。
- 循环 `MonitorEvent::DeviceFound`，为每个设备生成 `spawn_follow`。

### `discover`（内部，回退）

- 设置发现过滤器：`transport: Le`，**`duplicate_data: true`（承重的）**。
- 没有它 BlueZ 报告设备的广告数据一次并抑制重复——而信标的有效载荷是整个信号，因此第一次之后的每个节拍都不可见。
- 循环 `AdapterEvent::DeviceAdded`，为每个设备生成 `spawn_follow`。

### `spawn_follow`（内部）

- 只要一只鸭子的广告保持变化就监听它。
- 每只听到的鸭子一个任务：无论以哪种方式找到，*有效载荷*重复到达是设备上的属性变化，那就是携带节拍的东西。
- 首先读取已经广告的内容（否则第一个节拍在等待已经发生的变化时被错过）。
- 然后循环 `DeviceEvent::PropertyChanged(ManufacturerData)`，尽可能早地加盖时间戳（此进程能做到的最早：之后的一切都是相位平均必须吸收的抖动）。

## 九、测试（5 个）

| 测试 | 验证内容 |
|---|---|
| `a_beacon_survives_the_advertisement` | 信标往返：编码→解码一致 |
| `the_address_instance_is_not_heard_as_a_beat` | 标签存在的陷阱：另一个广告实例在同一 company id 下广播 4 字节 IPv4，读为信标的扫描器会在地址中听到节拍。反向也验证：信标不被读为地址 |
| `the_scan_pattern_matches_what_is_broadcast` | 扫描过滤器逐字节匹配信标实际广播的内容，否则控制器丢弃每个节拍，失败看起来像无线电问题 |
| `somebody_elses_payload_is_not_a_beacon` | 测试 company id 上的其他供应商，或此构建不知道的未来信标，不是节拍 |
| `the_beacon_is_faster_than_the_front_door` | 信标比前门广告快，且只有因为实例是瞬态的才可负担。如果有人使其永久，这些数字就是论据 |
#（注：内容由AI生成）
