# `session.rs` 解读

## 概述

`session.rs` 是 mediad 的 **WebRTC 控制会话管道**——把一个 peer 的控制通道当作"管道到拥有答案的服务"来驱动。它负责读取入站 JSON-RPC 行、路由到后端服务、并把服务的回复和通知原样转发回 peer。

**核心设计原则：传输无关（transport-agnostic）**：

> *Transport-agnostic on purpose: this takes lines in and gives lines out, and knows nothing about datachannels. That is what makes it testable without a WebRTC peer — the tests below drive it over channels against fake daemons on real unix sockets — and it is also what would let a WebSocket surface (`remote-webrtc.md` §11) reuse it unchanged.*
> —— 刻意设计为传输无关：吃进行、吐出行，对 datachannel 一无所知。这使得无需 WebRTC peer 即可测试——下面的测试用 channel 驱动它，对接跑在真实 unix socket 上的假守护进程——也让 WebSocket 表面（§11）可以不加修改地复用它。

### 两个"不做"

**1. 它从不解析回复（It never parses a reply）**：

> *Requests are read far enough to route them and no further; everything a service emits is forwarded verbatim.*
> —— 请求只读够路由就停；服务发出的一切逐字转发。

由此推出两个重要性质：
- 订阅无需特殊处理——它就是一条开着的连接上的通知流，每一条都要到达 peer。如果试图把回复和请求做关联（correlate），就会只保留第一个、丢掉其余的。
- 给 API 加方法在这里零成本。`duck-ipc-proto` 是方法定义的唯一位置，本文件不需要为新方法加 case。

**2. 它不认证（It does not authenticate）**：

> *`remote-webrtc.md` §4: there is no gate on the robot. A LAN peer may drive it, and a bridged peer authenticated to the rendezvous service — on both sides — before arriving. `route::permits` refuses `system.authenticate` by name rather than answering it, so a client that asks gets a clear no instead of a lie.*
> —— §4：机器人上没有闸门。局域网 peer 可以驾驶它，桥接 peer 在到达前已向会合服务认证过（双向）。`route::permits` 按名字拒绝 `system.authenticate` 而非回答它，所以问的客户端得到明确的"不"而不是谎言。

## 关键结构体与函数

### `Video` — 视频流描述

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Video {
    pub width: u32,
    pub height: u32,
    /// Degrees clockwise the camera is mounted from upright.
    /// —— 摄像头从直立位置顺时针安装的度数。
    pub rotate: u32,
}
```

描述编码器发送的帧尺寸，以及摄像头安装偏离直立的角度。由会话持有，因为 peer 必须能够**询问**它。

注释详细解释了为什么需要这个信息：

> *Pushing it when the channel appears races the browser: `mediad` writes the line the moment `webrtcsink` hands over the channel, and if the peer's datachannel is not open yet the line is dropped — which is exactly what happened, and the console showed a sideways picture with nothing in the log to say why.*
> —— 在通道出现时推送它会和浏览器竞速：`mediad` 在 `webrtcsink` 交出通道的那一刻就写行，如果 peer 的 datachannel 还没打开，这行就丢了——而这正是实际发生过的事：控制台显示侧着的画面，日志里却什么都没说。

> *The push is kept as a courtesy for a client that only listens; `media.video` as a call is what the console uses, because a question it asks when it is ready cannot arrive too early.*
> —— 推送保留给只听不应答的客户端作为善意；控制台用的是 `media.video` 作为*调用*——它在自己准备好时问的问题，不可能太早到达。

### `VIDEO_METHOD` 常量

```rust
const VIDEO_METHOD: &str = "media.video";
```

peer 用来询问视频信息的方法名。在本文件内应答，不路由到任何服务——因为没有服务拥有它。

### `video_notification` — 视频通知

```rust
pub fn video_notification(video: Video) -> String
```

构造一条无 id 的 JSON-RPC notification，告知 peer 视频尺寸和旋转角度。

> *The rotation is the whole point. Nothing on the robot rotates pixels any more — a `videoflip` in the pipeline cost the encoder its zero-copy path and the board 22 fps.*
> —— 旋转是全部重点。机器人上不再有任何东西旋转像素——pipeline 里一个 `videoflip` 让编码器丢掉零拷贝路径，板子少了 22 fps。

浏览器无法推断旋转角度：180° 安装和直立安装从宽高比上无法区分。

### `run` — 会话主循环

```rust
pub async fn run(
    mut inbound: mpsc::Receiver<String>,
    outbound: mpsc::Sender<String>,
    mut pool: Pool,
    video: Video,
)
```

驱动一个 peer 的控制通道，直到入站流结束。

- `inbound`：peer 发来的，每项一个 JSON-RPC 对象。
- `outbound`：回复和通知的去向，从所有服务合并而来——peer 按 `id` 自行排序，那是它的事不是我们的。

循环逻辑很简单：收一行 → `handle()` → 如果有回复就发回 → outbound 断了就退出。

### `handle` — 路由单行请求（核心）

```rust
async fn handle(line: &str, pool: &mut Pool, video: Video) -> Option<String>
```

路由一行请求。**只在本传输自己应答时返回回复——也就是只在拒绝时返回**。

处理流程：

#### 1. 解析 JSON-RPC

```rust
let request: proto::Request = match serde_json::from_str(line) {
    Ok(request) => request,
    Err(e) => {
        // Unparseable, so there is no `id` to answer against and no method to name.
        // A JSON-RPC parse error with a null id is the honest reply.
        // —— 无法解析，所以没有 id 可回应，也没有方法可指名。
        //    带 null id 的 JSON-RPC 解析错误是诚实的回复。
        return Some(error_line(None, ...));
    }
};
```

#### 2. 提取 id

```rust
// The id, kept before the request is consumed. A notification has none, and
// `Response` serialises that as `null` — which is right: a refusal to a
// notification is still worth sending, because silence would look like acceptance.
// —— id 在 request 被消费之前保存。notification 没有 id，`Response`
//    会序列化为 null——这是对的：对 notification 的拒绝仍然值得发出，
//    因为沉默看起来就像接受。
let id = request.id.clone();
```

#### 3. `media.video` 本文件直接应答

```rust
// Answered here, before anything tries to make a `Call` of it: this is `mediad`'s
// own question about `mediad`'s own pipeline, and there is no service to route it to.
// —— 在任何试图把它变成 `Call` 之前就在这里应答：这是 mediad 问自己
//    自己 pipeline 的问题，没有服务可路由。
if request.method == VIDEO_METHOD {
    return Some(...);
}
```

#### 4. 转为 `Call` 并路由

```rust
let call = match request.as_call() {
    Ok(call) => call,
    Err(e) => return Some(error_line(id, e)),
};
```

> *An unknown method or a params shape this release does not know. `as_call` names which, and that message is the whole value — a version skew fails on the one call that cannot be served rather than on the handshake.*
> —— 未知方法或本版本不认识的参数形状。`as_call` 指明是哪个，这条消息就是全部价值——版本不匹配在唯一无法服务的那个调用上失败，而不是在握手时。

#### 5. 路由决策

```rust
match route::route_for(&call) {
    Route::Refused => {
        Some(error_line(id, route::refusal(&call)))
    }
    Route::To(service, lane) => {
        match pool.send(service, lane, line).await {
            Ok(()) => None,
            Err(e) => {
                // The service is not there, or stopped reading. Answering is important:
                // a client that gets silence cannot tell "the robot is thinking" from
                // "nothing will ever come back", and `robotd` is the service most likely
                // to be missing because it is the one an update restarts.
                // —— 服务不在，或停止读取了。应答很重要：得到沉默的客户端
                //    无法区分"机器人在想"和"永远不会回来"，而 robotd 是最
                //    可能缺席的服务，因为它是更新时会被重启的那个。
                Some(error_line(id, ...))
            }
        }
    }
}
```

允许的调用转发到上游服务后返回 `None`——因为回复是服务异步发出的，经 `Pool` 的另一路通道回流到 `outbound`。本函数不等待回复。

### `error_line` — 构造错误行

```rust
fn error_line(id: Option<proto::Id>, error: proto::Error) -> String
```

通过 `proto::Response` 而非手写来构造，保证信封只有一个定义。

## 测试要点

测试体系设计得非常精巧：用真实的 unix socket 跑假守护进程，在真实 tokio runtime 上驱动 `run()`，完全不需要 WebRTC peer。

### `fake_daemon` — 假守护进程辅助函数

```rust
fn fake_daemon(
    dir: &Path,
    name: &str,
    replies: Vec<String>,
) -> mpsc::UnboundedReceiver<String>
```

在指定路径绑定 unix socket，读取每行、记录到 channel、回复预设的 canned 行。

### `a_permitted_call_reaches_its_service`

验证允许的调用到达正确服务的 socket，回复回到 peer。同时验证**没有走错服务**（`updater_seen.try_recv().is_err()`）。

### `a_refused_call_is_answered_here_and_forwarded_nowhere`

验证被拒绝的调用（如 `net.connect`）由 mediad 自己应答，**不经过任何 socket**。

> *`net.connect` is the interesting one: it is permitted over BLE and refused here, because reconfiguring wifi would take this session with it.*
> —— `net.connect` 是有意思的一个：它在 BLE 上被允许，在这里被拒绝，因为重新配网会带走这个会话。

### `the_pad_tap_reaches_padd`

验证 `pad.input` 到达 `padd`——这是 `btd` 刻意不持有 socket 的服务。且 `padd` 用流应答，本传输不为它发明回复。

### `every_notification_in_a_stream_reaches_the_peer`

> *This is the case that breaks if replies are ever correlated to requests.*
> —— 如果把回复和请求做关联，这个用例就会坏。

验证订阅流中的每条通知都到达 peer——三条 `update.progress` 全部送达，不会因为"只保留第一个回复"而丢失后续。

### `an_absent_service_is_reported`

> *`robotd` is the one most likely to be absent, because it is the one an update restarts.*
> —— `robotd` 是最可能缺席的，因为它是更新时会被重启的那个。

验证 socket 不存在时返回错误而非沉默。

### `an_unknown_method_names_itself`

验证未知方法（如 `robot.teleport`）的错误回复中点名方法名。

### `the_video_notification_carries_the_mount_rotation`

验证 `video_notification()` 输出正确的 JSON-RPC notification：无 id、带 width/height/rotate。

### `the_page_can_ask_what_the_video_is`

验证 `media.video` 作为调用被本文件直接应答，不路由到任何服务。

> *This is the path the console uses, and it exists because pushing the same information when the channel appears races the browser's datachannel and loses.*
> —— 这是控制台用的路径，它存在是因为通道出现时推送同样信息会和浏览器的 datachannel 竞速并输掉。

### `garbage_gets_a_parse_error`

验证非 JSON 输入返回 parse error，id 为 null。

## 与其他模块的关系

```
pipeline.rs
  │ webrtcsink "control" datachannel
  ▼
session.rs ──run()──▶ inbound / outbound (mpsc channel)
  │
  ├── handle()
  │     ├── route::route_for()  ──▶ route.rs (权限表)
  │     ├── pool.send()         ──▶ upstream::Pool (到各服务的 unix socket)
  │     └── media.video         ──▶ 本地应答
  │
  └── outbound ◀── Pool 的回复/通知回流
```

- **上游**：`pipeline.rs` 的 `webrtcsink` 在每个 peer 的 `"control"` datachannel 上把入站行送入 `inbound` mpsc。
- **路由**：调用 `route::route_for()` 和 `route::refusal()`。
- **下游**：经 `upstream::Pool`（连接五个后端服务的 unix socket 池）转发允许的调用，并接收服务回复/通知。
- **视频元数据**：`Video` 结构体携带 pipeline 的帧尺寸和旋转角度信息，由 pipeline 配置注入。
- **传输无关性**：本文件不依赖任何 WebRTC 类型，只吃 `String` 行、吐 `String` 行。WebSocket 表面可直接复用。
#（注：内容由AI生成）
