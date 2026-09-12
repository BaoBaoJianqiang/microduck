# `framing.rs` 文件解析 —— 把 GATT 特征值变成面向行的字节管道

## 1. 文件定位

- 路径：`src/framing.rs`
- 角色：纯逻辑模块。负责 BLE 分片（chunk）与 NDJSON 整行之间的双向转换——入站重组、出站切分。无任何蓝牙或异步依赖，最容易测试。

## 2. 模块设计要点

### 2.1 为什么需要分片

控制平面在所有传输上统一使用 NDJSON（每行一个 JSON 对象，见 `architecture.md` §2.2）。BLE 是唯一一条消息装不进一次写操作的传输：

- 可用载荷为 `ATT_MTU - 3`；
- 从不重新协商 MTU 的手机上只有 20 字节，良好链路上也只有几百字节；
- 因此一行请求要分多片到达，回复也要分多片发出。

### 2.2 没有帧头，换行符即分隔符

- 双向都用消息之间本来就存在的换行符 `\n` 作为帧分隔；
- 这是安全的而非侥幸：`serde_json` 会把字符串内部的换行转义成 `\n`，因此序列化后的 JSON 对象内部绝不会出现原始 `0x0A`——与 unix socket 上 NDJSON 成立的前提相同；
- 不加长度前缀，避免造出一种只有 BLE 讲的第二套帧方言（否则每个手机客户端都得实现）；
- 手机应用可以完全照搬 `robotctl` 的做法：写字节、读到换行为止。

## 3. 公共项详解

### 3.1 常量 `MAX_LINE: usize = 8 * 1024`

重组前允许的最长行，8 KiB。

- 防止从不发送换行的 BLE 客户端让缓冲区无限增长——而这种客户端无线电范围内谁都能充当；
- 相对真实请求足够宽松（最大的是带长 ref 的 `update.apply`）；
- 远低于 `updaterd` 自身的 1 MiB 行限制，因为那么大的数据不该走 BLE。

### 3.2 `struct Reassembler`

持有一个 `buf: Vec<u8>`，派生 `Debug`、`Default`。

- `new()`：委托给 `default()`；
- `push(chunk: &[u8]) -> Result<Vec<String>, FramingError>`：喂入一片，返回它补全的所有完整行。
  - 返回 `Vec` 是因为一次写操作合法地可以携带多条短行（客户端把 `hello` 和 `update.status` 打包进一次 40 字节写是高效而非错误）；
  - 若 `已有 + 新片 > MAX_LINE`：**清空**缓冲并返回 `LineTooLong`。清空而非保留残行，是因为后续内容已无法解析，且保留会让对端钉住内存；
  - 追加分片后循环查找 `\n`，逐条 `drain` 取出；
  - 去掉换行符，并容忍客户端多送的 CR（CRLF）；
  - 空行直接跳过；
  - 非 UTF-8：清空缓冲并返回 `NotUtf8`（不是 UTF-8 也就不可能是 JSON）；
- `pending() -> usize`：当前等待换行的字节数，用于在对端悄然断在半行时记日志。

### 3.3 枚举 `FramingError`

两个变体：`LineTooLong`（超出 `MAX_LINE` 仍无换行）、`NotUtf8`（非法 UTF-8）。

- 二者都意味着"丢弃连接"，但日志级别/含义不同：一个是不会分帧的客户端，另一个可能是攻击；
- 实现了 `Display`（给出人类可读信息）和 `std::error::Error`——本 crate 是库，使用它的客户端（第一个是 `duckctl`）希望能直接用 `?` 传播。

### 3.4 `chunks(line: &str, mtu: usize) -> Vec<Vec<u8>>`

把一条出站行切成通知大小的分片，**换行符包含在载荷中**：

- `mtu` 用 `.max(20)` 兜底：0 或荒谬的 MTU 会导致除零或每字节一片，而 20 是协商前 BLE 保证的下限；
- 行末若无换行则补一个，有则不重复补；
- 客户端重组到换行即结束，无需"消息结束"之外的概念。

## 4. 单元测试（9 个）

| 测试 | 验证内容 |
| ---- | -------- |
| `a_line_split_across_chunks_reassembles` | 分 3 片到达的行能重组，且 `pending()` 归零 |
| `several_lines_in_one_chunk_all_come_back` | 一片中含多行时全部返回（2 行） |
| `a_partial_trailing_line_is_retained` | 尾部残行被保留，作为下一条消息的开头 |
| `blank_lines_are_skipped` | `\n`、`\r\n` 等空行被忽略，不当作空请求转发（与 `robotd`/`updaterd` 行为一致） |
| `crlf_is_tolerated` | 容忍 CRLF |
| `a_line_without_a_newline_is_refused_at_the_cap` | 超过上限返回 `LineTooLong`，且缓冲被释放（防止重试钉住内存） |
| `invalid_utf8_is_refused` | `0xFF 0xFE \n` 返回 `NotUtf8` |
| `chunking_round_trips_at_every_mtu` | 在 MTU 20/23/100/185/512/4096 下，切分与重组互为逆变换；每片不超过 MTU |
| `chunks_terminate_with_exactly_one_newline` | 无论原行是否带换行，输出恰好一个换行且位于末尾 |
| `an_absurd_mtu_falls_back_to_the_ble_floor` | MTU=0 时回退到 20，41 字节输出 3 片，不 panic |

（注：模块内实际为 10 个测试函数。）

## 5. 本文件要点小结

1. 纯逻辑、无 IO，是 BLE 与 NDJSON 之间的帧层；
2. 不引入帧头，复用 JSON 中不会出现裸换行这一性质，以 `\n` 分隔双向消息；
3. 8 KiB 行上限防内存耗尽，超限/非法 UTF-8 均清空缓冲并报错关连接；
4. 出站分片按 `max(mtu,20)` 切分，恰好携带一个末尾换行；
5. 切分与重组的互逆性质在多个 MTU 下由测试锁定。
