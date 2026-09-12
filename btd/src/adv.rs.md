# `adv.rs` 文件解析 —— 广播报文中的 IPv4 地址字段

## 1. 文件定位

- 路径：`src/adv.rs`
- 角色：定义广播报文（advertisement）除名字之外携带的内容——机器人的 IPv4 地址。
- 平台无关。放在这里而不是 `bluez.rs`，原因与 `gatt.rs` 相同：它是**线路契约（wire contract）**。机器人端用 `address_data` 编码，笔记本上的客户端 `duckctl` 用 `address_in` 解码，两端共享同一份代码，因而不可能在字节布局上产生分歧。

## 2. 模块文档中的三个关键设计决策

### 2.1 为什么放进广播而不是通过一次调用获取

`net.status` 已经能返回地址，但读取它需要：建立连接、完成绑定（bond）、输入 PIN——而且是每台机器人一次。

`duckctl scan` 刻意不连接任何设备，这使它成为"机器人连不上时"首先使用的命令。因此列表只能报告广播报文里自带的信息。广播地址是 `scan` 能回答"我该往哪里 ssh？"的唯一途径，而这正是列表最常被用来回答的问题。

### 2.2 为什么只用 4 字节、不能更多

- 传统（legacy）广播报文只有 **31 字节**；
- `btd` 已用掉 21 字节：flags（3）+ 128 位服务 UUID（2 + 16）；
- 一个 manufacturer-data 字段头部占 2 字节、公司 id 占 2 字节，载荷只剩 6 字节，本字段使用其中 4 字节；
- 名字不在这份预算内——BlueZ 把 Local Name 放在 scan response 中，那是另外独立的 31 字节；
- SSID 单条最长就 32 字节，任何形式都放不下，所以 SSID 只能通过 `wifi status` 查询。

### 2.3 为什么地址字段始终存在

没有 wifi 的机器人广播 `0.0.0.0`（`Ipv4Addr::UNSPECIFIED`），而不是丢弃该字段。字段本身即信息：

- 字段缺失：运行的是早于此特性的版本；
- 字段存在且为零：机器人在运行但没有地址；
- 两种情况需要不同的后续排查动作，丢弃字段会把它们混淆成同一列空白。

## 3. 公共项详解

### 3.1 常量 `COMPANY_ID: u16 = 0xFFFF`

- 载荷登记在该公司 id 之下；
- `0xFFFF` 是蓝牙 SIG 为内部/互操作测试保留的 id，适合尚未被分配正式 id 的项目；
- 别人也可能使用它，因此**它不是身份校验**。`address_in` 只会被用于已经广播了 `gatt::SERVICE_UUID` 的设备，服务 UUID 才是真正的判别依据。

### 3.2 `address_data(address: Option<Ipv4Addr>) -> Vec<u8>`

- 把地址编码进广播：取 4 字节 octets；
- `None`（没有 wifi，或 `configd` 不肯说）编码为 `0.0.0.0`，而非缺字段，理由见 2.3。

### 3.3 `address_in(manufacturer_data: &HashMap<u16, Vec<u8>>) -> Option<Ipv4Addr>`

- 从扫描到的 manufacturer data 中解析地址；
- 取不到公司 id、字节数组无法转成 `[u8; 4]` 时返回 `None`；
- 解析出 `0.0.0.0` 时也返回 `None`（用 `filter(!is_unspecified)`）；
- `None` 实际涵盖三种本函数无法区分的情形：完全无字段、字段长度错误、字段为 `0.0.0.0`。区分前两种与第三种由持有完整广播报文的调用方（`duckctl` 列表）完成。

### 3.4 `has_address_field(manufacturer_data) -> bool`

- 判断设备是否**广播过**地址字段（不管值是什么）：存在公司 id 对应项且长度恰为 4；
- 与 `address_in` 分开，是因为"旧版本什么都不播"和"机器人没连 wifi"这两件事必须能被列表区分，否则会把读者引向错误的排查方向。

## 4. 单元测试（4 个）

| 测试 | 验证内容 |
| ---- | -------- |
| `an_address_survives_the_advertisement` | `192.168.1.42` 编码后再解码完全一致，且字段存在 |
| `no_wifi_is_a_present_field_and_no_address` | 无 wifi 时：`address_in` 为 `None` 但 `has_address_field` 为 `true`；空 map 则两者皆否 |
| `a_payload_of_the_wrong_length_is_not_an_address` | 长度 0/1/3/5/16 均不被识别为地址 |
| `the_payload_fits_the_budget` | 断言 flags + 服务 UUID + 厂商头 + 4 字节地址 ≤ 31 字节，载荷若想增大必须回来改这里 |

## 5. 本文件要点小结

1. 广播中用 4 字节 manufacturer data（公司 id `0xFFFF`）携带 IPv4 地址；
2. 编码端（机器人）与解码端（`duckctl`）共享代码，杜绝布局分歧；
3. 无 wifi 广播 `0.0.0.0` 而非丢字段，以区分"旧版本"与"无网络"；
4. 严格的 31 字节预算由测试锁定；
5. 公司 id 不做身份判别，服务 UUID 才是。
