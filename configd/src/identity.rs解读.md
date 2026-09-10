# `identity.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 164 行 |
| 角色 | 机器人身份——从 SoC 序列号派生的稳定名称与机器 ID |
| 平台 | 跨平台（Linux 上读 `/sys`，非 Linux 回退） |

## 二、核心设计：身份从硬件派生而非配置

### 为什么从 SoC 序列号派生

- 机器人需要一个**稳定的、每台不同的名称**，用于蓝牙广告、日志和网络标识。
- 从 SoC 序列号派生意味着：
  - **无需出厂配置**：每块板自动获得唯一身份。
  - **不可变**：重刷系统不改变身份（序列号在硬件中）。
  - **可复现**：同一序列号总是产生同一名称。

### 为什么不直接用序列号

- 序列号是长十六进制字符串（如 `0a1b2c3d4e5f`），不适合作为蓝牙名称或主机名。
- 派生为短名称（`duck-XXXX`）更可读，且只暴露序列号的哈希而非完整值。

## 三、常量

| 常量 | 值 | 说明 |
|---|---|---|
| `PREFIX` | `"duck-"` | 名称前缀，所有机器人共享 |
| `NAME_LEN` | 4 | 后缀长度（十六进制字符），共 16 位熵 |
| `SERIAL_PATH` | `"/sys/class/efuse/public_id"` | Radxa Zero 3W 的 SoC 序列号路径（eFuse 公共 ID） |
| `FALLBACK_SERIAL` | `"/proc/device-tree/serial-number"` | 设备树回退路径（其他板型） |
| `FALLBACK_NAME` | `"duck-robot"` | 无法读取序列号时的默认名称 |

## 四、`Identity` 结构体

```rust
pub struct Identity {
    pub name: String,      // 派生名称，如 "duck-5b21"
    pub serial: String,    // 原始 SoC 序列号
    pub machine_id: String, // 从序列号派生的机器 ID（用于 systemd 等）
}
```

### `Identity::load() -> Self`

- 主入口：从硬件加载身份。
- 流程：
  1. 尝试读 `SERIAL_PATH`（`/sys/class/efuse/public_id`）。
  2. 失败则尝试 `FALLBACK_SERIAL`（`/proc/device-tree/serial-number`）。
  3. 都失败则使用 `FALLBACK_NAME`，serial 为空。
  4. 从序列号派生名称和机器 ID。

### `Identity::from_serial(serial: &str) -> Self`

- 从已知序列号构造身份（用于测试和非 Linux 平台）。
- 派生算法：
  - **名称**：`PREFIX` + 序列号 SHA-256 哈希的前 `NAME_LEN` 个十六进制字符。
  - **机器 ID**：序列号 SHA-256 哈希的完整 32 字节十六进制（64 字符），符合 systemd machine-id 格式。

### 为什么用 SHA-256

- 单向：从名称无法反推完整序列号。
- 均匀分布：名称后缀在 16 位空间中均匀分布，碰撞概率极低（2^16 = 65536，对消费级机器人足够）。
- 确定性：同一序列号总是产生同一名称。

## 五、平台特定实现

### Linux

- `read_serial()` 尝试两个路径：
  - `/sys/class/efuse/public_id`（Radxa Zero 3W 主路径）
  - `/proc/device-tree/serial-number`（设备树回退）
- 读取后去除空白和 NUL 字节（设备树文件以 NUL 结尾）。

### 非 Linux

- `read_serial()` 返回 `None`。
- 使用 `FALLBACK_NAME`。
- 这使得 crate 可在 macOS 上构建和测试（入职路径）。

## 六、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `bluez.rs` | 广告名称使用 `identity.name` |
| `main.rs` | 启动时加载身份，传给各子系统 |
| `store.rs` | 配置存储可能使用 `machine_id` 作为命名空间 |
| `btd` | 通过 `system.info` 暴露名称给手机应用 |
| `updaterd` | 可能使用 `machine_id` 标识设备 |
| `systemd` | `machine_id` 可写入 `/etc/machine-id` |

## 七、测试

| 测试 | 验证内容 |
|---|---|
| `name_is_deterministic` | 同一序列号总是产生同一名称 |
| `different_serials_produce_different_names` | 不同序列号产生不同名称（碰撞概率极低） |
| `name_has_correct_prefix_and_length` | 名称格式正确：`duck-` + 4 个十六进制字符 |
| `machine_id_is_64_hex_chars` | 机器 ID 是 64 个十六进制字符（32 字节） |
| `empty_serial_falls_back` | 空序列号回退到默认名称 |
| `from_serial_matches_load_on_known_serial` | `from_serial` 与 `load` 在已知序列号上一致 |
#（注：内容由AI生成）
