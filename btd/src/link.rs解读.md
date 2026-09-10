# `link.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 84 行 |
| 角色 | 无线电与所有值得测试的东西之间的**接缝** |
| 平台 | 跨平台 |

## 二、设计决策：故意不是 trait

### 为什么不用 `GattLink` trait

- 一个 `GattLink` trait 需要异步 `recv` 和异步 `send`。
- 会话循环必须同时等待两者——这意味着要么将 link 拆成带有关联类型的两半，要么在 `select!` 中与借用检查器搏斗。
- **两个 channel 和一个普通 struct** 表达同样的东西，没有这些麻烦。
- 测试通过手工构造一个，而非实现任何东西。

### 后端的全部工作

1. 接受连接
2. 将入站块送入 `inbound`
3. 将 `outbound` 上出现的任何内容写回中心
4. 中心离开时丢弃 channel

这两个 channel 之间发生的是 `session`，它永远不知道是否涉及无线电。

## 三、常量

### `QUEUE = 512`

任一方向可排队的块数。

#### 为什么是 512

- 大小设计为**最大入试行永远不必阻塞**，这是正确性要求而非调优选择。
- 无线电后端必须无需等待就交出块：BlueZ 将每次写入作为自己的任务分发，因此在接收块和入队之间的任何让步点都是两个块交换位置的机会。
- **重排的块不会失败——它会重组为解析为错误内容的东西。**
  - 示例：3 块中的第 2 块最后到达，产生 `{"id":1,"jsonrpc":"2.info","params":{}}`——有效的 JSON，缺少字段，解析错误归咎于客户端。
- 因此队列必须足够深，使得同步 `try_send` 在合法流量上不会失败：
  - `QUEUE * 20 >= framing::MAX_LINE`，其中 20 是 BLE 保证的最小有效载荷。
  - 512 * 20 = 10240 >= 8192（MAX_LINE），满足。
- 超出此范围，洪水得到干净的 ATT 错误而非丢弃的块，因为失败写入是可恢复的，损坏一个则不是。

#### 编译时断言

```rust
const _: () = assert!(
    QUEUE * 20 >= crate::framing::MAX_LINE,
    "QUEUE * 20 must be at least framing::MAX_LINE, ..."
);
```

- 编译时检查而非测试，因为这是两个常量之间的关系，没有什么能在运行时为真而在构建时为假。
- 20 字节是每个 BLE 链路必须支持的有效载荷，因此是客户端可能使用的最小块。

## 四、`Link` 结构体

```rust
pub struct Link {
    pub inbound: mpsc::Receiver<Vec<u8>>,
    pub outbound: mpsc::Sender<Vec<u8>>,
    pub mtu: usize,
    pub peer: String,
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `inbound` | `mpsc::Receiver<Vec<u8>>` | 中心写入 `request` characteristic 的块，按到达顺序。断开时关闭 |
| `outbound` | `mpsc::Sender<Vec<u8>>` | 要在 `response` characteristic 上通知的块 |
| `mtu` | `usize` | 可用通知有效载荷——`ATT_MTU - 3`——按此连接协商。每会话读一次而非每消息，因为中心可能重新协商但不会在行中途，重新读取会让一行被两种方式分块 |
| `peer` | `String` | 中心地址，用于日志行。**从不用于授权**——BLE 地址可轻易伪造，配对才是授权（architecture.md §4.2） |

## 五、`Link::pair` 构造函数

```rust
pub fn pair(
    mtu: usize,
    peer: impl Into<String>,
) -> (Self, mpsc::Sender<Vec<u8>>, mpsc::Receiver<Vec<u8>>)
```

- 返回 `(Link, to_robot, from_robot)`。
- `to_robot`：测试/后端用来向机器人发送入站块。
- `from_robot`：测试/后端用来接收机器人的出站块。
- 用于测试和 `--fake` 模式。
- 两个 channel 都使用 `QUEUE` 容量。
#（注：内容由AI生成）
