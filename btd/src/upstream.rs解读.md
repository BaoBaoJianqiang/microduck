# `upstream.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 330 行 |
| 角色 | 到实际拥有答案的服务的连接——unix socket 连接池与一次性查询 |
| 平台 | 跨平台 |

## 二、设计原则

### 每服务一个 socket，直接连接

- 四个服务没有理由用 broker，总线会是另一个可能失败的组件（architecture.md §2.2）。

### 所有操作都有超时边界，无一例外

- 任何对等方可能已死，关闭或静默的 socket 是正常答案而非值得永远重试的错误。
- `robotd` 尤其最可能缺失，因为它是更新会重启的那个。

## 三、常量

| 常量 | 值 | 说明 |
|---|---|---|
| `CONNECT_TIMEOUT` | 3 秒 | 对加载的板足够长，对手机得到答案而非旋转器足够短。unix socket 连接要么立即成功要么守护进程不在那里 |
| `WRITE_TIMEOUT` | 5 秒 | 单次写入上限。阻塞的写入意味着守护进程已停止读取，这是死对等方而非慢的 |

## 四、结构体

### `NameChoice`

```rust
pub struct NameChoice {
    pub pinned: Option<String>,
    pub fallback: String,
}
```

- 广告什么作为机器人。
- 名字属于 `configd`（architecture.md §4.1）——SDK 不应必须通过蓝牙设置它——因此 `btd` 询问而非决定。
- 放在这里而非 `bluez`，因此 crate 的入口点在非 Linux 上有相同形状（那里没有无线电可广告）。

| 字段 | 说明 |
|---|---|
| `pinned` | `--name`，固定广告名称并关闭协调。台架使用：存在以便板可以获得已知名称而不接触其存储配置 |
| `fallback` | configd 无法到达时使用。`btd` 在恢复路径上，必须在机器人其余部分不工作时回答（`systemd/btd.service`），因此不可达的 configd 只损失派生名称 |

### `Sockets`

```rust
pub struct Sockets {
    pub updater: PathBuf,
    pub robot: PathBuf,
    pub config: PathBuf,
}
```

- 每个服务监听的位置。
- `path(upstream: Upstream) -> &Path`：根据 Upstream 枚举返回对应 socket 路径。

## 五、一次性查询：`ask`

### `ask(service, socket, call, timeout) -> Result<Response, String>`

- 代表 `btd` 自己向一个服务问一个问题。
- **一次性连接而非 Pool 条目**，因为这不是转发：这里没有任何东西属于客户端的会话。
- `btd` 自己问两个问题——配对交换中的 PIN，以及要广告的机器人名字——两者都想要现在的单个答案而非合并的行流。
- 恰好一个回复在飞行中，没有什么要关联，因此 `id` 是常量。
- 像此模块中其他一切一样有超时边界。调用者选择超时，因为截止时间差一个数量级：BlueZ 持有配对交换打开而手机显示旋转器，而没有东西在等名字。
- 返回响应时其错误已转为 `Err`，因此调用者只需反序列化它期望的结果。

### `ask_now`（内部）

实际执行：
1. 连接 unix socket
2. 发送 `proto::Request::call(Id::Number(1), call)` + 换行
3. 读取一行回复
4. 反序列化为 `proto::Response`
5. 如果有错误，返回 `Err(format!("{service} refused: {error}"))`

## 六、连接池：`Pool`

### 设计

```rust
pub struct Pool {
    sockets: Sockets,
    conns: HashMap<(Upstream, Lane), Conn>,
    replies: mpsc::Sender<String>,
}
```

- 在一个 BLE 会话中到目前为止打开的连接，按需创建。

### 惰性而非急切

- 大多数会话只接触一个服务：问版本的手机没有理由让 `robotd` 接受它永远不会用的连接。
- 急切连接意味着死的 `robotd` 会延迟或失败不需要它的会话。

### 按 lane 以及 service 键控

- **这是保持数分钟长的更新不静默客户端问的其他一切的东西。**
- 这里的每个守护进程一次一个连接服务一个请求，因此共享连接的调用共享队列——见 `Lane` 了解每服务单个连接打破的两种排序。
- 每服务每会话最多四个 socket，实践中两个。

### `Pool::send(upstream, lane, line) -> io::Result<()>`

1. 如果 `(upstream, lane)` 没有连接，先 `open` 一个。
2. 写入 `line + \n`，带 `WRITE_TIMEOUT`。
3. 错误处理：
   - **Broken pipe**：普通——守护进程重启了。删除连接以便下一次调用重连而非永远写入死 socket。只删此 lane 的连接：其他可能完全活着。
   - **超时**：删除连接，返回 `TimedOut` 错误。

### `Pool::open`（内部）

1. 带 `CONNECT_TIMEOUT` 连接 unix socket。
2. 拆分为读写两半。
3. **生成读半部分的泵任务**，为会话的生命周期运行：
   - 响应和通知对我们来说是同一件事：要转发的一行。
   - 这就是 `update.subscribe` 的进度流无需任何特殊情况就能工作的原因。
   - 完整队列意味着中心跟不上。放弃该行而非会话：进度是建议性的，在这里阻塞会停滞每个其他上游。
4. lane 在标签中，因为现在到每个服务有几个连接，"Updater closed" 没有它命名四个可能的 socket——包括进度流（其关闭是普通的）和操作 lane（其关闭不是）。

## 七、测试

| 测试 | 验证内容 |
|---|---|
| `a_question_is_asked_and_the_answer_deserialised` | 完整路径：问 configd，得到名字回来；线上方法是被问的那个 |
| `an_absent_service_is_an_error_naming_it` | 缺失的服务是命名它的错误而非挂起或 panic |
| `a_refusal_is_an_error_rather_than_an_empty_answer` | 拒绝的服务必须不读为答案（否则调用者会广告 `result_as` 对 null 结果做出的任何东西） |
#（注：内容由AI生成）
