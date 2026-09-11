# `web.rs` 解读

## 概述

`web.rs`（181 行）是 `mediad` 内置的 **Web 控制台服务器**——它在守护进程自己的进程里开一个 HTTP 端口，把控制台页面直接送到浏览器。整个模块的设计目标写在文件头：

> One route, one file, no build step. `http://<robot>:8080/` and there is nothing else to run — which is the whole of it.
> —— 一个路由、一个文件、没有构建步骤。`http://<robot>:8080/`，除此之外什么都不用跑——这就是全部。

它在 crate 中的角色非常克制：**只服务一个 HTML 页面，只暴露一个 GET `/` 路由**。但文件头用了一大段注释论证"为什么值得为此引入 axum 依赖、再开一个端口"，因为这一步看似简单，实际上一次性解决了四个部署问题。

## 核心设计：为什么由守护进程自己服务页面

文件头列出了四个被一次性消除的问题：

```rust
//! **It deletes four problems rather than one.**
//! —— 它一次消除了四个问题，而不是一个。
//! No `python3 -m http.server`, so the instruction is an address rather than two commands.
//! —— 不需要 `python3 -m http.server`，于是给用户的指引是一个地址而不是两条命令。
//! No URL to type, because a page served by the robot knows which robot it came from and derives
//! its signalling target from `location.hostname`.
//! —— 不需要手敲 URL，因为由机器人服务的页面天然知道自己来自哪台机器人，会从 `location.hostname`
//! 推出信令服务器地址。
//! Chrome's Private Network Access check stops applying, because that check is on requests from a
//! *public or opaque* origin to a private address — and a page served from `192.168.x` has a
//! private one, so the failure that made `file://` unusable cannot happen.
//! —— Chrome 的"私有网络访问"（PNA）检查不再适用：那个检查针对的是"从 public 或 opaque origin
//! 访问私有地址"，而从 `192.168.x` 服务出来的页面本身就是私有 origin，于是 `file://` 时代那种
//! 必现失败根本不会发生。
//! And page and binary ship together, so a client from a checkout can no longer be pointed at a
//! robot from a release.
//! —— 而且页面和二进制一起发布，从源码 checkout 里拿出来的客户端再也不可能被错误地指到一台
//! release 版机器人上。
```

这四点是理解整个模块的钥匙：

1. **易用性**：用户只需要在浏览器输入一个地址，不用先在笔记本上跑一个临时 HTTP 服务器。
2. **零配置信令地址**：页面自己从 `location.hostname` 推导信令服务器，不需要把机器人 IP 写进页面。
3. **绕开 Chrome PNA 限制**：这是最硬核的一点。`file://` 打开的页面是 "opaque origin"，Chrome 会拦截它访问 `192.168.x` 这种私有地址的 WebRTC 信令；而由机器人自己服务的页面 origin 是 `http://192.168.x:8080`，属于私有 origin，PNA 检查直接不适用。
4. **版本一致性**：页面与二进制同发同版，杜绝了"新版页面连旧版机器人"的诊断噩梦。

## 关键常量

```rust
/// The page as it sits in the source tree.
/// —— 页面在源码树里的样子。
const PAGE: &str = include_str!("../webclient/index.html");

/// Where the page carries its signalling port until this module fills one in.
/// —— 在本模块填充之前，页面里占位用的信令端口 token。
const PORT_TOKEN: &str = "{{SIGNALLING_PORT}}";

/// Where it carries the API version this release speaks.
/// —— 页面里占位用的 API 版本 token。
const API_TOKEN: &str = "{{API_VERSION}}";
```

### `PAGE` 为什么用 `include_str!` 而不是运行时读文件

这段注释是整个文件里最值得摘录的踩坑点：

```rust
/// `include_str!` rather than a file read at request time, and that is a decision: installing it
/// under `current/webclient/` would cost an `--include` line in three places that already drift
/// (`_build-release.yml`, `dev.yml`, `scripts/dev-push.sh`), put a filesystem read behind a
/// network request in a unit running `ProtectSystem=strict`, and make "which page is this robot
/// serving" a question with two answers. The cost is a rebuild to change a stylesheet, which is
/// the right trade for a page that is part of the daemon's interface.
/// —— 用 `include_str!` 而不是在请求时读文件，这是一个有意的决定：
/// 把它安装到 `current/webclient/` 下，就要在三处已经在漂移的地方加 `--include` 行
/// （`_build-release.yml`、`dev.yml`、`scripts/dev-push.sh`），还要在一个跑着
/// `ProtectSystem=strict` 的 systemd unit 里、把一次文件系统读取藏在一个网络请求后面，
/// 并且让"这台机器人现在服务的是哪个页面"变成一个有两个答案的问题。
/// 代价是改一下样式表就要重新编译——对于"本就是守护进程界面一部分"的页面，这是对的权衡。
```

设计权衡非常清晰：

- **反对外部文件**：安装路径要在三处 CI/部署脚本里同步，历史已经证明这三处会漂移；运行时读文件还会被 `ProtectSystem=strict` systemd 沙箱挡一次；外部文件还会带来"磁盘上这份文件到底是不是二进制里那份"的歧义。
- **接受内嵌的代价**：改 CSS 也要重新编译。但页面本身就是守护进程界面的一部分，本来就该随版本一起发布，所以这个代价是可接受的。

## 关键函数

### `pub fn page(signalling_port: u32) -> String`

```rust
/// The page, with the signalling port and the API version filled in.
/// —— 把信令端口和 API 版本填进去之后的页面。
pub fn page(signalling_port: u32) -> String {
    PAGE.replace(PORT_TOKEN, &signalling_port.to_string())
        .replace(API_TOKEN, &proto::API_VERSION.to_string())
}
```

启动时做一次字符串替换，把两个 token 换成真实值。设计要点：

- **端口只在这里出现一次**：注释说"`--port` 之所以改起来安全，是因为没有任何地方持有第二份拷贝"。页面里没有硬编码 8443，所有端口信息都从这一处流入。
- **API 版本也在这里填，不写死在页面里**：注释解释得很到位——"页面不知道自己说的是哪个版本，就没法做版本比对；如果在页面里写一个字面量，那就是 `proto::API_VERSION` 的第二份拷贝，在版本号 bump 的那天就会错，而且错的方向是**报告一致**"。也就是说，字面量会导致页面以为自己和机器人版本匹配，实际上不匹配——这种"假阳性"比直接报错危险得多。
- **host 由浏览器填**：端口是机器人告诉页面的，但 host（哪台机器人）是浏览器通过 `location.hostname` 自己知道的。这就是为什么"只需要传一个数字而不是一个完整 URL"。

### 关于 token 未替换的回退行为

文件头还有一段非常精彩的容错设计：

```rust
//! A page opened straight from the source tree keeps the token, reads it as `NaN`, and falls back
//! to `webrtcsink`'s own 8443. It also then knows it was *not* served by a robot, which is what
//! lets it tell "wrong host" apart from "the robot served me but its signalling port did not
//! answer" — the one failure two ports can produce, and §1.3's requirement that it reach a person
//! as a diagnosis.
//! —— 直接从源码树打开的页面会保留 token，把它读成 `NaN`，然后回退到 webrtcsink 自己的 8443。
//! 同时它也因此知道自己**不是**由机器人服务的，这让它能区分"host 写错了"和
//! "机器人服务了我，但它的信令端口没回应"——这是两个端口会产生的唯一一种失败，
//! 而设计文档 §1.3 要求这种失败必须以诊断信息的形式到达人眼前。
```

也就是说，页面里的 token 如果没被替换（例如开发者直接双击打开 `index.html`），前端 JS 会把 `{{SIGNALLING_PORT}}` 解析成 `NaN`，然后回退到默认 8443，并且知道自己"不是被机器人服务的"。这让前端能区分两种不同的故障：

- "我打开的是源码树里的文件" → 提示用户直接访问机器人地址；
- "机器人服务了我，但信令端口没响应" → 提示机器人内部出问题了。

### `pub async fn serve(host: &str, port: u16, page: String) -> Result<()>`

```rust
/// Serve `page` on `host:port` until the process ends.
/// —— 在 host:port 上服务 page，直到进程结束。
///
/// Returns only on failure — a bind that was refused, or a listener that died. The caller decides
/// what that costs; in `mediad` it costs the page and not the video, because a robot that streams
/// and answers control calls with no console is a great deal better than one that does neither.
/// —— 只在失败时返回——bind 被拒绝，或者监听器死掉。调用方决定失败的代价；
/// 在 mediad 里，这个代价是"没了控制台"，而不是"没了视频"，因为一台还在推流、还能应答控制调用
/// 但没有控制台的机器人，远比一台两样都没了的机器人好得多。
pub async fn serve(host: &str, port: u16, page: String) -> Result<()> {
    let address: SocketAddr = format!("{host}:{port}")
        .parse()
        .with_context(|| format!("{host}:{port} is not an address to listen on"))?;
    let listener = tokio::net::TcpListener::bind(address)
        .await
        .with_context(|| format!("could not listen on {address}"))?;

    tracing::info!(%address, "serving the console");
    axum::serve(listener, router(page))
        .await
        .context("the console's listener stopped")
}
```

关键设计：**控制台失败不拖垮视频**。`serve` 是一个独立的 async 任务，`main.rs` 把它和视频管线的任务并列 `tokio::spawn`。如果 8080 端口被占用，控制台挂掉，但视频管线继续跑。这与 `config.rs` 的"配置读不动也要推流"哲学一脉相承——**可观测性是次要的，机器人本身能跑才是第一位**。

错误信息用 `anyhow::Context` 包装得很具体："`{host}:{port} is not an address to listen on`"（不是个能监听的地址）、"could not listen on {address}"（监听失败）、"the console's listener stopped"（监听器停了）——每一种失败都有专属文案，运维看到日志能直接定位。

### `fn router(page: String) -> Router`

```rust
/// One route, returning `page`.
/// —— 一个路由，返回 page。
fn router(page: String) -> Router {
    Router::new().route("/", get(move || std::future::ready(Html(page))))
}
```

整个 axum 应用就是这么一行：挂一个 `GET /` handler，返回 `Html(page)`。`std::future::ready(Html(page))` 是因为 handler 不需要做任何异步工作——页面是早就替换好的字符串，直接包成一个 ready future 即可。`move` 把 `page` 字符串 move 进闭包，整个 server 持有这份页面的拷贝。

## 测试要点

5 个测试，覆盖面非常精巧：

1. **`both_tokens_are_filled_in`** —— 调 `page(8443)`，断言返回值里**不再包含** `{{SIGNALLING_PORT}}` 和 `{{API_VERSION}}` 两个 token，并且包含 `"8443"` 和 `proto::API_VERSION` 的字符串形式。这是核心承诺："页面不能带着 token 上线"。
2. **`the_page_carries_the_api_token_this_module_replaces`** —— 反向断言：**原始 `PAGE` 字符串里必须真的有 `{{API_VERSION}}` 这个 token**。如果有人改了 `webclient/index.html` 把 token 改了个名，这个测试立刻红。它保护的是"替换逻辑和页面模板之间的契约"。
3. **`a_moved_port_reaches_the_page`** —— `page(9000)` 必须包含 `"9000"`。对应文件头那句"`--port` 之所以改起来安全，是因为没有第二份拷贝"。
4. **`the_route_answers_with_the_page`**（`#[tokio::test]`）—— **端到端真 socket 测试**。绑一个 loopback 临时端口，spawn axum server，自己手写 HTTP 请求 `GET / HTTP/1.1\r\nHost: robot\r\n...`，断言响应以 `HTTP/1.1 200` 开头、包含 `"duck console"`、不残留端口 token。注释说这是"浏览器做的全部事情，成本极低；否则就要等上板子才发现路由挂在一个浏览器不会问的地方"。
5. **`the_page_carries_the_token_this_module_replaces`** —— 与第 2 条对称，断言原始 `PAGE` 里有 `{{SIGNALLING_PORT}}`。注释特别点出："没有这条测试，任何一边改了名，页面都会静默回退到 8443——在你换端口之前一切正常，然后毫无征兆地挂掉，日志里啥也读不出来"。

这组测试的设计思路是**双向锁**：既测试"模块输出里 token 被替换掉了"，又测试"页面模板里 token 还在等着被替换"。任何一端单方面改名都会立刻失败，避免静默漂移。

## 与其他模块的关系

- **`webclient/index.html`**：通过 `include_str!` 编译期嵌入。前端 JS 从 `location.hostname` 推信令地址，从替换后的端口号连 WebRTC 信令服务器。
- **`duck_ipc_proto as proto`**：只用了 `proto::API_VERSION`，用于填版本 token。
- **`main.rs`**：调用 `serve(host, port, page(port))`，把它和 pipeline 任务并列 spawn。
- **`pipeline`**：不直接依赖，但前端页面连上的信令服务器就跑在 pipeline 的 `webrtcsink` 里。两个端口——8080（控制台）和 8443（信令）——是两个独立失败点，前端要能区分它们。
- **`config`**：不依赖。控制台的 `--port` 是命令行参数，不是 TOML 配置。

## 设计哲学小结

`web.rs` 是一个典型的"**用极小的代码量消除一大类运维问题**"的模块：

- 引入 axum + 一个端口，换来"给用户一个地址就够了"；
- 用 `include_str!` 换来"页面和二进制永远同版"；
- 用启动时 token 替换，换来"`--port` 改起来全局一致"；
- 用独立 spawn 任务，换来"控制台挂了视频还在跑"；
- 用双向 token 测试，换来"模板和替换逻辑不会静默漂移"。

整个文件反复出现的主题是：**机器人已经出问题的时候，让人能打开浏览器看到它本身**。这与 `config.rs` 的降级哲学、`producer.rs` 的"问不到名字也要推流"哲学完全一致。
#（注：内容由AI生成）
