# `gatt.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 52 行 |
| 角色 | GATT 线协议契约——客户端必须知道的 UUID |
| 平台 | **跨平台**（故意不放在 `bluez.rs`） |

## 二、为什么放在这里而非 `bluez.rs`

这些 UUID 是**线协议契约**的一部分，就像 `duck-ipc-proto` 中的方法名一样：
- 机器人服务它们，每个客户端都必须查找它们。
- 无法为 Linux 编译的客户端（如笔记本上的 `duckctl`）仍然需要它们。
- 因此放在跨平台的 `gatt.rs`，而非 Linux-only 的 `bluez.rs`。

## 三、UUID 设计

### 随机 v4 UUID

- 使用随机 v4 UUID，而非任何派生值。
- 它们是我们的，一旦应用针对它们发布就**不能更改**。
- 完整写出，以便 grep 某个值能找到此注释。

### 两个 UUID

| 常量 | UUID | 用途 |
|---|---|---|
| `SERVICE_UUID` | `6f5d2a10-3b47-4c8e-9a1f-2d7e8c4b6019` | 机器人服务，客户端扫描的目标 |
| `RPC_UUID` | `6f5d2a11-3b47-4c8e-9a1f-2d7e8c4b6019` | RPC 管道：读一次获取 API 版本，写 NDJSON 请求，订阅获取响应 |

- 两个 UUID 仅在第 4 段末位不同（`...6019` vs `...6019` 的前一位 `10` vs `11`），便于识别为同一系列。

## 四、一个 characteristic，双向

### 设计决策

客户端**写入**请求到 `RPC_UUID`，并**订阅**同一个 characteristic 获取响应。

### 为什么不是两个 characteristic

- 更传统的形状是两个 characteristic：一个写、一个通知。
- 最初就是这样写的，但在这里更差，原因具体：
  - BlueZ 将写入和订阅报告为**独立事件**。
  - 用两个 characteristic，机器人必须通过设备地址将一个的写入半部分与另一个的通知半部分配对，猜测关联。
  - 用一个 characteristic，两个事件按构造都属于它，连接是真正的**双工流**。

### 代价

- 一个同时具有 `write` 和 `notify` 的 characteristic 是普通 BLE。
- 代价是在 nRF Connect 等通用浏览器中看起来略奇怪，同一行既是写又是通知。

## 五、RPC characteristic 的契约

### 读（Read）

- 读是契约的一部分，不是可选的好处。
- 它需要**已认证的加密链接**，因此它使中心在写入之前**先配对**。
- 订阅不需要加密，因此如果没有读，客户端订阅后第一次写入会被静默拒绝，（在 macOS 上）既看不到提示也看不到错误。
- 读还返回机器人的 `API_VERSION`，因此不匹配的客户端可以在发送任何内容之前说明。

### 写（Write）

- 向 `RPC_UUID` 写入 NDJSON 请求字节。
- 双向分块，由已分隔 NDJSON 消息的换行符定界（见 `framing` 模块）。

### 通知（Notify）

- 订阅 `RPC_UUID` 获取答案。
- 与写入共用同一个 characteristic。

## 六、测试

```rust
#[test]
fn the_uuids_are_distinct() {
    assert_ne!(SERVICE_UUID, RPC_UUID);
}
```

- 服务和 characteristic 必须不同。
- 两者一旦应用针对它们发布就被冻结。
#（注：内容由AI生成）
