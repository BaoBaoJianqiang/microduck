# WebRTC 客户端：从测试页面到机器人控制台

状态：已落地 · 日期：2026-08-25 · 负责人：pierre

`mediad/webclient/index.html` 证明了传输。本文是把它变成人用的东西：由机器人而非 `python3` 服务、通过 `btd` 已经广播的地址发现、并演练 `route.rs` 实际允许的控制表面。

本文是 [`remote-webrtc.md`](remote-webrtc.md) 的姊妹篇，后者负责传输——信令、会话模型、授权和对端可调用什么。两者相交处，那一页是负责人，本文指向它。

**全部四项变更都已入。** 本页在它们第一行代码之前写就，所以决策是可论证的而非由 diff 暗示；保留它是因为替代方案是值得能重读的部分。代码和本页可能漂移的地方，代码是答案：`mediad/src/web.rs` 服务页面，`mediad/src/producer.rs` 填 `meta`，`duckctl` 的 `ip` 和 `open` 发现机器人，`mediad/webclient/index.html` 是控制台。§7 说还剩什么。

## 0. 它有什么问题

对它的用途来说，不多。它回答了"浏览器能拿到视频和数据通道吗"，答案是能。作为*一个客户端*它有六个问题：

| | |
|---|---|
| it needs `python3 -m http.server` | a second tool and a second terminal, and the page is served from the laptop to talk to the robot |
| the URL is typed by hand | `ws://radxa-zero3.local:8443` — and `radxa-zero3` is the hostname on **every** board flashed from one image (`configd/src/main.rs` says so where it falls back to it), so two robots on one network collide |
| its comment block is mostly warnings | `file://` and Private Network Access, http not https — a header that had to be corrected twice already (`7f52a34`) |
| it exercises the protocol, not the robot | `route.rs` permits move, head, look, pose, mouth, do, sound, enable, init, relax, stop, subscribe, `tof.stream`, `pad.input`. The page offers `robot.health` and a JSON textbox |
| it is shipped nowhere | no `--include` names it, so a robot in the field has no client |
| it cannot say which robot it reached | the producer's `meta` is unset, so `list` returns an id and nothing else |

其中每一个都是人和能用的机器人之间的一步，且没有一个是关于 WebRTC 的。

## 1. `mediad` 服务页面

**已落地** —— `mediad/src/web.rs`，`--web-port`，默认 8080。

`mediad` 里的一个 HTTP 监听器，一条路由，返回页面。`http://<robot>:8080/`，没有别的要跑。

这一次删掉四个问题而非一个：

- **无 `python3`。** 指令变成一个地址。
- **无 URL 要打。** 页面把它的信令目标默认为 `ws://${location.hostname}:8443`——它由机器人服务，所以它知道在跟哪个机器人说话。
- **Private Network Access 不再适用。** Chrome 拦截从*公共或不透明*源到私有地址的请求。从 `192.168.x` 服务的页面有私有源，所以破坏 `file://` 的检查根本不到达。头部的警告块消失因为它警告的失败不可能发生。
- **一个版本。** 页面和二进制一起发货（§1.2），所以来自 checkout 的客户端不能再指向来自 release 的机器人。

### 1.1 哪个 HTTP 服务器

**`axum`。已定。** 它已在 `Cargo.lock` 里——`updater` 把它用作测试镜像的 dev-dependency——所以该 crate 针对此工具链和交叉构建已知良好。

替代方案是手写 HTTP/1.1 响应器：这只通过一个方法服务一个文件，所以大概六十行。六十行绑定到 `0.0.0.0` 的手写请求解析是一个暴露给网络上每个人的解析器，写来避免一个构建已解析的依赖——而 `axum` 底下的 `hyper` 是该语言中该解析器被读得最多的实现。

`mediad.service` 里的 `RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6` 已允许监听器；unit 里什么都不变。

### 1.2 页面是嵌入的，不是安装的

`include_str!("../webclient/index.html")`，从内存服务。

替代方案——把它装在 `/opt/robot/daemon/current/webclient/` 下并在请求时读取——要在*三个*地方（`_build-release.yml`、`dev.yml`、`scripts/dev-push.sh`）加一行 `--include`，`xtask` 的打包绊线存在就是为了让它们同步，而那正是之前漂移过的列表。它还在跑 `ProtectSystem=strict` 的 unit 里把文件系统读取放在网络请求后面，且让"这台机器人在服务哪个页面"成为有两个答案的问题。

嵌入的代价是改样式表要重新构建。对作为 daemon 接口一部分的页面来说这是正确的权衡。

### 1.3 两个端口，且没人需要知道有两个

`webrtcsink` 拥有 8443 上的监听器（`run-signalling-server`，只有 `-host` 和 `-port` 可说），所以页面不能是它上的一条路由。两个端口：8080 服务页面，8443 保持信令服务器。媒体路径什么都不变，这正是先这么做的全部理由。

**两个端口是实现的事实，且它绝不能成为人的一步。** 三件事让它保持如此，它们是对变更的要求而非希望：

- **只打一个地址。** `http://<robot>:8080`。8443 由页面自己的 JavaScript 到达，从不由人。`duckctl open`（§2）甚至连那一个都去掉。
- **`mediad` 在服务页面时填入信令 URL**，而非页面携带常量。它知道请求到达的主机且知道自己的 `--port`，所以页面得到真实答案——启动时对嵌入字符串做一次 `str::replace`。这就是让 `--port` 可安全更改的原因：没有第二个地方持有它的过时副本。
- **两个端口能产生的那个失败用文字点名。** 如果 8080 应答而 8443 不应答——最可能是中间有防火墙——页面必须说*页面来自这台机器人，但它的信令端口不应答*，而非 `websocket error`。这是第二个端口能到达人的唯一途径，且它应以诊断形式到达。

单端口变体是自己跑信令服务器——上游把它作为库与插件一起发布——并把 `webrtcsink` 的 signaller 指向 `ws://127.0.0.1:8080/ws`。然后一个 `axum` 服务器在一个源上服务页面和协议。更多工作，且要与我们发货的 `.so` 保持同步的第二份协议版本，所以不在第一次变更里。

**但这是它的终点，原因如下。** 浏览器不给通过纯 http 服务到私有地址的页面麦克风，且取决于浏览器也不给手柄——那些是安全上下文 API，而 `http://192.168.1.42` 不是安全上下文（`http://localhost` 是，这正是为什么还没人碰到这个）。双向音频在 `remote-webrtc.md` §2。机器人将需要服务 TLS，而 `webrtcsink` 的内置服务器不提供配置证书的方式，`axum` 服务器提供普通方式。所以顺序是：现在两个端口，想要音频或浏览器手柄时自己跑信令服务器，同一天上 TLS。

值得现在写下来而非在接麦克风时才发现。

## 2. 发现机器人：两个命令，其中一个已是手写的

**已落地** —— `duckctl ip` 和 `duckctl open`，广告优先，`net.status` 兜底。

`btd` 把机器人的 IPv4 填进它的广告，company id `0xFFFF`，而 `duckctl` 已经解析它——`Address::At`、`Unassigned`、`Unsaid`，三个答案而非两个，且 `scan` 今天打印它。所以工作是一个命令，不是一个机制。

### 2.1 `duckctl ip`

机器人地址输出到 stdout 且什么都没有，所以 `ssh radxa@$(duckctl ip)` 能工作——工具已经保持的分离，诊断在 stderr 数据在 stdout。

**这在本仓库不是新想法；是一个已经写得很糟的想法。** `scripts/dev-push.sh` 正需要这个且手写了它：`resolve_board()` 调用 `duckctl wifi status` 并把 JSON 通过嵌在 shell 脚本里的六行 Python 程序管道取出 `result.ip4`。那就是这个命令，只差一个家。

读广告而非调用 `net.status` 在三点上更好，全部可见于 `dev-push.sh` 不得不写的东西：

- **无连接，所以无绑定无 PIN。** `resolve_board` 带着一整个"机器人拒绝 `net.status`——通常是 PIN 错"的失败分支。广告读取不能被拒绝，所以那个分支消失。
- **秒级而非数十秒。** `dev-push.sh` 按机器人缓存地址恰恰因为"BLE 发现要十到二十秒"；在第一个匹配广告处停止的扫描约一秒，因为 `duckctl` 已经轮询直到有东西出现而非睡过 `SCAN_TIME`。
- **不过时。** `btd` 的 `reconcile_advertisement` 每 `ADV_POLL`（5s）重读 `net.status` 并在答案变化时重新广告，所以广告*就是*带五秒延迟的 `net.status`——远在新租约已经断开 ssh 的窗口内。

**带一个兜底，非可选。** 绑定到这台 Mac 的机器人经常停止向它广告服务——`duckctl` 自己的扫描分层就是为此——所以看不到广告时，`ip` 连接并问 `net.status`，那正是 `dev-push.sh` 今天做的。先廉价读取，再调用。没有兜底这个命令会在最常用它的笔记本上失败。

三态 `Address` 已经携带正确的失败文本，这正是它的回报：

- `Unassigned` —— 机器人没有网络。修复是 `duckctl wifi connect`，且必须通过 BLE，因为 `net.connect` 被设计为通过 WebRTC 拒绝（"从未见过网络的机器人不能通过那个网络配置"）。
- `Unsaid` —— 早于 `btd` 广告地址的 release。兜底仍能回答；更新让它变快。

它的第三个调用者不是上面两个：`install-dev.md` 开头要"板子的 **IP 地址**"，附说明此镜像的 mDNS 不可靠，且不提供获取方式。

### 2.2 `duckctl open`

解析，然后在浏览器打开 `http://<address>:8080/`。`--print` 改为打印 URL，用于无浏览器的机器或脚本；`--port` 用于以非默认 `--web-port` 启动的机器人。

一个命令而非文档化的 shell 替换，原因只有一个：**端口默认应住在唯一一处人从不需要读的地方。** `open "http://$(duckctl ip):8080"` 能用，且是那种人写一次然后永远查的行。

### 2.3 不是这些

- **`duckctl url`。** 那是 `open --print`。一个整个内容是端口号的第三个命令。
- **`scan` 上加 URL 列。** `scan` 也列耳机，且机器人行已经带地址。列表下一条指向 `open` 的注释够了。
- **任何必须连接的东西。** 两个命令都是广告读取加兜底。这里一个*要求*绑定的命令会是不同种类的命令，它该和 `wifi`、`update` 在一起而非和发现机器人在一起。

### 2.4 这关上的循环，以及一个后续

BLE 配置网络然后把它的 URL 交给你。两种传输不再是替代而是序列——值得一说因为 `route.rs` 刻意通过 WebRTC 拒绝 `net.connect`，这是那个拒绝的另一半。

后续，在它自己的变更而非这四个里：`dev-push.sh` 的 `resolve_board` 变成 `client --name "$1" ip`，删掉嵌入 Python 和 PIN 错分支。分开因为它触碰推送路径，且因为它应在 `ip` 被手动用过几次之后落地。

`duckctl` 是手机 app 的权宜，所以这里保持两个命令且无新机制。持久的两半是比它活得久的：`btd` 广播地址，`mediad` 在已知端口服务。App 会原生做同样两步。

## 3. 工具以它即将不再是的传输命名

**已定且已落地** —— `duck-btctl` 是 `duckctl`，在它自己的 crate 里。本节其余是推理，保留因为替代方案是值得能重读的部分。

`open` 在 http URL 启动浏览器，而做这件事的工具的名字写着 `bt`。值得拉一拉，因为它指向比一个命令更大的东西。

**`open` 本身没放错。** 它的实质*是*蓝牙：它扫描广告以得知机器人在哪，末尾的 `xdg-open` 是一行。`duckctl open` 读起来是"用无线电找机器人，然后给我看它的控制台"，正是它做的——就像 `wifi connect` 是一个 wifi 命令但其整个机制是 BLE。

**名字仍是错的，原因随 `mediad` 而非 `open` 到来。** 即将有到这台机器人的第二个传输，且两者按设计到达*不同*的方法集：`robot.move` 通过 BLE 被拒绝通过 WebRTC 被允许；`net.connect` 反过来。所以人会需要两者，而他们真正想要的不是选择二进制——是一个知道哪个传输能服务调用且在两者都不能时说明的工具：

> `wifi connect` 是仅蓝牙的，且这台机器人不在广告。

那个工具不能叫 `btctl`。而叫 `btctl` 的工具悄悄教每个人蓝牙是联系机器人的方式，恰在它不再是唯一方式的时刻。

### 3.1 是搬家，不是改名，且这修了两个现存的疙瘩

传输无关的客户端不能留在 `btd/examples/duck-btctl.rs`。它会需要 `btleplug` *和*一个 WebSocket 客户端，而 `btd` 是 BLE daemon——它的 example 是其中一半的错误归宿。所以诚实的形态是它自己的工作区成员，路上两个今天别扭的东西不再别扭：

- **安装行。** `cargo install --path btd --example duck-btctl` 足够奇怪，`dev-push.sh` 带着对从未跑过它的 clone 的兜底——它 shell 出到 `cargo run -q -p btd --example duck-btctl`。`cargo install --path duckctl` 不需要兜底。
- **其实是产品的 example。** 它是 example 的原因是让 `btleplug` 不进机器人，因为 example 的 dev-dependencies 从不到达发货 artifact。它自己的 crate 免费保住这一点，通过不成为任何 daemon 的依赖——同样的保证，直接陈述而非作为文件位置的副作用。

### 3.2 `duckctl`

它与 `robotctl` 配对，正如两者实际使用方式：`robotctl` 在机器人上，`duckctl` 对着它。它保持仓库已命名的 `duck-` 家族，且短到不用别名就能打。

`duck` 单独更好打但被占了：Cyberduck 发货一个叫那个的 CLI，在 Mac 上说得通。`microduck` 是仓库且对一小时跑二十次的命令太长。

### 3.3 什么没被重写

约二十个文件里 196 处引用，大多是散文。**带日期的记录保留旧名。** `update-over-ble.md` 和 `install-path-gap.md` 描述一个时刻——一次更新会话、四个安装路径 bug 及什么堵住了它们——把工具名重写进工具还没那个名字时发生的事的记述会让记录略假。它们的*链接*被重指向以便仍可解析；它们的散文不动。

`roadmap.md` 坐在同一目录且是例外，因为它不是某时刻的记录：它描述仓库现在的样子，细到逐 crate 布局表。点名一个不存在目录的布局表就是错的。

**无兼容垫片，无别名。** 一个用户一台机器人：仍工作的 `duck-btctl` 是要保持同步的第二个名字，也是旧名在某人 shell 历史里存活一年的理由。

### 3.4 什么让它不进板

`cargo board --bins`——在 `dev-push.sh` 和 release 工作流里——为 aarch64 构建每个默认成员。作为 example 它免费被排除；作为 crate 它会被交叉编译，意味着在发布路径上为绝不能见到蓝牙栈的板子构建蓝牙栈。

**工作区根的 `default-members`，除 `duckctl` 外的一切。** 一个列表，而非在两个 `--bins` 调用点点名二进制并手动保持两者同步——那正是本仓库不断写下的失败。`--workspace` 不受影响，所以 CI 像以前一样 lint 和测试它。

## 4. 页面变成控制台

**已落地** —— `mediad/webclient/index.html`，仍一个文件且仍无构建步骤。

允许子集很大且几乎没有从页面可达。围绕人来此要做的事重组：

| | |
|---|---|
| **header** | robot name, release, API version — from `hello` and `system.info`, sent automatically when the channel opens, not clicked |
| **video** | plus link quality from `getStats()`: bitrate, fps, loss, RTT. Today a stream that degrades is a picture that looks worse and a log that says nothing |
| **drive** | keys and an on-screen stick → `robot.move` at a fixed rate; drag on the video → `robot.look` |
| **posture** | `robot.enable`, `init`, `relax`, `stop`, `shutdown` — confirm on the last two |
| **do / sound** | the `Do` and `Sound` enums as menus |
| **telemetry** | `robot.subscribe` at 2 Hz into a live panel: mode, health |
| **console** | the raw JSON box, the log, and the two refusal buttons — collapsed, because they prove the route table rather than drive the robot |

三个约束：

- **停止按钮不能读作急停。** `route.rs` 允许 `robot.stop` 基于此通道可靠且 deadman 已在意图停止到达时停下机器人，然后明说 UI 不应暗示它是物理急停。一个标签，不是大红圈。
- **版本差异是横幅，不是锁门。** `hello` 报告不一致就说明且页面继续工作——与 `duck-btctl` 在 #102 定下的规则相同。
- **仍一个文件，仍无构建步骤。** 那个约束是客户端能运行的原因且它保留。如果它长出一个文件就变成三个——`index.html`、`app.js`、`app.css`，三个 `include_str!`，仍无构建步骤，仍无 npm。

从页面驾驶也是对 `remote-webrtc.md` 提出且没东西演练过的两个主张的第一次真正测试：deadman 在会话断开时停下机器人（§6），以及 `control` 上的排序足以让 `intents.rs` 诚实（还是 §6）。两者相信便宜错了贵。

## 5. 机器人在会话开始前命名自己

**已落地** —— `mediad/src/producer.rs`。

`webrtcsink` 接受一个 `meta` 结构，信令服务器把它交给 `list` 中的每个对端——页面已经记录它且它是空的。从 `configd` 填它：name、serial、release、`API_VERSION`。

小，且回报三次：页面能在开始会话前命名机器人、找到两个 producer 的客户端能说哪个是哪个，以及 §7 的会合服务正需要这个字段来路由。这里最便宜的项。

## 6. 它打开但不是什么

`remote-webrtc.md` §11 推迟"服务端程序的 WebSocket 表面——同样的 JSON-RPC，无媒体栈，`get_frame` 返回 JPEG"，并称一旦 §5 的路由存在只要几十行。一旦 `mediad` 里有了 `axum` 服务器，它就是已运行服务器上的一条路由，且帧已在那：`main.rs` 里的 `_frames` 是 tee 上的原始 NV12 抽头，还没东西读它。

点名以便形态可见，不在此提议。它是第二个传输，第一个应先做好。

## 7. 顺序，以及还剩什么

四个变更，每个独立且分别落地，按此顺序：

1. **服务页面，服务时填入信令 URL。** 删掉 python 指令和警告块。小，且其他一切在它上面更顺。API 版本以同样方式替换，用于 §4 的横幅——页面里的字面量会是 `API_VERSION` 的第二份副本，在它被 bumped 那天错且错在报告一致的方向。
2. **Producer `meta`。** 更小，且独立。
3. **`duckctl` 增加 `ip` 和 `open`。** 仅客户端，没碰 daemon。
4. **控制台。** 大的那个，最后做，在一个已经可达的页面上。

**未做，且刻意分开：** `dev-push.sh` 的 `resolve_board` 仍用 `duckctl wifi status` 和六行嵌入 Python 手写这个（§2.4）。它变成 `duckctl ip`，删掉 Python 和 PIN 错分支——分开因为它触碰推送路径，且因为它应在 `ip` 被手动用过几次之后落地。

## 8. 不做

- **控制台上的门。** `remote-webrtc.md` §4 负责那个决定，这里什么都不改变其条款。一个关闭页面的 `--no-web` 标志对该节标出的家庭场景值得有——一个标志，不是机制。
- **JS 框架、打包器或 `gstwebrtc-api`。** 页面手写协议因为需要 npm 的客户端是没人跑的客户端。大四倍时仍成立。
- **通过 TLS 服务。** §1.3 说何时，以及为什么不是现在。
- **教广告一个端口。** 它携带四字节 IPv4；以非默认 `--web-port` 跑的机器人是 `duckctl open` 上的 `--port`，不是线格式变更。
