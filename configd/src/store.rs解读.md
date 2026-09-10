# `store.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 约 200 行 |
| 角色 | 配置存储——持久化键值对到文件 |
| 平台 | 跨平台 |

## 二、核心设计

### 为什么需要独立存储模块

- configd 需要持久化一些配置（如机器人名称、配对 PIN、wifi 凭据引用等）。
- 这些配置需要：
  - 跨重启持久化。
  - 原子写入（防止崩溃导致损坏）。
  - 简单的键值接口。

### 为什么不用数据库

- 配置量小（几十个键值对）。
- 不需要查询语言。
- 文件 + JSON 足够，且易于调试和备份。

## 三、`Store` 结构体

```rust
pub struct Store {
    path: PathBuf,
    data: std::collections::BTreeMap<String, serde_json::Value>,
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `path` | `PathBuf` | 存储文件路径 |
| `data` | `BTreeMap<String, Value>` | 内存中的键值对，BTreeMap 保证排序（输出稳定） |

### 为什么用 BTreeMap 而非 HashMap

- BTreeMap 按 key 排序，序列化输出稳定。
- 稳定输出意味着：
  - 文件 diff 可读。
  - 相同数据产生相同文件（可哈希、可验证）。
  - 调试时易于查找。

## 四、核心方法

### `Store::load(path: &Path) -> Result<Self, String>`

- 从文件加载存储。
- 如果文件不存在，返回空存储（不报错）。
- 如果文件存在但解析失败，返回错误（不静默忽略损坏数据）。

### `Store::save(&self) -> Result<(), String>`

- 原子写入：
  1. 写入临时文件（同目录，`.tmp` 后缀）。
  2. `fsync` 确保数据落盘。
  3. `rename` 原子替换原文件。
- 原子性保证：崩溃时要么是旧文件，要么是新文件，不会是半写文件。

### `get(&self, key: &str) -> Option<&Value>`

- 读取键值。

### `set(&mut self, key: &str, value: Value)`

- 设置键值。
- 调用后需手动 `save()` 持久化（批量修改时只写一次）。

### `remove(&mut self, key: &str) -> Option<Value>`

- 删除键值，返回旧值。

### `keys(&self) -> impl Iterator<Item = &String>`

- 遍历所有键（按排序顺序）。

## 五、类型化访问辅助

### 为什么需要类型化方法

- 直接操作 `serde_json::Value` 容易出错（类型不匹配）。
- 提供类型化方法减少运行时错误。

### 常见方法

| 方法 | 说明 |
|---|---|
| `get_string(key) -> Option<&str>` | 读取字符串 |
| `set_string(key, value)` | 设置字符串 |
| `get_bool(key) -> Option<bool>` | 读取布尔值 |
| `set_bool(key, value)` | 设置布尔值 |
| `get_u64(key) -> Option<u64>` | 读取无符号整数 |

## 六、存储的键

### 已知键（根据项目上下文）

| 键 | 类型 | 说明 |
|---|---|---|
| `name` | string | 机器人名称（覆盖 identity 派生的默认名） |
| `pairing_pin` | string | 配对 PIN（出厂默认 `000000`） |
| `wifi_ssid` | string | 已配置 wifi 的 SSID（引用 NetworkManager 连接） |
| `dev_keys_allowed` | bool | 是否允许开发密钥（仅 dev board） |

### 键命名约定

- 小写蛇形（snake_case）。
- 不带前缀（存储本身就是命名空间）。

## 七、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `main.rs` | 启动时加载存储，传给各子系统 |
| `identity.rs` | `name` 键覆盖 identity 派生的默认名 |
| `net.rs`/`nm.rs` | wifi 配置可能引用存储中的连接名 |
| `bluez.rs` | 配对 PIN 可能从存储读取 |
| `btd` | 通过 `system.setName` 等修改存储 |
| `updaterd` | 更新时可能迁移存储格式 |

## 八、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **原子持久化** | 临时文件 + fsync + rename，崩溃不损坏 |
| **稳定输出** | BTreeMap 排序，文件 diff 可读 |
| **简单接口** | 键值对，无需数据库 |
| **批量优化** | set 不自动 save，批量修改只写一次 |
| **类型安全** | 提供类型化访问方法，减少 Value 操作错误 |
| **容错加载** | 文件不存在返回空存储，解析失败才报错 |
#（注：内容由AI生成）
