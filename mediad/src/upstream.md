# upstream.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\upstream.rs`

## 核心设计决策

到持有答案的服务的连接。每服务一个 socket，直连——五个服务仍无需 broker，总线只是又一个会失败的组件（`architecture.md` §2.2）。

**五个，`btd` 只持三个。** 这是两个传输的具体区别：`mediad` 承载 pad 输入与深度流，BLE 因容量拒绝，`btd` 故意不持它们的 socket。决定住在 `mediad::route`。

每个操作都有超时边界，无一例外。任何对等端都可能死，关闭或静默的 socket 是正常答案而非值得无限重试的错误——`robotd` 尤其最可能缺席（它是被更新重启的那个）。

### 与 `btd::upstream` 接近但暂不共享

刻意如此。两者差异不止服务数：`btd` 的携带 `NameChoice`（关于蓝牙广播什么，在此无意义）。从一个例子提取共享池是在猜形状；等这个在真实对等端上跑过，从两个例子提取才是有证据的重构。重复在此点名，是决策而非疏忽。

## 常量

| 常量 | 值 | 含义 |
|---|---|---|
| `CONNECT_TIMEOUT` | 3 s | unix socket 连接要么立即成功要么守护进程不在 |
| `WRITE_TIMEOUT` | 5 s | 阻塞写意味着守护进程已停止读 |

## 类型

### `Sockets`
五个服务的路径，默认 `proto::socket::*`，杜绝客户端各自复制路径漂移。`path(service)` 映射。

### `Conn`
`write: OwnedWriteHalf`（读半在后台 pump 任务里）。

### `Pool`
- `conns: HashMap<(Service, Lane), Conn>`——按（服务，车道）键控，所以一个几分钟的更新不会静默该对等端的其他请求。每服务每会话最多 4 个 socket（实际约 2 个）。
- `replies: mpsc::Sender<String>`——所有服务的回复与通知合并。JSON-RPC 按 `id` 关联是对等端的事，`mediad` 只转发行（与 `btd` 同理：订阅是通知流，关联会丢后续）。

## 函数

### `Pool::new(sockets, replies)`

### `Pool::send(service, lane, line) -> io::Result<()>`
懒连接，写入带 `WRITE_TIMEOUT`。写失败或超时移除该 lane 的连接（其他 lane 可能活着），下次重连。

### `Pool::open(service, lane) -> Conn`
连接后 spawn 后台任务 pump 读到 `replies`。满队列 `try_send` 失败则丢该行（对等端跟不上），不阻塞其他服务。日志带 `{service}/{lane}` 标签（每服务多连接，"Updater closed" 可能指进度流——正常——或 operation lane——不正常）。

## 关键摘要

- 每（服务，车道）一个连接，车道隔离防止长操作阻塞其他请求。
- 回复/通知合并转发，不解析（订阅流语义）。
- 全超时边界；写失败只移除对应 lane 的连接。
- 与 `btd::upstream` 刻意重复，等两个实现都跑过后再考虑共享。
