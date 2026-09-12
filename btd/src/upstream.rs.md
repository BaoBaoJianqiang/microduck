# `upstream.rs` 文件解析 —— 到上游服务的连接

## 1. 文件定位

- 路径：`src/upstream.rs`
- 角色：管理与真正拥有答案的服务（`updaterd`、`robotd`、`configd`）之间的 unix socket 连接。
- 设计取向：每个服务一条直连 socket，不引入消息代理（broker）。四个服务的规模没有上代理的必要，而总线会是又一个可能故障的组件（`architecture.md` §2.2）。
- 铁律：这里的每个操作都有超时上限。任何对端都可能已死；关闭或沉默的 socket 是正常应答，而不是值得无限重试的错误。`robotd` 尤其最可能缺席，因为更新流程会重启它。

## 2. 超时常量

- `CONNECT_TIMEOUT = 3s`：对负载较重的板卡足够，又能让手机拿到应答而非一直转圈。unix socket 连接要么立即成功，要么守护进程不在；
- `WRITE_TIMEOUT = 5s`：单次写上限。写被阻塞意味着守护进程已停止读取，是死对端而非慢对端。

## 3. `NameChoice`（广播名选择）

```rust
pub struct NameChoice { pub pinned: Option<String>, pub fallback: String }
```

- 名字归 `configd` 所有（SDK 不该为改名字而走蓝牙），所以 btd 只"询问"而不"决定"；
- `pinned` 对应 `--name`：钉死广播名并关闭名字对账，台架用途；
- `fallback` 在联系不上 `configd` 时使用。btd 处于恢复路径（见 `systemd/btd.service`），必须在机器人其余部分不响应时仍能应答，因此取不到 configd 至多损失派生名；
- 该结构放在本模块而非 `bluez`，是为了让 crate 入口在没有无线电的非 Linux 平台也保持相同形状。

## 4. `Sockets`

三个 `PathBuf`：`updater`、`robot`、`config`。方法 `path(upstream: Upstream) -> &Path` 按枚举返回对应路径。

## 5. 一次性询问：`ask()` 与 `ask_now()`

```rust
pub async fn ask(service, socket, call, timeout) -> Result<proto::Response, String>
```

- 代表 btd **自己**向某服务提一个问题（配对时取 PIN、启动/对账时取机器人名与地址），不是转发客户端会话，因此用一次性连接而非 `Pool` 条目；
- 恰好一个在途回复，无需关联，故 `id` 用常量 `Id::Number(1)`；
- 超时由调用方给定（PIN 场景 BlueZ 持有配对交换、时限以秒计；名字场景无人等待，deadline 相差一个数量级）；
- `ask()` 用 `tokio::time::timeout` 包裹，超时错误为 "{service} did not answer in time"；
- `ask_now()`：连接 socket（错误信息含 `cannot reach {service} at {path}`）、拆分为读写两半、发一行带换行的 `Request::call`、flush、读一行回复并反序列化；若响应带 `error` 字段则转成 `Err("{service} refused: {error}")`，使调用方只需反序列化它期望的 result。

## 6. 会话连接池：`Pool`

```rust
pub struct Pool {
    sockets: Sockets,
    conns: HashMap<(Upstream, Lane), Conn>,
    replies: mpsc::Sender<String>,
}
```

### 6.1 按需（惰性）连接

- 大多数会话只接触一个服务，询问版本的手机没理由让 robotd 接受一条永远用不到的连接；
- 惰性连接也避免一个死掉的 robotd 拖垮本不需要它的会话。

### 6.2 以 (服务, lane) 为键

- 每个守护进程都是"一条连接一次处理一个请求"，共享连接即共享队列；
- 以 lane 为键使得长达数分钟的更新不会让客户端的其他查询全部沉默；
- 每服务每会话至多四条 socket，实践中为两条。

### 6.3 `send(upstream, lane, line)`

- 键不存在则先 `open()`；
- 取连接（unwrap 成立：刚插入或上面的 contains_key 成立）；
- 给行补换行，在 `WRITE_TIMEOUT` 内 `write_all + flush`；
- 写错误（如管道破裂，守护进程重启）：**只移除该 lane 的连接**，下次调用会重连；其他 lane 可能完好；
- 超时：移除连接并返回 `TimedOut` 错误。

### 6.4 `open(upstream, lane)`

- 在 `CONNECT_TIMEOUT` 内连接，超时返回 io 错误；
- 拆分读写两半，克隆 `replies` 发送端，派生一个读任务，在会话生命周期内持续泵取；
- 读任务里"响应"和"通知"是同一种东西——都是要转发的一行，这让 `update.subscribe` 的进度流无需特殊处理；
- 读到一行用 `try_send` 进合并通道：队列满（中心设备跟不上）时**丢弃该行而非放弃整个会话**（进度是建议性的，阻塞在此会连带拖死其他所有上游），记 debug；读到 EOF 或出错则结束任务；
- 标签格式 `"{upstream:?}/{lane:?}"`，因为现在每服务有多条连接，"Updater closed" 不带 lane 会指向四个可能的 socket（含关闭很正常的进度流与关闭不正常的操作 lane）。

所有上游的回复/通知被合并进同一个 `replies` 通道。合并是安全的：JSON-RPC 用 `id` 关联，而关联是客户端的事——btd 不读这些行的内容，只转发。

## 7. 单元测试（3 个）

辅助：`serve_once(path, response)` 在真实 unix socket 上应答一次后挂断，并把收到的请求行作为 JoinHandle 结果返回。

| 测试 | 要点 |
| ---- | ---- |
| `a_question_is_asked_and_the_answer_deserialised` | 全链路：问 configd `SystemInfo`，反序列化得到 `name="duck-7f3a"`；并断言线上方法名确为 SYSTEM_INFO，而非只是恰好能解析 |
| `an_absent_service_is_an_error_naming_it` | 服务缺席时错误含 `cannot reach configd`，不挂起、不 panic（btd 在恢复路径上必须如此） |
| `a_refusal_is_an_error_rather_than_an_empty_answer` | 响应带 error 时 `ask` 返回错误（含 `configd refused`），不会被 `result_as` 读成空答案后拿去广播 |

## 8. 本文件要点小结

1. 三服务直连 socket、无代理；全部操作有连接/写超时；
2. `NameChoice`/`Sockets` 是启动与连接的配置载体，名字归属 configd，btd 只询问；
3. `ask()` 是 btd 自用的一次性一问一答（PIN、名字、地址），错误信息可诊断；
4. `Pool` 按 (服务, lane) 惰性建连，长操作与快速查询互不阻塞；
5. 读半连接常驻泵取，响应与通知一视同仁地转发；客户端跟不上时丢单行而非垮会话；
6. 死连接只移除对应 lane，下次自动重连。
