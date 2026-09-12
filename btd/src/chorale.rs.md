# `chorale.rs` 文件解析 —— 鸭子合唱（Chorale）的无线电

## 1. 文件定位

- 路径：`src/chorale.rs`
- 角色：合唱功能的无线电：向外广播一个信标（beacon），并接收其他鸭子的信标。
- 仍然**只是传输适配器**：btd 不持有合唱状态、不做合唱决策，只广播被交给它的字节、上报它听到的字节，与它为 JSON-RPC 做的工作同构。谁指挥、谁唱什么、何时起拍，全部由 `robotd` 决定（那些是"行为"）。
- 为什么不单独做一个守护进程：它需要的东西这里全都有——适配器、bluer、D-Bus 权限、一条到 robotd 的连接。再起一个进程争用同一适配器只会增加零件，换不来任何有意义的隔离。

## 2. 模块文档中的关键设计

### 2.1 出站：独立的第二广播实例

- 板载控制器报告有 5 个广播实例、仅用了 1 个，所以信标用自己的实例，**现有前门广播完全不被改动**；
- 这不是洁癖：`adv` 记录的是 31 字节预算，而本机控制器上报 251 字节；在原实例上 BlueZ 会乐于接受更大载荷，并按尺寸在 legacy/扩展 PDU 间切换，从而把机器人变成只支持 legacy 的扫描器看不见的设备——手机发现能力会退化，且看起来像蓝牙故障而非合唱故障。

### 2.2 按需注册（承重设计）

- 控制器会交错播出多个广播实例，注册第二个会让**第一个的速率减半**；
- `bluez.rs` 的 100–150ms 间隔是实测调出来的（默认 1.28s 曾让机器人一次消失长达 31s），不能为一个还没人要求的功能永久花掉这份裕度；
- 因此信标只在需要合唱时注册，不需要时立即注销。

### 2.3 入站：两种监听方式

理想方式是 BlueZ 的 advertisement monitor（`bluer::monitor`）：在**控制器内**按字节模式过滤，主机只在已是合唱信标时才被唤醒，且被动监听、不发射（同一根天线还承载手柄链路与 wifi）。

- 但本机 BlueZ 5.82 中 `AdvertisementMonitorManager1` 仍需 `bluetoothd --experimental`，而守护进程未带该标志，`RegisterMonitor` 返回 `UnknownMethod`；全局开启 experimental 会在手机唯一入口之下启用一堆未完成接口，为省电不值得；
- 因此 `watch()` 先试 monitor，失败回退到历史悠久的普通 discovery。回退代价真实存在：
  - **主动扫描**，鸭子会发射，占手柄的空中时间；
  - 需要 **`duplicate_data`**：BlueZ 默认抑制重复广播数据，而信标的*载荷*就是全部信号，不打开则第一拍之后全不可见，跟随者什么都锁不到；
  - 过滤发生在主机而非控制器，字节模式由 `beacon_in` 事后应用。

无论哪条路，判别依据相同：公司 id `0xFFFF` 的 manufacturer data 且以 `ChoraleBeacon::TAG` 开头。这个 TAG 正是另一广播实例那 4 字节 IPv4 地址不会被误当成一拍的原因。

### 2.4 到达时间的含义

- 目击时间戳在本进程看到时打下，经历了控制器、内核、D-Bus 属性变化，含数毫秒延迟与抖动；
- 上游设计（`sounds` crate 的 `sounds::chorale::beat`）本就对此鲁棒：在很多拍上对相位取平均，随机量被平均掉；*常量*偏差在相同硬件的每只鸭子上都一样因而不可闻；
- 无法吸收的是指挥"认为自己何时发出"与跟随者之间的系统性差异——该常量应上真机测量而非假设。

## 3. 平台无关部分

- `COMPANY_ID: u16 = crate::adv::COMPANY_ID`（即 `0xFFFF`）：与地址字段共用测试保留 id；
- `BEACON_INTERVAL_MIN/MAX = 20ms / 40ms`：比前门的 100–150ms 更快，因为信标承载节拍，载荷变化到达空中的及时性*就是*同步误差；仅在合唱运行时存在才负担得起；
- `scan_pattern() -> Vec<u8>`：返回小端公司 id 两字节 + TAG，共 3 字节，即控制器匹配模式（从 manufacturer-data AD 字段起始处）；
- `beacon_data(beacon) -> Vec<u8>`：调 `ChoraleBeacon::to_bytes()`。返回载荷而非 map，呼应 `adv::address_data`，也因两端容器不同（bluer 广播用 BTreeMap、上报扫描结果用 HashMap）；
- `beacon_in(&HashMap<u16,Vec<u8>>) -> Option<ChoraleBeacon>`：调 `ChoraleBeacon::from_bytes`，对同公司 id 下的其他内容（前门实例的 4 字节地址、其他厂商对 0xFFFF 的合法使用）返回 `None`，TAG 是判别符。

## 4. Linux 无线电实现（`mod radio`）

### 4.1 `struct Sighting { beacon, from: bluer::Address, at: Instant }`

一次听到的其他鸭子信标。`from` 仅用于去重的身份；信标除 register 与 tie-break 字节外不说明谁在广播。

### 4.2 `broadcast(adapter, beacon)`

- 在自己的实例上广播：仅 manufacturer data；`discoverable=false`（这是信标不是入口，前门广播不变）；20–40ms 间隔；
- 不带服务 UUID（扫描按厂商数据过滤，18 字节 UUID 无价值）也不带名字（只会把载荷撑到改变 PDU 类型）；
- drop 返回的句柄即停播，这正是"按需注册"的释放机制。

### 4.3 `run(adapter, robot_socket)`

- 一条到 robotd 的连接承载双向：下行 `chorale.subscribe` 流告诉广播什么，上行 `chorale.heard` 通知上报听到了什么；robotd 决定一切，本函数只持无线电；
- 连接或适配器消失即返回，调用方按 `bluez::serve` 同样方式重启；启动时 robotd 尚不在是正常重试而非失败；
- 状态：`_advertisement: Option<AdvertisementHandle>` 与 `listening: Option<JoinHandle>` 仅为"能被 drop/abort"而持有；`heard` channel 容量 64；
- select 循环：
  - 收到 `ChoraleBeaconSet(want)`：有 beacon 就 `broadcast`（失败记 warn、置 None），无则停播；按 `want.listening` 启动一次监听任务或 abort 之（不会每拍重启 monitor）；
  - 收到 `Sighting`：构造 `ChoraleHeard { beacon, from: 地址字符串, age_us }` 发通知。上报的是**年龄（age）而非时间戳**：两守护进程共享机器却不共享时钟纪元，age 经 socket 传递后仍有意义。

### 4.4 `watch()` / `monitor()` / `discover()`

- `watch`：先 `monitor`，**注册失败**才回退 `discover`（已注册后死亡的 monitor 按其本身上报）；
- `monitor`：注册 `OrPatterns` 监视器，模式为 AD 类型 `0xFF`（厂商数据）、起始 0、内容为 `scan_pattern()`；`rssi_sampling_period = All`——要每个包而不仅是首次目击，因为变化的载荷才是拍；随后对 `DeviceFound` 逐设备 `spawn_follow`；
- `discover`：设置 LE 过滤并打开**承重的** `duplicate_data: true`（否则重复载荷被抑制，第一拍之后全丢），用 `discover_devices_with_changes()`，对 `DeviceAdded` 逐设备跟踪。

### 4.5 `spawn_follow(device, tx)`

- 每听到一只鸭子一个任务：无论经哪条路发现，重复到达的*载荷*都表现为设备上的属性变化，这才承载节拍；
- 先把设备发现时已在广播的内容处理一次（否则第一拍会在等待"变化"时被错过）；
- 然后监听 `DeviceEvent::PropertyChanged(ManufacturerData)`，尽早打 `Instant::now()` 时间戳（其后的一切都是相位平均要吸收的抖动），`beacon_in` 解析成功就发 `Sighting`，发送失败（接收方消失）即退出。

## 5. 单元测试（5 个，全部平台无关）

| 测试 | 要点 |
| ---- | ---- |
| `a_beacon_survives_the_advertisement` | 广播端 `beacon_data` 与扫描端 `beacon_in` 往返一致 |
| `the_address_instance_is_not_heard_as_a_beat` | 前门实例的 4 字节 IPv4（同公司 id）不被读成信标；反向信标也不被读成地址（`address_in=None`、无地址字段） |
| `the_scan_pattern_matches_what_is_broadcast` | 扫描模式逐字节等于实际广播字段前缀 `[0xFF,0xFF,TAG]`；地址广告不得匹配，否则过滤器失效 |
| `somebody_elses_payload_is_not_a_beacon` | 空/各种长度的他方载荷、以及不同公司 id（`0x004C`）的良形信标都不被认作本节拍 |
| `the_beacon_is_faster_than_the_front_door` | 20ms ≤ min < max < 100ms：不低于规范下限、确实快于常驻前门广播（快的可负担性依赖实例是临时的） |

## 6. 本文件要点小结

1. 合唱仍只是字节搬运，决策权全在 robotd；因所需资源齐备而并入 btd；
2. 信标用独立广播实例、按需注册，避免拖慢前门广播或改变其 PDU 类型；
3. 监听优先控制器内 monitor，不可用时回退主动 discovery（需 duplicate_data、主机侧过滤）；
4. TAG 区分信标与同公司 id 的 IPv4 地址字段；
5. 与 robotd 单连接双向收发，上报用 age 而非时间戳；每只鸭子一个 follow 任务，包含首次值并尽早打时间戳；
6. 20–40ms 信标间隔与往返/模式/抗混淆均由测试锁定。
