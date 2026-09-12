# producer.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\producer.rs`

## 核心设计决策

在会话开始前宣告本机器人是谁。`webrtcsink` 接收一个 `meta` 结构，信令服务器把它放进 `list` 应答发给每个对等端——于是客户端在协商任何东西之前就知道自己找到的是哪台机器人。

四个字段，各有调用方：

- **`name`**：页面可用机器人命名自己而非用 id；同一网络上两个生产者可区分。
- **`serial`**：持久句柄。名字可改、peer id 随会话，序列号寿命更长，是 app 键控机器人所需（`app-path-design.md` §8.6）。
- **`release`**：正在运行什么，让行为古怪的客户端与落后一个 release 的机器人可区分，无需开会话询问。
- **`api_version`**：与会话 `hello` 报告的同一偏差，早一个来回。

### 名字来自 configd，缺席不是失败

`system.info` 持有名字和序列号，`mediad` 已有到 `configd` 的连接。但 `mediad` 与 `configd` 同时启动而非之后（unit 是 `After=` 非 `Requires=`，因为某服务挂掉不能阻止本服务启动）。所以这里询问、等 `ASK_TIMEOUT`（3 秒），无论如何继续：无名生产者照样流化，因学不到自己名字就不流化是更糟的交易。

只在启动时问一次，重命名在下次重启生效（与 `configd` 对 `btd` 广播的处理一致）；要实时答案的对等端可在自己控制通道上调用 `system.info`。

## 类型

### `Producer`
`name: Option<String>`、`serial: Option<String>`、`release: String`、`api_version: u32`。

## 函数

### `local(build: BuildInfo) -> Self`
进程自知的部分。`release` 从可执行文件路径推导（`proto::release_from_path`），而非版本号——整个 workspace 共享一个版本行，`0.9.1` 在 dev 渠道上可能指向两个不同构建，安装目录才区分它们。手工构建的二进制无此路径，则用编译进的 build 字符串。

### `learn(sockets, build) -> Self`
`local` 加上 `configd` 的名字与序列号。`configd` 不应答只 `warn` 不 `error`。

### `fields() -> Vec<(&str, String)>`
构造 `GstStructure` 的键值对。缺席字段直接缺席而非空串（`serial: ""` 需要约定"空=无"，缺键则无需约定）。

### `ask_configd(sockets)`
一次 `system.info` 调用，用 `Pool` 共享连接/写超时（不另长一套）。

## 单元测试

| 测试 | 意图 |
|---|---|
| `what_is_known_without_asking_anything` | `api_version`、`release` 始终已知，`name` 为 None |
| `absent_fields_are_absent` | 无名机器人只发 2 个字段，不是 4 个里两个空 |
| `a_named_robot_publishes_the_lot` | 有名有序列号时发全部 4 个字段 |
| `a_silent_configd_still_yields_a_producer` | configd 不应答只丢名字，其余正常 |

## 关键摘要

- 生产者身份在协商前通过 `webrtcsink` 的 `meta` 传达。
- 名字/序列号向 `configd` 问一次，超时 3 秒，缺席不致命。
- `release` 从安装路径推导而非版本号（dev 渠道同版本不同构建）。
