# nm.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/nm.rs`
- **角色**：`Net` trait 的生产后端，经 D-Bus 驱动 **NetworkManager**（仅 Linux）。这是「输错口令为什么能被明确报告」这一选型价值的兑现处。
- **重要现状（文件头明示）**：**尚未对真实 NetworkManager 验证过**；它能为 aarch64 通过类型检查，文件中每项断言在跑上板子之前都只是意图。

## 2. 技术选型（模块文档）

- 用 `zbus` 而非 `dbus` crate：纯 Rust、无 vendored C；NM 的设置是嵌套 `a{sa{sv}}`，`zvariant` 表达毫不费力。代价是产物携带两套 D-Bus 栈（`btd` 经 `bluer` 链 libdbus）；若 `bluer` 长出 zbus 后端，或 BlueZ 调用小到可手写，值得回头重估。

## 3. 常量

### `mod ids`：NM 原生数值（取自上游头文件 `nm-database-interface.h`，命名而非内联）

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `DEVICE_TYPE_WIFI` | 2 | Wi-Fi 设备类型 |
| `STATE_UNAVAILABLE` / `DISCONNECTED` / `ACTIVATED` / `FAILED` | 20 / 30 / 100 / 120 | 设备状态 |
| `ACTIVE_ACTIVATED` / `DEACTIVATING` / `DEACTIVATED` | 2 / 3 / 4 | **一次激活尝试**（ActiveConnection）的状态，与设备状态是不同问题 |
| `STATE_NEED_AUTH` | 60 | 设备正在索要密钥；NM 在口令被拒、放弃前经过此态，该迁移带着失效原因 |
| `REASON_NO_SECRETS` | 7 | 无密钥——错口令的关键原因码 |
| `REASON_SUPPLICANT_DISCONNECT` / `TIMEOUT` | 8 / 11 | supplicant 断开/超时 |
| `REASON_SSID_NOT_FOUND` | 53 | 找不到 SSID |
| `REASON_IP_CONFIG_UNAVAILABLE` | 5 | 拿不到 IP 配置 |
| `SEC_KEY_MGMT_PSK` / `802_1X` / `SAE` | 0x100 / 0x200 / 0x400 | AP 安全标志位 |
| `AP_FLAGS_PRIVACY` | 0x1 | AP 启用隐私位（用于识别 WEP） |

模块命名为 `ids` 而非 `nm`，是为避开 clippy 的 `module_inception`（避免 `nm::nm::...` 的重复）。

### 超时常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `CONNECT_TIMEOUT` | 45 秒 | 等待一次加入尘埃落定。关联很快，忙网络上 DHCP 才耗时；超时本身是一种可报告结果而非挂死 |
| `SCAN_WAIT` | 10 秒 | 等待请求的扫描完成的上限；`duckctl` 给扫描留 60 秒，此值必须远小于它 |
| `SCAN_POLL` | 250 毫秒 | 等待期间重读 `LastScan` 的频率 |

## 4. zbus proxy（D-Bus 接口声明）

| proxy | 接口 | 关键成员 |
| --- | --- | --- |
| `Manager` | `org.freedesktop.NetworkManager` | `GetDevices`；`AddAndActivateConnection(connection, device, specific_object)` 返回 `(connection_path, active_path)` |
| `Device` | `…Device` | 属性 `DeviceType`/`Interface`/`State`/`StateReason`/`Ip4Config`/`Ip6Config`；**信号 `StateChanged(new, old, reason)`** |
| `ActiveConnection` | `…Connection.Active` | 属性 `State`——观察「一次激活」而非设备 |
| `Wireless` | `…Device.Wireless` | `RequestScan`、`GetAllAccessPoints`；属性 `HwAddress`、`ActiveAccessPoint`、`LastScan` |
| `AccessPoint` | `…AccessPoint` | 属性 `Ssid`(字节)/`Strength`/`Flags`/`RsnFlags`/`WpaFlags` |
| `Ip4Config` / `Ip6Config` | …IP4/6Config | 属性 `AddressData` |
| `Settings` | `…Settings` | `ListConnections` |
| `Connection` | `…Settings.Connection` | `GetSettings`、`Delete` |

`Device::device_state_changed` 的命名经过刻意安排：属性 `state` 已生成 `receive_state_changed`，信号若同名会冲突。注释强调：**激活失败后再读 `StateReason` 属性只会得到 0**（此时 NM 已把设备转回自动连接旧配置），唯一可靠的错口令来源是实时信号。

## 5. `struct NetworkManager` 与内部方法

仅含 `bus: zbus::Connection`；`new()` 连接**系统总线**（连不上只代表 D-Bus 不可达，不代表没有 NM）。

- `wifi_device() -> NetResult<Option<OwnedObjectPath>>`：枚举设备取第一个 Wi-Fi。**`None` 是真实答案而非错误**：仍跑 netplan 的板上 NM 不管任何 Wi-Fi，应报 `Unavailable`（可提示 `scripts/migrate-network.sh`）而非失败。
- `first_address(addresses)`：从 `AddressData` 取首个 `address` 字符串。
- `saved_connections(ssid) -> Vec<path>`：**刻意返回复数**。NM 允许重复连接 id，「这个 SSID 的配置文件」并不唯一存在。曾返回 `Option` 的错误很实际：`net.forget` 只删了两份中的一份就报成功，残留带旧口令的配置稍后仍会被自动连接。SSID 在 `802-11-wireless.ssid` 下以**原始字节**存放（SSID 不保证 UTF-8），此处按用户输入的文本比较。
- `saved_ssids() -> HashSet<String>`：一次遍历拿到所有已存 SSID 集合。旧实现曾在 `scan` 中对每个 AP 调一次 `saved_connections`，导致 N 倍枚举；扫描本就是最慢且客户端同步等待的调用。
- `delete_saved(ssid) -> usize`：删除该 SSID 的**每一份**配置并返回删除数。
- `fn bus_err(e) -> String`：统一包装为 `NetworkManager D-Bus call failed: {e}`。

## 6. 两个纯映射函数

### `fn security_of(flags, rsn, wpa) -> proto::Security`

按优先级：`802_1X` → `Enterprise`；`SAE` → `Wpa3Sae`；`PSK` → `WpaPsk`；仅有隐私位 → `Wep`（提示「老到无法加入」而非隐晦失败）；否则 `Open`。WPA2/WPA3 过渡 AP 同时通告 PSK+SAE 时 SAE 胜出；Enterprise 压过一切。

### `fn failure_of(state, reason) -> (ConnectFailure, Option<String>)`

- `NO_SECRETS(7)` → `BadKey`（选型 NM 的核心回报：独特、可操作）；
- `SSID_NOT_FOUND(53)` → `NotFound`；
- `SUPPLICANT_TIMEOUT/DISCONNECT`、`IP_CONFIG_UNAVAILABLE` → `Timeout`；
- 其他 → `Other`（绝不能把未知原因伪装成错口令，否则用户会反复重输本来正确的密钥）。
- 同时返回携带原始数值的 detail（`NetworkManager state {state}, reason {reason}`），供支持工单使用，但永远不是用户看到的主信息。

## 7. `impl Net for NetworkManager`

### `status`

无 Wi-Fi 设备 → 全空字段的 `Unavailable`。否则读设备状态映射：`ACTIVATED→Connected`、`UNAVAILABLE→Unavailable`、`DISCONNECTED|FAILED→Disconnected`、中间态一律 `Connecting`（客户端应轮询而非下定论）。再从 `ActiveAccessPoint`（路径为 `/` 视为无）取 SSID/信号（SSID 非 UTF-8 则略过），从 `Ip4Config`/`Ip6Config` 取首个地址，并附 `HwAddress` 与接口名。

### `scan`

无设备 → 空列表。关键在**等扫描真正完成再读列表**：

- `RequestScan` 一被受理就返回，无线电扫频尚未结束；且 NM 会修剪久未见到的 AP，关联中缓存常只剩当前 AP——旧代码读得太快，第一次调用列出 1 个、完全相同的第二次列出 8 个。对一个专用来「到新地方挑网络」的客户端，「问两次」不可接受。
- 以 `LastScan`（毫秒，CLOCK_BOOTTIME；无则 -1）变化作为完成信号，轮询至 `SCAN_WAIT`。扫描被限速/拒绝不是错误：NM 在刚扫过时拒绝，恰恰说明缓存新鲜，直接读列表即可。属性中途消失则带缓存返回。
- 然后一次取 `saved_ssids`，枚举所有 AP：跳过无/空 SSID（隐藏网络与非 UTF-8 SSID 无法在 JSON 中命名）；计算 security、信号、saved 标记；**同一 SSID 只留一条、信号最强者胜**（mesh/双频路由会呈现多个 AP）；最后按信号降序。

### `connect`（最复杂，处处是历史坑）

1. 无 Wi-Fi 设备 → `Failed { Unsupported, "…may still be on netplan" }`。
2. **改动任何东西之前先拒绝做不到的**：先 `scan()` 在结果中找 SSID——企业网 → `Unsupported`（缺证书流程）；非开放网但未给口令 → `Unsupported`；找不到 SSID → `NotFound`（带可操作文案；隐藏 SSID 也在此被拒，因为 API 还没有 `hidden` 形态）。这一检查直到扫描会等待完成后才可信。
3. **先删除该 SSID 已有配置再添加**，实现「重配网幂等」：`AddAndActivateConnection` 总会*添加*且 NM 允许同名双配置，手机上输错一次、再改对会留下两份，重启后自动连哪份无保证。删除失败只 `warn` 不致命（残留是下一次启动的问题，拒绝当下上网是眼前的问题）。若删的是活动配置会断开机器人——改密钥本就要重新关联，BLE 客户端按设计不受影响。
4. 组装设置字典：`connection`（id/type）、`802-11-wireless`（ssid 字节、`mode=infrastructure`）、必要时 `802-11-wireless-security`（`key-mgmt=wpa-psk`、psk；`wpa-psk` 覆盖 WPA2 与 WPA2/WPA3 过渡；纯 WPA3 理论上要 `sae`，留待板子验证后改由扫描的 Security 决定）。`autoconnect` 默认开——这正是机器人重启后自行重入网、`configd` 完全不参与重连业务的属性。
5. **在激活开始前**订阅设备 `StateChanged` 信号，否则携带原因的迁移已经错过。
6. 调 `AddAndActivateConnection(settings, device, "/")`；调用本身失败 → `Failed { Other, detail }`。
7. **观察 ActiveConnection 而非设备**（核心修复）：设备在已连网络上保持 `ACTIVATED`，新激活可能在它身旁失败；旧代码轮询设备状态，曾对 `connect("Tehaupoo","lol")` 报成功、名字却是机器人一直连着的 `SFR-e994`——谎报成功会让手机误以为已完成配网。循环中：
   - `ACTIVE_ACTIVATED` → 再调 `status()` 拿 IP，但**返回请求的 SSID**（而非 status 报告的名字）；
   - `DEACTIVATED/DEACTIVATING` 或到达 45 秒 deadline → 取原因：优先用实时观察到的 `observed_reason`，否则回退读 `StateReason` 属性（通常已为 0）；经 `failure_of` 映射，超时则覆盖为 `Timeout`；
   - 失败时**删除本次 `added` 配置**（NM 即使激活失败也保留 AddAndActivate 添加的配置，否则错口令会留下一份被自动连接永远重试、并让 `net.status` 误报 saved 的配置）；
   - 用 `tokio::select!` 同时等 500ms 轮询与下一个信号：信号到达 `FAILED` 或 `NEED_AUTH` 且 reason 非 0 时记录 `observed_reason`（错口令会先经过 `NEED_AUTH/NO_SECRETS`，稍后才到 FAILED 且原因可能被清空）。

### `forget`

`delete_saved(ssid)`，`removed = deleted > 0`。

## 8. 单元测试说明（均为无需总线的纯函数测试）

| 测试 | 验证内容 |
| --- | --- |
| `security_flags_map_to_what_a_client_must_ask_for` | 开放/WEP/PSK（rsn 或 wpa 任一）/SAE/802.1X 映射；PSK+SAE 过渡取 SAE；PSK+802.1X 取 Enterprise |
| `a_rejected_key_is_reported_as_bad_key` | reason 7 必须是 `BadKey`，detail 含 `reason 7`；53→NotFound；11→Timeout；未知 999→`Other`（不得让用户围着正确密钥反复重输） |

## 9. 要点小结

- 用 zbus 系统总线实现 `Net`；每个「为什么」都对应一个真机上踩过、被注释记录的坑。
- 错口令（reason 7）经 `StateChanged`/`NEED_AUTH` 实时信号捕获，事后读属性只会得到 0。
- 判定成功看 **ActiveConnection** 且返回请求的 SSID；失败必清理 AddAndActivate 残留配置。
- 重配网先删后加保证幂等；扫描等 `LastScan` 变化再读；同名 AP 取最强信号。
- 企业网、隐藏 SSID、缺失口令在改动前明确拒绝；代码尚未对真机 NM 验证。
