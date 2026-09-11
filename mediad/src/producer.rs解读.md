# `producer.rs` 解读

## 概述

`producer.rs`（213 行）负责回答一个问题：**"在任何对端开始协商之前，这台机器人怎么自我介绍？"**

GStreamer 的 `webrtcsink` 接受一个 `meta` 结构，信令服务器会把它放进 `list` 应答里——任何客户端在发起 WebRTC 连接之前，就能先看到"网络上有哪些机器人、它们分别是谁"。`producer` 就是这个 meta 结构的 Rust 侧构建器。

文件头把四个字段逐一列出，并为每个字段指了一个"调用者"：

```rust
//! Four fields, and each has a caller:
//! —— 四个字段，每个字段都有它的调用方：
//!
//! - **`name`** — so a page can title itself with the robot rather than with an id, and so a client
//!   that finds two producers on one network can say which is which. This is the whole of the
//!   field's value today.
//!   —— name：让页面用机器人名字而不是 id 做标题，让在一个网络里发现两台生产者的客户端能区分它们。
//!   这就是今天这个字段的全部价值。
//! - **`serial`** — the durable handle. A name is renamed and a peer id is per-session; the serial
//!   outlives both, which is what an app keying on a robot needs (`app-path-design.md` §8.6).
//!   —— serial：持久句柄。名字会被改，对端 id 是每次会话一个；serial 比两者都长寿，
//!   这正是按"机器人"为键的应用所需要的。
//! - **`release`** — what is running, so a client that behaves oddly against one robot can be told
//!   apart from a robot that is a release behind, without opening a session to ask.
//!   —— release：当前跑的是什么版本，让"客户端在某台机器人上表现异常"能和
//!   "那台机器人落后了一个 release"区分开，而且不用开一个会话去问。
//! - **`api_version`** — the same skew a session's `hello` reports, one round trip earlier. A
//!   client can put the banner up before it starts negotiating.
//!   —— api_version：和会话的 hello 消息报告的版本偏差是同一个，只是早一个往返。
//!   客户端可以在开始协商之前就把版本不一致的横幅挂出来。
```

## 关键常量

```rust
/// How long `configd` gets to say who this robot is.
/// —— configd 有多长时间来回答"这台机器人是谁"。
///
/// It is one unix-socket round trip against a daemon that answers `system.info` from memory, so
/// this is generous. It is spent once, before the pipeline exists, on a boot where every daemon
/// is starting at once — which is the only case where it is spent at all.
/// —— 这只是一次 unix socket 往返，而 configd 是从内存里回答 system.info 的，所以这个时间已经很宽松了。
/// 它只在 pipeline 还没建起来之前花一次，花在一次"所有守护进程同时启动"的开机上——这也是它唯一会被花掉的场景。
const ASK_TIMEOUT: Duration = Duration::from_secs(3);
```

`ASK_TIMEOUT = 3 秒` 的设计非常克制：

- **为什么这么短**：configd 是从内存回答的本地 unix socket，一次往返通常是毫秒级。3 秒已经留了上千倍余量。
- **为什么不能无限长**：mediad 是 `After=configd.service` 而不是 `Requires=configd.service`——也就是说 configd 没起来，mediad 也要起来。如果这里无限等，就把"`After=`"的解耦优势给抵消了。
- **只花一次**：注释强调"只在开机时问一次，之后改名要重启 mediad 才生效"。这与 `configd` 对 `btd` 广播的处理方式一致——广播不跟踪实时改名。

## 关键结构体

```rust
/// What a peer learns about this robot from the producer list.
/// —— 对端从生产者列表里能了解到的关于这台机器人的一切。
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Producer {
    /// As `configd` has it, or `None` when it did not answer in time.
    /// —— configd 给出的名字；configd 没及时回答时为 None。
    pub name: Option<String>,
    /// The SoC serial. `None` on a board with none to read, as `system.info` reports it.
    /// —— SoC 序列号。读不到序列号的板子上为 None，与 system.info 报告一致。
    pub serial: Option<String>,
    /// The release this process was installed as, or the build it was compiled from.
    /// —— 这个进程被安装成哪个 release，或者它是从哪个 build 编译出来的。
    pub release: String,
    pub api_version: u32,
}
```

字段设计有个非常值得注意的细节：**`name` 和 `serial` 是 `Option<String>`，而 `release` 和 `api_version` 是必填的**。原因写在 `fields()` 的注释里（见下文）——缺失的字段就**不放进 meta 结构**，而不是放一个空字符串。这样客户端不需要"空字符串 = 没有"的约定，直接看 key 在不在就行。

## 关键函数

### `Producer::local(build: proto::BuildInfo) -> Self`

```rust
/// What this process knows about itself, with nothing else asked.
/// —— 这个进程不问任何外部服务时，对自己的全部了解。
///
/// **The release, from the path rather than from the version.**
/// —— release 字段从**可执行文件路径**推导，而不是从 Cargo 版本号推导。
/// Every crate in this workspace shares one version line, so `0.9.1` names two different builds
/// on a dev channel and the directory a release installs into is what tells them apart — the same
/// reasoning [`proto::Identity::release`] exists for. A hand-built binary run from a home
/// directory has no such path, and then the compiled-in build string is the honest answer rather
/// than a version that implies a release nobody installed.
/// —— 这个 workspace 里所有 crate 共享同一行版本号，所以 `0.9.1` 在 dev 通道上可能指两个不同的 build，
/// 真正区分它们的是 release 安装到的那个目录——这正是 proto::Identity::release 存在的理由。
/// 一个从家目录里跑的手工编译二进制没有这种路径，那就用编译期 build 字符串作为诚实的答案，
/// 而不是一个暗示"有人装过某个 release"的版本号。
pub fn local(build: proto::BuildInfo) -> Self {
    let release = std::env::current_exe()
        .ok()
        .and_then(|exe| proto::release_from_path(&exe.display().to_string()))
        .map(|version| version.to_string())
        .unwrap_or_else(|| build.to_string());

    Self {
        name: None,
        serial: None,
        release,
        api_version: proto::API_VERSION,
    }
}
```

这是整个文件里**踩坑点最密集**的函数：

1. **为什么 release 不从 Cargo 版本号读**：整个 workspace 共享一个版本号，dev 通道上多个 build 都标 `0.9.1`，光看版本号根本区分不出"哪一个"。真正能区分 release 的是**安装目录**——官方 release 会装到 `/opt/duck/releases/0.9.1/` 这种路径，`proto::release_from_path` 就是从路径里反解出版本号的函数。
2. **为什么手工编译的二进制要用 build 字符串兜底**：如果是开发者 `cargo run` 出来的，可执行文件在 `target/debug/` 下，`release_from_path` 解不出版本号。这时候硬报一个 `0.9.1` 是撒谎——因为没人真的装过 0.9.1 release。诚实的做法是用编译期 `BuildInfo`（git revision、构建时间等）。
3. **`local` 与 `learn` 的关系**：`local` 是"不问任何人的自己"，`learn` 是"`local` + 问 configd 拿到的 name/serial"。分层得很干净。

### `Producer::learn(sockets: Sockets, build: proto::BuildInfo) -> Self`

```rust
/// [`Producer::local`], plus whatever `configd` says about the name and the serial.
/// —— local 的结果，再加上 configd 告诉它的 name 和 serial。
pub async fn learn(sockets: Sockets, build: proto::BuildInfo) -> Self {
    let mut producer = Self::local(build);

    match ask_configd(sockets).await {
        Ok(info) => {
            producer.name = Some(info.name);
            producer.serial = info.serial;
        }
        // At `warn` rather than `error`: the pipeline still starts and still streams, and the
        // only cost is a producer a client cannot name. The reason is in the message because
        // "no socket" and "answered something unexpected" want different next moves.
        // —— 用 warn 而不是 error：pipeline 照样启动、照样推流，唯一的代价是生产者报不出名字。
        // 日志里要带原因，因为"没有 socket"和"回答了意料之外的东西"需要不同的下一步动作。
        Err(e) => tracing::warn!(
            error = %e,
            "configd did not say who this robot is; the producer will have no name"
        ),
    }
    producer
}
```

文件头注释解释了为什么"问不到名字也要继续"：

```rust
//! `system.info` owns the name and the serial, and `mediad` has a connection to `configd` for it
//! already. But `mediad` starts alongside `configd` rather than after it, and the unit says so
//! deliberately — `After=` and not `Requires=`, because a service being down must not keep this
//! one from starting. So this asks, waits [`ASK_TIMEOUT`], and goes on with what it knows about
//! itself either way: a producer with no name is a robot that streams, and a robot that will not
//! stream because it could not learn its own name would be a much worse trade.
//! —— system.info 拥有 name 和 serial，mediad 也已经有到 configd 的连接。
//! 但 mediad 和 configd 是**同时**启动的，而不是在它之后——unit 文件里故意写的是 After= 而不是 Requires=，
//! 因为一个服务挂了不能让这个服务也起不来。所以这里问一次，等 ASK_TIMEOUT，然后无论结果如何都继续用
//! 自己知道的东西：一个没有名字的生产者，至少还是一台会推流的机器人；
//! 而一台因为不知道自己叫什么就不推流的机器人，是一笔糟糕得多的交易。
```

这段注释是整个文件的哲学核心：**机器人的首要责任是"能被看到"，不是"知道自己叫什么"**。这与 `config.rs` 的"配置读不动也要推流"、`web.rs` 的"控制台挂了视频还在"完全同源。

### `Producer::fields(&self) -> Vec<(&'static str, String)>`

```rust
/// The `meta` fields, as the pairs a `GstStructure` is built from.
/// —— meta 字段，按 GstStructure 构造时需要的 (键, 值) 对形式返回。
///
/// Absent fields are absent rather than empty strings: a client reading `serial: ""` has to know
/// that means "none", where a missing key needs no convention. Keys are snake_case, as every other
/// field on this wire is.
/// —— 缺失的字段就**不存在**，而不是空字符串：客户端读到 `serial: ""` 还得知道这意思是"没有"，
/// 而一个根本不存在的 key 不需要任何约定。键名用 snake_case，与这条线上的其他所有字段保持一致。
pub fn fields(&self) -> Vec<(&'static str, String)> {
    let mut fields = Vec::new();
    if let Some(name) = &self.name {
        fields.push(("name", name.clone()));
    }
    if let Some(serial) = &self.serial {
        fields.push(("serial", serial.clone()));
    }
    fields.push(("release", self.release.clone()));
    fields.push(("api_version", self.api_version.to_string()));
    fields
}
```

设计要点：

- **Option 字段不进 Vec**：`None` 时不 push，GStreamer `GstStructure` 里就根本没有这个 key。客户端用"key 是否存在"判断，不需要"空字符串 = 无"的隐性约定。这是一个非常成熟的 API 设计选择——把"可选"这件事编码进数据形状，而不是编码进字符串值。
- **键名统一 snake_case**：注释特意强调，因为 GStreamer `GstStructure` 的键名习惯上是驼峰，而本协议其他字段都是 snake_case。这里**打破 GStreamer 习惯，跟本协议走**，避免客户端要写两套解析规则。
- **`release` 和 `api_version` 永远在**：它们是 `String`/`u32` 而非 `Option`，因为本进程永远知道自己编译自哪个 build、说的是哪个 API 版本。

### `ask_configd(sockets: Sockets) -> Result<proto::SystemInfoResult, String>`

```rust
/// One `system.info` call, and the first line back.
/// —— 一次 system.info 调用，以及回来的第一行。
///
/// A [`Pool`] rather than a socket opened here, so this shares the connect and write timeouts
/// every other call to a service gets rather than growing a second set of them.
/// —— 用 Pool 而不是在这里开一个新 socket，这样就能复用其他所有对服务的调用所共用的连接超时和写超时，
/// 而不是新长出第二套超时参数。
async fn ask_configd(sockets: Sockets) -> Result<proto::SystemInfoResult, String> {
    let (replies_tx, mut replies_rx) = mpsc::channel::<String>(4);
    let mut pool = Pool::new(sockets, replies_tx);

    let request = serde_json::json!({
        "jsonrpc": "2.0",
        "id": 1,
        "method": proto::method::SYSTEM_INFO,
        "params": {},
    })
    .to_string();

    pool.send(proto::Service::Config, proto::Lane::Prompt, &request)
        .await
        .map_err(|e| format!("could not ask configd: {e}"))?;

    let line = tokio::time::timeout(ASK_TIMEOUT, replies_rx.recv())
        .await
        .map_err(|_| format!("configd did not answer within {ASK_TIMEOUT:?}"))?
        .ok_or_else(|| "configd closed the connection".to_owned())?;

    let reply: serde_json::Value =
        serde_json::from_str(&line).map_err(|e| format!("configd answered nonsense: {e}"))?;
    if let Some(error) = reply.get("error") {
        return Err(format!("configd refused system.info: {error}"));
    }
    serde_json::from_value(reply["result"].clone())
        .map_err(|e| format!("configd's system.info does not fit: {e}"))
}
```

这是一个**最小化 JSON-RPC 客户端**，值得逐行看：

1. **用 `Pool` 而不是裸 socket**：`upstream::Pool` 是 mediad 自己的连接池抽象，管所有到其他守护进程的连接。复用它意味着连接超时、写超时、重连策略全部与其他调用一致——不会因为这里单独开 socket 就长出第二套超时魔数。
2. **JSON-RPC 2.0 手写请求**：`serde_json::json!` 宏现场构造请求，`id: 1`（因为只发一次），方法名走 `proto::method::SYSTEM_INFO` 常量。
3. **错误信息逐阶段定制**：
   - "could not ask configd" — 连接/发送失败；
   - "configd did not answer within 3s" — 超时；
   - "configd closed the connection" — 对端主动关闭；
   - "configd answered nonsense" — 回的不是 JSON；
   - "configd refused system.info" — JSON-RPC error 字段非空；
   - "configd's system.info does not fit" — JSON 对但 schema 不对。
   
   这与 `learn` 函数里"日志里要带原因，因为不同失败需要不同下一步"的设计意图完全一致——每一种失败的措辞都对应一种运维动作。
4. **`Result<_, String>` 而不是 `anyhow::Error`**：这个函数是模块内部的，错误就是一句人读得懂的话，不需要结构化错误类型。

## 测试要点

4 个测试：

1. **`what_is_known_without_asking_anything`** —— `Producer::local(build())` 不联网，断言 `api_version == proto::API_VERSION`、`release` 非空、`name` 是 `None`。锁定"不问 configd 也能造出一个 producer"。
2. **`absent_fields_are_absent`** —— 未命名机器人调 `fields()`，断言键列表是 `["release", "api_version"]`，**只有两个键**，不是四个键两个空字符串。这是把"缺失字段就不放 key"的设计钉死成测试。
3. **`a_named_robot_publishes_the_lot`** —— 手工构造一个带 name/serial 的 producer，断言 `fields()` 返回 `["name", "serial", "release", "api_version"]` 四键，且第一个值是 `"duck-c51b"`。
4. **`a_silent_configd_still_yields_a_producer`**（`#[tokio::test]`）—— 把 `Sockets.config` 指向一个不存在的路径 `/nonexistent/configd.sock`，调 `Producer::learn`，断言 `name == None` 但 `api_version` 仍然正确。注释点明："这就是开机场景——mediad 是 `After=configd.service` 而不是 `Requires=`，所以它完全可能、也确实会先于 configd 启动。"

测试覆盖了 producer 生命周期的三条路径：完全不联网、有名字、configd 沉默。

## 与其他模块的关系

- **`upstream::Pool` / `Sockets`**：通过它们问 configd。连接池、超时、lane 抽象都来自这里。
- **`duck_ipc_proto as proto`**：协议常量来源——`API_VERSION`、`BuildInfo`、`SystemInfoResult`、`method::SYSTEM_INFO`、`Service::Config`、`Lane::Prompt`、`release_from_path`。
- **`pipeline`**：把 `Producer::fields()` 的返回值喂给 GStreamer `webrtcsink` 的 meta 结构。pipeline 是 producer 的唯一消费者。
- **`configd`（外部守护进程）**：`system.info` 的服务方，提供 name 和 serial。mediad 对它的态度是"能问就问，问不到就算了"。
- **`web`**：前端控制台会读 webrtcsink `list` 应答里的 meta 字段，用 `name` 做页面标题、用 `api_version` 决定是否挂出版本不一致横幅。

## 设计哲学小结

`producer.rs` 把一个看似简单的"自我介绍"问题，拆成了三个层次的容错设计：

1. **身份分层**：永远知道的（release、api_version） vs. 需要问别人的（name、serial）。前者必填，后者可选。
2. **失败降级**：问不到 configd → warn 日志 + 不带名字继续推流，而不是拒绝启动。
3. **数据编码**：缺失字段从 key 集合里消失，而不是填空字符串，把"可选"编码进数据形状而不是值。

这三点合起来回答了一个工程上的根本问题：**机器人在启动顺序不确定、外部服务可能缺席的情况下，怎么向网络宣告自己？** 答案是——"先把能确定的都说出来，不确定的就不说，绝不因为不确定就闭嘴"。这与 `config.rs`、`web.rs` 的可靠性取向完全一致，构成了 mediad 整个 crate 的性格。
#（注：内容由AI生成）
