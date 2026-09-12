# identity.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/identity.rs`
- **角色**：提供**每台设备的身份**（SoC 序列号）以及由它派生的**默认机器人名**。平台无关：在板上读 devicetree 属性，在笔记本上返回 `None` 由调用方回退。

## 2. 要解决的问题与关键抉择（模块文档）

- **房间里三个朋友各有三台机器人，必须能连到自己那台**。同一镜像刷出的板子过去都广播 `radxa-zero3`，手机无法区分；选错意味着把自家 wifi 凭据写进别人的机器人。因此机器人必须**在任何人改名之前**就可区分，需要一个每板唯一的东西来挂名字。
- **用 SoC 序列号，而不是蓝牙地址**。蓝牙地址看似自然（对端链路层本就可见，不泄露新信息），但实测会漂移：一块板在十六次启动间，`btd` 启动行先后记录了 `50:37:CD:16:2B:EC` 与 `50:37:CD:16:1B:92`，其间没有重刷。无法解释的身份变化不叫身份。（地址漂移本身还会让 `/var/lib/bluetooth/<address>/` 下的绑定全部失联，但那不是本模块的问题。）
- 序列号来自 **SoC 一次性可编程熔丝，经 bootloader 暴露**：抗重刷、抗更换无线电模块、无需任何预置步骤即可存在，因此手刷板子也能工作；而且**立即可读**——一个普通文件，不要 root、不要 D-Bus、不用等这块板上 `hci0` 出现所需的约 73 秒。

## 3. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `SERIAL_PATH` | `/proc/device-tree/serial-number` | bootloader 放置序列号的位置。选 devicetree 属性而非 `/proc/cpuinfo` 的 `Serial` 行：两者在板上报相同值（`bb7b734a7717ac41`），但前者是任何板子都可满足的通用绑定，后者是内核按架构选择打印方式的怪癖，未来换 SoC 更可能保留前者 |
| `NAME_PREFIX` | `duck` | 派生名前缀。硬编码而非取 hostname（否则会得到更长且无意义的 `radxa-zero3-7f3a`）；机器人是一只鸭子 |

## 4. 函数逐项解析

### `pub fn serial() -> Option<String>`

从默认路径 `SERIAL_PATH` 读取序列号，委托给 `serial_at`。

### `pub fn serial_at(path: &Path) -> Option<String>`

可对任意路径读取，便于离板测试。处理流程：

1. `std::fs::read(path)`，读不到直接 `None`（笔记本没有 `/proc/device-tree`）。
2. `String::from_utf8_lossy` 容错转字符串。
3. **按第一个 NUL 切分**：devicetree 属性以 NUL 结尾，且一个属性可能含多个字符串，所以第一个 NUL 是值的结束，而非待修剪的尾部垃圾。
4. `trim()` 后校验：空串、或含非「ASCII 可见图形字符」（`is_ascii_graphic`）的控制字符等，一律视为没有身份返回 `None`——因为该值还会经 `system.info` 原样送达客户端。

### `pub fn default_name(serial: &str) -> String`

生成形如 `duck-c51b` 的默认名：

- **哈希而非切片**：直接取序列号末四位等于假定两片芯片恰好差在那里，没有任何保证（厂商可让任意部分顺序化或恒定）；摘要把序列号中无论多少的熵扩散到整个输出。
- **用 SHA-256 而非 `std::hash::DefaultHasher`**：后者输出明确不保证跨 Rust 版本稳定，一次 `rustup update` 会悄悄改名现场所有机器人。
- 取摘要前两字节格式化为 `duck-{digest[0]:02x}{digest[1]:02x}`：**4 个十六进制字符 = 65 536 种可能**，房间里三台机器人约 22 000 次才撞一次。它是求「可区分」的默认值，不是唯一键；`system.setName`（手机）与 `robotctl system set-name`（预置）都可覆盖，撞名时改名就是逃生口。

## 5. 单元测试说明

| 测试 | 验证内容 |
| --- | --- |
| `a_nul_terminated_property_reads_as_the_value` | 板上真实形态 `bb7b734a7717ac41\0` 能正确读出去掉 NUL 的值 |
| `a_missing_property_is_no_serial` | 属性文件缺失返回 `None`（回退而非 panic） |
| `a_blank_or_unprintable_property_is_no_serial` | 空、仅 NUL、空白、控制字符等输入均为 `None`，避免此类板子全被命名成 `duck-e3b0` |
| `the_name_derived_from_this_boards_serial_is_pinned` | **钉死**：`default_name("bb7b734a7717ac41") == "duck-c51b"``，派生方式一旦改变在此失败，而不是在某人的蓝牙列表里出事 |
| `different_serials_give_different_names` | 相邻序列号（尾号 1/2）派生名不同 |
| `a_name_is_shaped_like_a_name` | 后缀恰为 4 位十六进制，且总长不超过 `store::MAX_NAME` |

## 6. 要点小结

- 身份取熔丝经 bootloader 暴露的 devicetree 序列号，抗重刷、免预置、立即可读。
- 拒绝使用会漂移的蓝牙地址作为身份。
- 默认名用跨版本稳定的 SHA-256 前两字节，格式 `duck-xxxx`，仅求默认可区分，改名是撞名逃生口。
- 真实板子序列号的派生结果 `duck-c51b` 被测试钉死。
