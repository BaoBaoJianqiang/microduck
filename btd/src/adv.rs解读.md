# `adv.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 126 行 |
| 角色 | 广播中除名称外携带的内容：机器人的 IPv4 地址 |
| 平台 | 跨平台（线协议契约，与 `gatt.rs` 同理不放在 `bluez.rs`） |

## 二、为什么广播地址而非调用

- `net.status` 已经报告地址，但读取它需要连接、绑定和 PIN——每台机器人都要。
- `duckctl scan` 故意不连接任何东西，这就是它成为机器人不可达时要使用的命令的原因，因此列表只能报告广播携带的内容。
- 广播地址因此是 `scan` 能回答"我该 ssh 到哪里"的唯一方式，而这是列表最常被读取来回答的问题。

## 三、为什么是 4 字节且不再多

### 传统广播 31 字节预算

| 字段 | 字节 |
|---|---|
| flags | 3 |
| 128-bit service UUID | 2 + 16 = 18 |
| manufacturer-data 头部 | 2（AD 头）+ 2（company id）= 4 |
| **已用** | **25** |
| **剩余给地址** | **6**，本实现用 **4** |

- 名称不在此预算中——BlueZ 将 Local Name 放在扫描响应中，扫描响应有自己的 31 字节。
- SSID 不在这里：SSID 本身最多 32 字节，任何版本都放不下。它保持为 `wifi status` 问题。

## 四、为什么地址总是存在

- 没有 wifi 的机器人广播 `Ipv4Addr::UNSPECIFIED`（`0.0.0.0`）而非丢弃字段。
- 字段本身就是证据：
  - **缺席** = 运行早于此版本的机器人
  - **存在且为零** = 没有地址的机器人
  - 这两者需要不同的下一步。
- 丢弃字段会把它们合并成一个空白列。

## 五、常量与函数

### `COMPANY_ID = 0xFFFF`

- 蓝牙 SIG 保留用于内部和互操作性测试的 ID。
- 对于未被分配 ID 的项目是正确选择。
- **任何人都可以使用它，因此这不是身份检查**：`address_in` 只被询问已经广播了 `gatt::SERVICE_UUID` 的设备，那才是区分器。

### `address_data(address: Option<Ipv4Addr>) -> Vec<u8>`

- 机器人地址进入广播的形式。
- `None`（无 wifi，或 configd 不肯说）→ `Ipv4Addr::UNSPECIFIED` 而非缺席字段。

### `address_in(manufacturer_data: &HashMap<u16, Vec<u8>>) -> Option<Ipv4Addr>`

- 扫描报告的地址（如果报告了）。
- `None` 涵盖三种情况，此函数不区分（因为它不能）：
  1. 根本没有字段
  2. 字段长度错误
  3. 字段说 `0.0.0.0`
- 调用者有广播，可以区分第一种和第三种（见 `duckctl` 的列表）。

### `has_address_field(manufacturer_data: &HashMap<u16, Vec<u8>>) -> bool`

- 此设备是否广播了地址字段（无论内容如何）。
- 与 `address_in` 分离，因为"较旧版本，什么都不广播"和"不在 wifi 上的机器人"是空白地址可能意味着的两件事，无法区分它们的列表会把读者送到错误的地方检查。

## 六、测试

| 测试 | 验证内容 |
|---|---|
| `an_address_survives_the_advertisement` | 地址往返：编码→解码一致 |
| `no_wifi_is_a_present_field_and_no_address` | 无 wifi = 存在字段但无地址；与完全无字段区分 |
| `a_payload_of_the_wrong_length_is_not_an_address` | 错误长度（0/1/3/5/16 字节）不是地址 |
| `the_payload_fits_the_budget` | 有效载荷 fits 31 字节广播预算（编译时断言的运行时对应） |
#（注：内容由AI生成）
