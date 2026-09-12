# `session.rs` 文件解析 —— 一个中心设备的完整会话

## 1. 文件定位

- 路径：`src/session.rs`
- 角色：`btd` **全部行为**所在。处理一个中心设备从 `hello` 到断开的整段会话。
- 它不持有任何关于机器人的状态，只持有关于"对话"的状态：一个重组缓冲区，以及本会话按需打开过的上游 socket 连接。

## 2. 核心原则：逐字转发

请求被**逐字（verbatim）**转发。`btd` 把每行只解析到足以回答两个问题的程度：

1. 该方法在此是否被允许；
2. 归哪个 socket。

随后就把原始字节转发出去。它绝不重写 `id`、绝不重新序列化参数、绝不捏造结果。这使其保持为传输层而非 API 的第二份实现；协议新增方法只需在 `route` 表加一行、此处无需改动。

## 3. 常量 `PIN_ATTEMPTS: u32 = 3`

- 六位 PIN 有一百万种可能，链路加密但未认证，限制尝试次数是无线电范围内暴力枚举前的唯一屏障；
- 错 3 次即结束会话；重连需要完整的 BLE 连接与绑定，把一下午的猜解拉长到很久，同时对误输两次的合法用户仍可接受。

## 4. `run(link: Link, sockets: Sockets)`

服务一个中心设备，直到它断开或违反分帧。主要结构：

1. 记会话开启日志（peer、mtu）；
2. 建立 `replies` channel（容量 `QUEUE`），创建 `Pool`，创建 `Reassembler`；
3. 状态：`authenticated = false`、`attempts_left = 3`；
4. 主循环 `tokio::select!` 两个分支：
   - **入站分片**：`link.inbound.recv()`。通道关闭则退出；交给 `Reassembler::push`，分帧失败（无完整请求、无法应答）直接结束会话并记 warn；对重组出的每一行调用 `dispatch`，若有响应则 `send_line` 发回（发送失败说明中心设备已走，直接返回）；`outcome.close` 为真（PIN 尝试耗尽）则返回；
   - **上游回复/通知**：`replies.recv()`。每行 `send_line` 发回；通道理论上不会关闭（`Pool` 与循环同寿），但仍按会话结束处理而非 panic；
5. 结束时若 `inbound.pending() > 0`，以 debug 记录"会话在半条消息中断开"（区分正常离开与中途消失），最后记会话关闭日志。

## 5. `Outcome` 与 `dispatch()`

`Outcome { response: Option<String>, close: bool }`，构造器 `nothing()`（上游会应答）与 `reply()`（btd 自己应答）。

`dispatch()` 处理一行完整文本，顺序如下：

1. **解析为 `proto::Request`**：失败则返回 `id=null` 的 PARSE_ERROR 错误（规范如此，所有客户端都能处理）；
2. 取出 `id`。通知（无 id）不期待应答，即使被拒也静默丢弃，但**仍不转发**——白名单不是建议；
3. **`as_call()` 识别方法**：失败时，有 id 就回错误，无 id 则什么都不回；
4. **PIN 闸门**：未认证时，除 `SystemAuthenticate` 与 `Hello` 外一律拒绝（PERMISSION_DENIED，消息明确提示先发送 `system.authenticate` 并用 `robotctl system pin` 查 PIN）。`hello` 例外是因为它只报告版本（与未认证就能读到的 GATT 读信息相同），拒绝它会让版本不匹配的客户端无从知道原因；
5. **路由**：
   - `Route::To(upstream, lane)`：交给 `pool.send(...)` 转发；上游不可达时回 INTERNAL_ERROR，且错误中**点名服务**（如 "Robot is not answering"），便于从手机截图判断是"robotd 没应答"还是"机器人拒绝"；
   - `Route::Local`：必须是 `SystemAuthenticate`，否则记"本地路由但无处理器"的内部错误（说明路由表长了本地方法却没教 `dispatch`）；转入 `authenticate()`；
   - `Route::Refused`：回 `route::refusal(call)`（PERMISSION_DENIED）。

## 6. `authenticate()` —— PIN 校验与尝试配给

- 每次尝试都通过 `pairing::pin()` 现取期望 PIN（不缓存），因此改 PIN 下次即生效；`configd` 答不上来就拒绝认证而非放行；
- **常量时间风格比较**：先比长度相等，再逐字节 XOR 后折叠 OR，结果为 0 才算相等（不提前返回）。BLE 上时序信号淹没在毫秒级无线电抖动里，更多是卫生习惯，但零成本；
- 成功：置 `authenticated=true`，记录是否默认 PIN；若为出厂 PIN 则高声 warn 提示设置每机 PIN；回 `{authenticated:true, attempts_remaining:3}`；
- 失败：`attempts_left` 饱和减 1，记 warn，回 `{authenticated:false, attempts_remaining:n}`；当剩余为 0 时 `close=true`，第三次失败后结束会话。

## 7. 发送与编码辅助

- `send_line(link, line)`：用 `framing::chunks(line, link.mtu)` 分片，逐片 `link.outbound.send().await`；后端关闭即视为中心设备离开，返回 `Err(())`；
- `encode(response)`：`serde_json` 序列化；理论上不会失败，若失败也返回一段客户端可解析为错误的固定 JSON（避免客户端干等）。

## 8. 测试模块（约 12 个，基于真实 unix socket）

测试基础设施：

- `FakeDaemon`：在临时目录起真实 `UnixListener`，记录收到的行并按预置内容回复。用真实 socket 而非 mock，因为 btd 与守护进程之间的分帧本身就在被测范围；
- `read_reply()`：像真实客户端一样收集通知分片并用 `Reassembler` 重组；
- `authenticate()`：每个测试都先用 PIN `424242` 完成认证，从而每个测试都顺带施压于 PIN 闸门；
- `sockets()` / `tempdir()`：构造 socket 路径；临时目录前缀短（unix socket 路径受 `sun_path` 长度限制）。

关键测试：

| 测试 | 要点 |
| ---- | ---- |
| `an_allowed_call_is_forwarded_verbatim_and_answered` | 允许的调用**逐字节**到达正确守护进程，回复原样返回 |
| `a_refused_call_never_reaches_the_daemon` | 被拒调用收到 PERMISSION_DENIED，且守护进程确实收不到（白名单非建议） |
| `robot_calls_go_to_robotd` | `robot.health` 到 robotd 而非 updaterd，回复正确 |
| `every_notification_in_a_stream_reaches_the_client` | 订阅流的 3 条进度通知全部到达；23 字节 MTU 下顺带验证背靠背消息重组 |
| `an_unparseable_line_is_answered_and_the_session_survives` | 垃圾行回 id=null 的 PARSE_ERROR，会话不中断、后续 hello 仍正常 |
| `a_dead_daemon_is_reported_rather_than_hanging` | robotd 缺失时错误中含 "Robot is not answering"，不干等 |
| `a_refused_notification_is_answered_with_silence` | 无 id 通知被拒时既不回复也不转发 |
| `nothing_is_served_before_the_pin` | 未认证调用被拒、消息提示 system.authenticate、且未到达守护进程 |
| `hello_is_allowed_before_the_pin` | 认证前 hello 可用 |
| `wrong_pins_are_rationed_and_then_the_session_closes` | 剩余次数 2→1→0，第三次后会话关闭，第四次无法再送 |
| `a_leading_zero_is_part_of_the_pin` | 存 `042042` 时输入 `42042` 认证失败（前导零是 PIN 的一部分） |
| `an_apply_still_reaches_updaterd_while_a_progress_stream_is_open` | lane 的核心缺陷回归测试：单连接时 apply 会被写进已停止读取的订阅连接；现在二者走不同连接 |
| `a_status_poll_does_not_travel_behind_an_apply` | apply 进行中的 status 走另一条连接，不排在其后 |

后两个测试用 `spawn_serial_updaterd`（每条连接一次处理一个请求、收到 subscribe 即永久占用）并以 `<连接号> <行>` 上报，从而能断言两次调用是否走了同一条连接。

## 9. 本文件要点小结

1. `session` 是 btd 唯一的行为载体，但只持对话状态、不持机器人状态；
2. 每行只解析到"是否允许 + 归哪个 socket"，随后逐字转发；
3. PIN 闸门：认证前仅放行 hello 与 authenticate；错 3 次断连；PIN 现取、按串比较、近常量时间；
4. 上游不可达的错误点名服务，便于手机端诊断；
5. 分帧错误直接关会话，解析错误回 id=null 但保留会话，通知被拒不回执；
6. 测试全部经真实 unix socket，并重点锁定逐字转发、权限边界、PIN 配给与 lane 不阻塞。
