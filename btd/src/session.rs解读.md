# `session.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 939 行（含 ~500 行测试） |
| 角色 | 一个连接的中心，从 `hello` 到断开——`btd` 的全部行为 |
| 平台 | 跨平台（可在无蓝牙的真实 unix socket 上测试） |

## 二、核心定位

- 这是 `btd` 的全部行为，它不持有关于机器人的状态——只关于对话：重组缓冲区和此会话有理由打开的上游 socket。
- 请求被**原样**转发。`btd` 解析每一行只够回答两个问题——此方法在这里允许吗，哪个 socket 拥有它——然后传递原始字节。
- 它从不重写 `id`，从不重新序列化 params，从不发明结果。
- 这就是保持它是传输而非 API 的第二个实现的原因，也是添加协议方法在 `route` 中花一行、此处什么都不花的原因。

## 三、常量

### `PIN_ATTEMPTS = 3`

- 会话在关闭前可以提供的错误 PIN 数。
- 六位数 PIN 是一百万次猜测，链接加密但未认证，因此定量尝试是站在无线电范围内的对等方和暴力破解之间的唯一东西。
- 三次，然后会话结束：重连花费完整 BLE 连接和绑定，这把一下午的猜测变成长得多的东西，同时对输错两次的合法客户端不可见。

## 四、`run(link, sockets)`——会话主循环

```rust
pub async fn run(mut link: Link, sockets: Sockets) {
```

### 初始化

1. 记录会话打开（peer, mtu）。
2. 创建 `replies` channel（容量 `QUEUE`）：所有上游回复和通知合并到此。
3. 创建 `Pool`（惰性连接池）。
4. 创建 `Reassembler`（入站字节重组）。
5. `authenticated = false`：在客户端证明 PIN 之前，除了 `hello` 和 `system.authenticate` 什么都不服务。
6. `attempts_left = PIN_ATTEMPTS`。

### 主循环 `select!`

#### 分支 1：来自无线电的字节（`link.inbound.recv()`）

- `None`（中心断开）→ break。
- 调用 `inbound.push(chunk)`：
  - 成帧失败 → 记录警告，break（成帧失败结束会话而非被回答：没有 id 可以回答*到*——我们从未看到完整请求——无法成帧的对等方不会被它也无法解析的 JSON 错误帮助）。
- 对每个完整行调用 `dispatch()`。
- 如果 `outcome.response` 是 Some，通过 `send_line` 发送。
- 如果 `outcome.close`（PIN 尝试耗尽），记录警告并返回。

#### 分支 2：来自服务的回复或通知（`replies.recv()`）

- `None` 不可达（channel 由 pool 持有，pool 与此循环同寿命），但视为会话结束而非 panic。
- 通过 `send_line` 发送到中心。

### 会话结束

- 如果 `inbound.pending() > 0`，记录 debug（区分"客户端完成并离开"和"客户端在消息中间消失"，正常断开和 bug 的区别）。
- 记录会话关闭。

## 五、`Outcome` 结构体

```rust
struct Outcome {
    response: Option<String>,  // btd 必须自己发送的响应。None 意味着上游会回答——普通路径
    close: bool,                // 发送后结束会话。只有耗尽 PIN 尝试时设置
}
```

辅助构造函数：`nothing()`, `reply(response)`。

## 六、`dispatch()`——处理一行

### 步骤

1. **解析为 `proto::Request`**：
   - 失败 → 用 `null` id 回答 `PARSE_ERROR`（无法从不可解析的行恢复 id，因此响应携带 `null`——规范要求，每个客户端已经处理）。

2. **提取 `id`**：通知（无 id）期望无回复，因此被拒绝的通知被静默丢弃。它仍然不被转发：白名单不是建议性的。

3. **提取 `call`**（`request.as_call()`）：
   - 失败 → 用该 id 回答错误（或通知则什么都不做）。

4. **PIN 门**：
   - 如果 `!authenticated` 且调用不是 `SystemAuthenticate` 或 `Hello` → 拒绝，`PERMISSION_DENIED`，消息说明需要先认证（发送 `system.authenticate` 和机器人 PIN）。
   - `hello` 被允许通过因为它只报告版本——与 GATT 读已经给未认证客户端的信息相同——拒绝它会让不匹配的客户端无法学习为什么什么都不工作。

5. **路由**（`route::route_for(call)`）：
   - `Route::To(upstream, lane)` → 通过 `pool.send()` 转发原始行。失败 → 回答 `INTERNAL_ERROR`，命名服务（"Robot is not answering: ..."——从手机截图可诊断）。
   - `Route::Local` → 必须是 `SystemAuthenticate`（否则是路由表增长了本地方法而没教此函数——回答 `INTERNAL_ERROR`）。调用 `authenticate()`。
   - `Route::Refused` → 回答 `route::refusal(call)`（`PERMISSION_DENIED`，命名方法，建议用 `robotctl`）。

## 七、`authenticate()`——检查 PIN，定量尝试

### 流程

1. 从 `configd` 获取期望的 PIN（`pairing::pin(config_socket)`）：
   - 每次尝试获取而非缓存，因此 `robotctl system set-pin` 在下一次尝试时生效而非下一次重启。
   - `configd` 无法回答意味着会话被拒绝而非准入。
   - 失败 → 回答 `INTERNAL_ERROR`（"cannot check the PIN: configd is not answering"）。

2. **常量时间-ish 比较**：
   - 比较等长的完整字符串而非在第一个不同数字上提前返回。
   - 通过 BLE 时序信号被埋在几毫秒的无线电中，因此这是卫生而非防御——但不花什么。
   - `expected.pin.len() == params.pin.len() && expected.pin.bytes().zip(params.pin.bytes()).fold(0u8, |acc, (a,b)| acc | (a^b)) == 0`

3. **成功**：
   - `authenticated = true`。
   - 如果 `is_default`，每次都警告（出厂 PIN 认证了任何读了此仓库的人，建议设置每机器人 PIN）。
   - 回答 `AuthenticateResult { authenticated: true, attempts_remaining: PIN_ATTEMPTS }`。

4. **失败**：
   - `attempts_left -= 1`（saturating）。
   - 回答 `AuthenticateResult { authenticated: false, attempts_remaining }`。
   - 如果 `attempts_left == 0`，设置 `close = true`。

### 字符串比较而非数字

- `000042` 和 `42` 是不同的 PIN，数字解析会使它们相同。
- 存储形式是字符串正是为此。

## 八、`send_line()`——分块输出

```rust
async fn send_line(link: &Link, line: &str) -> Result<(), ()> {
    for chunk in framing::chunks(line, link.mtu) {
        if link.outbound.send(chunk).await.is_err() {
            return Err(());  // 后端丢弃了它的一半：中心走了
        }
    }
    Ok(())
}
```

## 九、`encode()`——序列化响应

```rust
fn encode(response: &proto::Response) -> String {
    serde_json::to_string(response).unwrap_or_else(|_| {
        r#"{"jsonrpc":"2.0","id":null,"error":{"code":-32603,"message":"internal error"}}"#.to_owned()
    })
}
```

- Response 是普通字符串、整数和枚举；这不能失败。
- 如果它 somehow 失败了，发送什么都没有会挂起客户端，因此发送它可以解析为错误的东西。

## 十、测试（13 个，使用真实 unix socket）

测试基础设施：
- `FakeDaemon`：接受连接、记录收到的行、用测试排队的任何东西回复的守护进程替身。使用真实 unix socket 而非 mock，因为 btd 和守护进程之间的成帧是被测内容的一部分。
- `read_reply()`：收集通知块并像客户端那样重组它们。
- `authenticate()`：做真实客户端现在必须先做的事：证明 PIN。每个测试都经过此，意味着每个测试也锻炼门——停止认证的会话会失败所有测试而非静默服务一切。
- `spawn_serial_updaterd()`：带 `updaterd` 连接模型的替身——**每连接一次一个请求**，交给 `update.subscribe` 的连接永远不读另一行。这是 lane 存在的全部原因。

| 测试 | 验证内容 |
|---|---|
| `an_allowed_call_is_forwarded_verbatim_and_answered` | 普通路径：允许的调用**逐字节**到达正确的守护进程，其回复回来。原样转发是保持 btd 是传输的属性 |
| `a_refused_call_never_reaches_the_daemon` | 被拒绝的调用永远不接触上游。正确回答不够——白名单的要点是守护进程永远看不到它 |
| `robot_calls_go_to_robotd` | `robot.*` 到达 `robotd` 而非 `updaterd`。一个表驱动路由和权限，因此这里的错误会把更新触发器发送到控制守护进程 |
| `every_notification_in_a_stream_reaches_the_client` | 订阅是开放连接上的通知流，每个都必须到达中心。这是如果回复与请求相关联而非到达时转发会破坏的情况 |
| `an_unparseable_line_is_answered_and_the_session_survives` | 垃圾得到带 null id 的错误，不是被丢弃的会话：发送了一行坏内容的客户端应该能够继续 |
| `a_dead_daemon_is_reported_rather_than_hanging` | 不运行的守护进程必须产生命名它的可诊断错误，而非挂起。`robotd` 恰好在更新刚重启它时缺失，这正是某人最可能看手机的时候 |
| `a_refused_notification_is_answered_with_silence` | 通知（无 id）即使被拒绝也得不到回复——规范这么说，等待一个的客户端会永远等 |
| `nothing_is_served_before_the_pin` | 门。未认证的调用被拒绝，消息说明该怎么做——这是手机应用作者会碰到的第一件事。且它永远不到达守护进程：门不是建议性的 |
| `hello_is_allowed_before_the_pin` | `hello` 是例外，因为它只报告版本——与 GATT 读已经告诉未认证客户端的东西相同——拒绝它会让不匹配的客户端无法学习为什么什么都不工作 |
| `wrong_pins_are_rationed_and_then_the_session_closes` | 错误 PIN 倒数然后关闭会话。六位数 PIN 是一百万次猜测，在加密但未认证的链接上，定量尝试是使暴力破解昂贵的唯一东西 |
| `a_leading_zero_is_part_of_the_pin` | 仅在前导零上不同的 PIN 不能认证。存储形式是字符串正是为此，`042042` 和 `42042` 是不同的秘密 |
| `an_apply_still_reaches_updaterd_while_a_progress_stream_is_open` | **lane 被添加来修复的失败**，传输中最糟的一个：单连接时，apply 被写入 `stream_progress` 已停止读取的 socket。无回复、无错误、无更新——主人点"更新"机器人什么都不做。此文件中没有其他测试会抓到它，因为每个其他测试只做一个调用 |
| `a_status_poll_does_not_travel_behind_an_apply` | 更新期间的状态轮询不能在 apply 后面排队。`updaterd` 费了很大劲在引擎忙时回答 `update.status`——如果请求在 socket 中未读地坐更新花费的几分钟，所有努力都浪费了 |
#（注：内容由AI生成）
