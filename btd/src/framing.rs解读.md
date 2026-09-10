# `framing.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 216 行 |
| 角色 | 将 GATT characteristic 变成面向行的字节管道——分块与重组 |
| 平台 | 跨平台（纯逻辑） |

## 二、核心问题：BLE 是唯一消息不适合单次写入的传输

- 控制平面是 NDJSON——每行一个 JSON 对象——在每个传输上（architecture.md §2.2）。
- BLE 是唯一消息不适合单次写入的：可用有效载荷是 `ATT_MTU - 3`，在从不重新协商的手机上是 20 字节，在好的链接上是几百字节。
- 因此一行分块到达，分块离开。

## 三、没有帧头：换行符就是帧定界符

### 为什么安全

- **没有帧头。** 已经分隔消息的换行符在两个方向上都是帧定界符。
- 这是安全的而非幸运的：`serde_json` 将字符串内的换行符转义为 `\n`，因此原始 `0x0A` 从不出现在序列化的 JSON 对象内——这与使 NDJSON 在 unix socket 上工作的属性相同。

### 为什么不添加长度前缀

- 添加长度前缀意味着第二个只有 BLE 说的帧方言，每个客户端都必须实现它。
- 手机应用可以改为完全做 `robotctl` 做的事：写字节，读到换行。

## 四、常量

### `MAX_LINE = 8 * 1024`（8 KiB）

- 在放弃对等方之前我们会重组的最长行。
- 从不发送换行符的 BLE 客户端会否则无界增长此缓冲区，并且范围内任何人都可达。
- 对任何真实请求都慷慨——最大的是带长 ref 的 `update.apply`——远低于 `updaterd` 自己的 1 MiB 行限制，因为那么大的东西没有任何理由通过 BLE 到达。

## 五、`FramingError` 枚举

```rust
pub enum FramingError {
    LineTooLong,  // MAX_LINE 内没有换行符
    NotUtf8,      // 不是有效 UTF-8，因此也不能是 JSON
}
```

- 两种情况都意味着"丢弃连接"，但记录不同：
  - 一个是无法成帧的客户端
  - 另一个可能是攻击
- 实现 `Display` 和 `Error` trait（真实错误，不只是 Debug 枚举：此 crate 是库，使用它的客户端——`duckctl` 示例是第一个——想要 `?` 工作）。

## 六、`Reassembler` 结构体

```rust
pub struct Reassembler {
    buf: Vec<u8>,
}
```

将入站块重组为完整行。

### `push(chunk: &[u8]) -> Result<Vec<String>, FramingError>`

喂入一个块；返回它完成的每个完整行。

- 返回 `Vec` 因为一次写入可以合法地携带几个短行——将 `hello` 和 `update.status` 批处理成一次 40 字节写入的客户端是高效的，不是错误的。

#### 算法

1. **溢出检查**：如果 `buf.len() + chunk.len() > MAX_LINE`，清除缓冲区（而非保留部分行：后面的无论如何都不可解析，持有它会让对等方钉住内存），返回 `LineTooLong`。
2. 扩展缓冲区。
3. 循环查找换行符：
   - 取出到换行符（含）的行。
   - 去掉换行符，容忍 CRLF（客户端好心加的）。
   - 空行跳过（与 `robotd` 和 `updaterd` 在 socket 上做的一致）。
   - UTF-8 验证：失败则清除缓冲区并返回 `NotUtf8`。
4. 返回所有完整行。

### `pending() -> usize`

等待换行符的字节数。用于记录已在行中间安静下来的对等方。

## 七、`chunks(line: &str, mtu: usize) -> Vec<Vec<u8>>`

将一个出站行拆分为通知大小的块，含换行符。

- 换行符是有效载荷的一部分而非单独的最终块：重组到换行符的客户端不需要超出它已有的"消息结束"概念。

### 算法

1. `mtu = mtu.max(20)`：零或荒谬的 MTU 会除零或每字节发一个块。20 是 BLE 在任何协商前保证的下限。
2. 将行转为字节，如果不以 `\n` 结尾则追加。
3. 按 `mtu` 分块。

## 八、测试（11 个）

| 测试 | 验证内容 |
|---|---|
| `a_line_split_across_chunks_reassembles` | 跨块拆分的行正确重组 |
| `several_lines_in_one_chunk_all_come_back` | 一次写入中的多行全部返回 |
| `a_partial_trailing_line_is_retained` | 尾部部分行被保留（是下一条消息的开始） |
| `blank_lines_are_skipped` | 空行被忽略而非作为空请求转发 |
| `crlf_is_tolerated` | CRLF 被容忍 |
| `a_line_without_a_newline_is_refused_at_the_cap` | 无换行符的行在上限被拒绝，且缓冲区被释放（对等方不能通过重试钉住内存） |
| `invalid_utf8_is_refused` | 无效 UTF-8 被拒绝 |
| `chunking_round_trips_at_every_mtu` | 分块和重组互为逆——在从 BLE 下限到大于消息的 MTU 上验证 |
| `chunks_terminate_with_exactly_one_newline` | 换行符在缺席时追加，在存在时从不加倍（否则客户端在每条消息之间看到空行） |
| `an_absurd_mtu_falls_back_to_the_ble_floor` | 退化 MTU（0）回退到 BLE 下限，不 panic 或每字节一块 |
#（注：内容由AI生成）
