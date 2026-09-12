# WebRTC：会话、信令和控制通道

手机、浏览器或服务端程序如何通过 WebRTC 驱动和观察机器人。[`architecture.md`](architecture.md) §5 陈述需求；本文负责机制。

范围限于**本地信令**：以下一切都在机器人上运行，在 LAN 上无需任何后端即可工作。从 LAN 外联系机器人是同样的设计，前面加一个代理（§7）——刻意不是最先构建的，因为本地情况是其他一切情况的定义基准。

## 0. 状态：在硬件上工作

一块 Radxa Zero 3W 通过硬件编码器把 `videotestsrc` 推流到 LAN 上的浏览器，浏览器同时得到一个 `control` 数据通道。2026-08-25 端到端验证：

| | |
|---|---|
| signalling | `mediad` runs the server in-process; producer registers, consumer lists and starts a session |
| video | `mpph264enc` → `webrtcsink` → browser, negotiated as `profile-level-id=42e01f` — constrained baseline, which is §2's whole point |
| bundling | `a=group:BUNDLE video0 application1`, `a=sctp-port:5000` — one transport for media and data |
| datachannel | `control` arrives at the peer |

两件从未运行过且第一次尝试就出问题的事，记录下来因为它们是这个设计容易招来的 bug 形态而非偶发：

- **从 GStreamer 信号线程 `tokio::spawn` 会中止进程。** 那个线程不在运行时里，而 panic 跨越 C 闭包不会展开。journal 说 `thread caused non-unwinding panic`，回溯经过 `g_closure_invoke`，对原因只字未提。那些处理器里什么都不能 panic——见 `mediad::pipeline` 的头部。
- **客户端无法从 `file://` 页面到达信令端口。** 对私有 IP 的不透明源正是 Chrome 的 Private Network Access 拦截的；通过 `http://localhost` 服务页面即可修复。
- **协商失败的编解码器被丢弃并警告，而非报错。** 从预编码 H.264 转到原始视频（§2）把编码器放进了 `webrtcsink` 内部，它的发现阶段要求 `profile=constrained-baseline`——而 `mpph264enc` 的 pad 模板没列它，所以 H.264 从 offer 中消失，VP8 被协商，会话死掉。每个症状都指向别处：可见错误来自上游四个元素的 `videorate`，抱怨 NV12。这需要插件 release `v3` 或更高，这也是为什么 `mediad` 把 GStreamer 的调试日志*和*流水线总线都桥接进 journal——没有那个，上面这些根本不可见。

### 摄像头，以及 35% 丢帧的两个独立原因

头部摄像头通过硬件编码器以 **29.3 fps** 推流。达到那里花了两个无关的修复，花了些时间的原因是单独任何一个都不改变数字——这让每个看起来都无效。

**捕获池深度。** rkisp 不实现 `V4L2_CID_MIN_BUFFERS_FOR_CAPTURE`，所以 `gst_v4l2_object_decide_allocation` 从零计算 `own_min` 并落在两个缓冲区上。三是个悬崖而非斜坡：主路径上 `v4l2-ctl --stream-mmap=N` 在两个时 19.7 fps，三个或更多时 29.2。提高它需要 ALLOCATION 查询里有 `GstVideoMeta`（否则 `can_share_own_pool` 为 false 且读查询 `min` 的分支从不被取）和一个 `min` 非零的首个池（任何下游元素提议池都会设 `update` 并放弃 `+2` 奖励——`GstVideoEncoder::propose_allocation` 正是那样提议）。

**像素格式。** rkisp 提供非连续的两平面 `NM12`  alongside 单平面格式。两者都映射到 GStreamer `NV12`，`v4l2src` 偏好多平面的那个，而它在这里任何池深度下都无法全速驱动：

| caps | 2 buffers | 4+ buffers |
|---|---|---|
| `NV12` (selects `NM12`) | 19.5 fps | 19.6 fps |
| `UYVY` (single plane) | 19.7 fps | **29.3 fps** |

`mpph264enc` 在其 sink pad 上列出 `UYVY` 并在 RGA 上转换，所以 4:2:2 到 4:2:0 不耗 CPU。

**被排除的，通过测量而非论证**——其中每个都是合理的嫌疑：`v4l2-ctl` 在同一节点上用任一格式都能到 29.2 fps；传感器 subdev 报告 1/30 间隔；驱动不实现 `S_PARM`；`mpph264enc` 全速编码 720p 达 130 fps；且 `v4l2src` 无约束时偏好的 DMABuf caps 不是它无约束时快的原因。

**方法论教训，比上面任何一个代价都大。** 四个不同速率各自被当作捕获速率，而没有一个是：`rkvenc` 中断计数的是*编码器*消费的，在 `webrtcsink` 的队列和其转换器 bin 里 `videorate drop-only` 之后；`v4l2src` 的 `lost frames detected` 计数驱动序列号间隙，在源只是慢时静默；tee 原始分支上的计数器在一个刻意漏的单缓冲区队列之后；而 `/dev/video1` 是 ISP 的自路径，不是 daemon 用的主路径。`mediad` 现在在 tee 之前计量 pad，那是它和驱动之间没有任何有损东西的唯一地方。

这暴露的两件未修复的事：`rtpgccbwe` 耗约 40% 的一个核，GStreamer 自己的 `INFO` 级日志从不进 journal 尽管桥接把它映射到 `tracing::info!`，这就是为什么其中几个问题是用慢方式回答的。

仍未测试的：通过桥接的任何东西。

### 选择质量，以及为什么它是一个设置而非四个

流是什么——摄像头还是测试图案、帧尺寸、速率、比特率——是 `/etc/robot/robotd.toml` 里的 `[media]`，`robotd` 已经读的逐板配置文件。`sudo robotctl configure` 编辑它并提供它需要的 `systemctl restart mediad`；`mediad` 在启动时读一次，像这里每个其他 daemon 读它的配置一样。

**一个 `quality` 键点名一档——`1080p30`、`720p30`、`720p15`、`360p30`——而非宽度、高度和 fps。** 这三者不独立变化：捕获路径产生不了的组合是起不来的流水线，而那会连带*控制*通道一起损失，因为数据通道与视频轨道绑定（§2）。每档都是 16:9，传感器自身的宽高比，所以"更小"从不悄悄意味着"裁剪"。`bitrate` 是唯一仍可单独设的数字，不设时跟随档位——720p30 时 2 Mb/s，上面所有测量都在这个速率下取的。

在该节存在之前，这些数字是 `mediad.service` 里的 `ExecStart` 标志。release 安装器重写那个 unit 文件，所以改一个意味着 systemd drop-in——一种为*接线*不同的板子准备的机制，而非为问为什么画面模糊的人。

### 拥塞控制也是 CPU 设置

`[media] congestion_control` —— `gcc`、`homegrown` 或 `disabled` —— 是 `webrtcsink` 自己的 `congestion-control` 属性，设置而非继承。`gcc` 是元素的默认，所以点名它不改变什么；它换来的是上游哪天改那个默认时不是每台机器人的发送速率跟着变的那天。

**它是进程中最大的单个 CPU 消耗者。** 每线程，板上一个对端连接时：

| thread | %CPU (of one core) |
|---|---|
| `rtpgccbwe1:src` | 7.6 |
| `queue1:src` | 6.0 |
| `mpph264enc2:src` | 5.0 |
| … | |
| `v4l2src0:src` | **0.3** |
| `queue3:src` (the raw tee branch) | **0.3** |

总共约一个核的 25%，其形态才是重点：捕获和原始分支帧拷贝——仅有的两个随*像素*缩放的东西——加起来 0.6%。其他一切是包处理，随比特率缩放。这就是为什么选更小的档不降低 CPU：在一条从不饱和的链路上，估计器把 360p 流 ramp 到约 720p 在用的比特率，把它花在每像素质量上。

`disabled` 删除 `rtpgccbwe` 线程。它损失自适应性，那正是把原始视频而非预编码 H.264 交给 `webrtcsink` 的全部理由（§1）——在降级链路上它保住画面而非卡顿。它也让 `bitrate` 名副其实：没有东西移动它，所以它是速率而非起点。

**720p30 是唯一测过的档。** 传感器被钉在 1920x1080 模式跑 30，ISP 从它缩小，所以 1080p30 要求完全不缩放；没人测过的是捕获路径和编码器在 2.25 倍上表像素下能否保持 30 fps。保持不了的档跑得更慢——不是启动失败。

## 1. 这不是什么

不用 `webrtcbin`。`mediad` 用 `gst-plugins-rs` 的 **`webrtcsink`**，区别正是本文简短的全部原因：`webrtcsink` 带来信令协议、会话模型和逐消费者编码器管理，所以剩下要设计的是*控制*表面而非媒体管道。

`webrtcbin` 意味着要写全部三个。它在 Debian 里而 `webrtcsink` 不在（[`media-bringup.md`](../project/media-bringup.md) 覆盖该插件如何构建和发货），这是它的唯一论据——且被协议免费带来压过，因为那个协议正是远程桥接代理的（§7）。

## 2. 一个会话，四个流

```
peer (browser / phone / server-side program)
   ├── video track      camera, hardware H.264 (mpph264enc, constrained-baseline)
   ├── audio track      mic; two-way for telepresence
   ├── datachannel "control"   reliable, ordered      → the robot API (§5)
   └── datachannel "teleop"    unreliable, unordered  → input and high-rate telemetry
```

两个数据通道而非一个，原因见 `architecture.md` §5.2：重传一个 80 ms 前的摇杆命令比没用更糟，所以 teleop 走 `maxRetransmits: 0` 并永远取最新。

**第一版只开 `control`。** Teleop 不是近期优先，且把它排除不只是推迟——§6 讲它移除了什么。

**`webrtcsink` 在其 sink pad 上接受预编码 H.264**，所以编码器从不进入协商。在硬件上验证；四个是决策而非默认的编码器属性在 [`media-bringup.md`](../project/media-bringup.md)。

**流水线在编码器之前 tee 原始 NV12**，那个位置是刻意的：

```text
                              ┌─ queue ─ mpph264enc ─ h264parse ─ webrtcsink
capture ─ NV12 ─ capsfilter ─ tee
                              └─ queue(leaky, 1) ─ appsink ─ latest frame
```

§5.3 要服务端程序按需取帧——"它想要每秒一两帧加一个状态 blob"，不是 30 fps 的 H.264 轨道去解码——而 `architecture.md` §2 要感知挨着传感器，派生特征而非把像素运到 `robotd`。两者都需要*像素*，从编码分支取意味着解码刚编码的东西。

全程 NV12，因为那是 rkisp 路径输出的、`mpph264enc` 接受的，所以没有地方转换。每个分支有自己的 `queue`——没有它们的 `tee` 从一个线程跑两者，所以慢读者会卡住视频轨道——原始那个是漏的且一缓冲区深，那是 `architecture.md` §2 要求的 last-value-wins、非阻塞快照。停滞的读者损失帧，从不损失编码器。

分支从一开始就存在而非在有东西读它时加：向活动流水线插入 tee 比有一个一直在那的难得多。

## 3. `mediad` 自己跑信令服务器

`webrtcsink` 有 `run-signalling-server`，带 `signalling-server-host` 和 `signalling-server-port`（gst-plugins-rs 0.15.3）。所以服务器在 **`mediad` 自己的进程里**运行，没有第二个二进制要构建、发货或监督——这很重要，因为我们发货的插件是 `.so`，而 `gst-webrtc-signalling-server` 是同一上游 crate 的另一个 Rust 二进制。不发货它是真正的简化，不是捷径。

`webrtcsink` 自己的 signaller 默认 `ws://127.0.0.1:8443` 并连接到它刚启动的服务器。LAN 客户端直接连到同一个服务器——`mediad/webclient/index.html` 是一个，单文件无构建步骤，它手写这个协议而非通过 `gst-plugins-rs` 的 JS 库，所以尝试它不需要装任何东西。[`webrtc-console.md`](webrtc-console.md) 负责那个页面变成什么——机器人服务它、发现机器人，以及它应到达的控制表面。

**绑定地址是决策，不是默认。** 仅环回意味着 LAN 对端根本联系不到它，每个会话都经过桥接，这违背了本地模式的意义。所以它绑定所有接口，§4 是那对谁可以驱动意味着什么。

## 4. 授权：机器人上没有，以及为什么这在本地和远程都成立

**第一版没有门。** 能到达信令服务器的对端可以启动会话、驱动机器人、看它的摄像头。这是决策，不是疏忽。

在这个阶段可用性胜过加固，且这里的权衡甚至不接近。机器人的配对 PIN 是共享的 `000000`——每台机器人都一样的 PIN 认证不了任何人——所以通过 WebRTC 要求它会给每次首次连接加一步却换不来任何安全。别扭的首次连接是真实代价；这个特定的门是有代价无收益。

它的代价，明说以便没人需要自己发现：同一网络上的任何人都有机器人及其摄像头。在实验台和办公室没问题。**在家里不行**，这是发货到家里之前要重新审视的事。

### 授权实际住在哪：桥接，且它已经在那

远程路径也不需要机器人上的门，因为它在*到达机器人之前*就被认证了——两边都：

- **客户端**用 OAuth 向会合服务认证，服务只向它显示其账户拥有的机器人。到达桥接中路由到给定机器人的那部分*就是*证明。
- **机器人**向外认证：它的中继持有一个账户令牌并用它连接到服务（§7）。所以机器人也证明它属于该账户。

服务因此匹配两个已认证的双方，通过它到达的会话被授权了两次。在其上加 `system.authenticate` 是对已回答问题的第二个答案——且更差，因为共享的 `000000` PIN 比账户令牌证明得少。

**这意味着信任移动了而非消失了**，值得点名它去了哪：机器人没有独立检查，所以机器人和账户之间的绑定现在是必须正确的东西，且它住在服务而非这里。那是它可接受的位置——它是唯一能知道答案的组件——但它是依赖，而非没有依赖。

这一切都不覆盖的一种安排是机器人的信令端口被直接暴露到互联网，通过端口转发而非通过桥接。那样没有桥接认证任何东西，§4 的 LAN 推理也不适用，因为能联系到它的人群不再是楼里的人。那是部署错误而非设计决策，值得大声说出来恰恰因为机器人里没有东西会注意到。

### 如果需要，钩子在哪

`system.authenticate`——BLE 已用的方法，在 `API_VERSION` v4 加入。控制通道会服务那一个方法并在它通过前按名拒绝其余，PIN 从 `configd` 通过 unix socket 读取而非通过正在被认证的通道。

它在这里被点名以便答案存在，不是因为计划中。它的理由很窄：它对桥接路径什么都不加，那里已经认证得更好；在 LAN 上它每次连接耗一步却只证明对端读了印在每台机器人上的数字。如果需要，它很便宜——§5 的路由表已经需要一个传输可达哪些方法的概念，而"认证前哪些方法"是同一个表用更小的子集而非新机制。

## 5. 控制通道是到现有 API 的管道

`control` 上的帧是 **JSON-RPC 2.0，每行一个对象**，那正是 [`duck-ipc-proto`](../../duck-ipc-proto/src/lib.rs) 已定义的、`robotctl` 和 `btd` 已说的。不发明新东西：`mediad` 把调用路由到拥有它的服务的 unix socket 并把回复泵回去。

`btd` 是工作先例且应成为共享的：

| `btd` today | what `mediad` needs |
|---|---|
| `route.rs` — which calls may travel, which socket answers, which lane carries them | the same table, with a *different permitted subset* |
| `session.rs` — the `system.authenticate` gate | the same gate |
| `upstream.rs` — dial the sockets, timeout everything | the same |
| `framing.rs` — BLE MTU chunking | **not needed**; SCTP frames itself |

所以四个文件中三个是传输无关的，一个不是。**把路由表提升到两个传输共用的东西**，参数化为一个传输可达哪个子集。

值得保留的性质不是代码，是穷举匹配。`route.rs` 明说：给 `proto::Call` 加一个变体让该文件编译失败，所以新方法不能因为某人忘了这个文件存在而到达无线电。`_ => None` 通配会静默拒绝新方法，第一个症状会是 app 看不到没人记得路由的功能。那个保证必须**逐传输**成立，否则 WebRTC 就成了它的洞。

### WebRTC 可达什么，以及为什么不是 BLE 的子集

BLE 的子集窄是因为无线电慢且几米内任何人都能跟它说话。这里两者都不适用，所以 WebRTC 得到更多——但"更多"不是"一切"，两类仍排除：

- **`system.pairingPin` 和 `system.setPairingPin`。** 不是因为它们会危及*这个*传输——§4 反正留着它开着——而是因为它们授权**另一个**传输。能改写配对 PIN 的 LAN 对端能把手机锁在 BLE 外，那是恢复路径。让 PIN 离开每个网络传输与让它不可路由到 BLE 本身是同一规则。
- **`update.*` 变更。** 目前仅限，且原因与 PIN 不同：应用更新重启 `mediad` 并断开会话。稍后想要；§8 是需要什么。

### 回复不被关联，刻意

`btd` 转发 socket 发出的任何东西而不解析它，且有测试钉住这一点：订阅是开放连接上的通知流，每个都必须到达客户端。把回复关联到请求会正好破坏那个。`mediad` 继承同一规则，这也意味着**加方法时 `mediad` 里无逐方法工作**——管道保持哑，`duck-ipc-proto` 仍是定义方法的唯一地方。

lane 概念也转移过来，且很容易假设它不会。每个 daemon 每条连接一次只服务一个请求，所以 `update.subscribe` 后跟同一连接上的任何东西都会挂——正是 `app-path-design.md` §7 记录的 bug。一个数据通道是一个有序流，有同样的危险，`btd` 的答案原样可用：按方法路由到 per-lane socket、把每个 socket 泵回去、永不关联。

## 6. 为什么先做 `control`-only，以及 `teleop` 落地时的代价

`intents.rs` 把每个意图存在 `ArcSwap` 里并采用 last-writer-wins。这在今天是正确的，因为每个写者都通过 unix socket 到达它，那里后到的消息不可能先于早到的到达。

**可靠、有序的数据通道保持这成立。** 该模式下的 SCTP 按定义有序交付，所以通过 `control` 到达的意图保留 `intents.rs` 已经依赖的性质。所以从一个通道开始不是积攒工作的妥协——它意味着第一版根本没有排序问题要解。

### 相反它的代价，以便没人吃惊

队头阻塞。在可靠通道上丢包会卡住它后面的一切，包括控制 RPC，所以差链路表现为*一切*暂停而非摇杆过时。通过 `control` 驾驶在中等速率下没问题，随速率和丢包变差——这正是 `architecture.md` §5.2 规定第二个通道的原因，也是"机器人在差链路上感觉卡顿"的答案是 teleop 而非调优。

### 当 teleop 落地时，它需要序列号

**`maxRetransmits: 0` 的 SCTP 会重排。** 80 ms 前的转向可能在更新的之后到达并赢得 last-writer-wins，机器人然后基于过时命令驾驶，没有任何地方报告问题。这不是罕见竞态：它是通道的正常行为，刻意选择的。

所以 teleop 帧携带**每流单调序列号**，写者丢弃任何不比它上次应用的新的东西。这是*传输*的性质，所以它属于 `mediad` 而非 `robotd`——`robotd` 应继续接收它能信任顺序的意图，这正是让 `intents.rs` 保持简单的原因。

值得在通道存在之前而非之后写下来：失败是静默的，看起来像调优差而非 bug，且如果设计进去修复是微不足道的，如果先得诊断过时转向就很别扭。

Deadman 两种情况下都不需要什么：`safety.gate(command, twist_age)` 已经基于年龄，所以分区会让机器人停下而无需新代码。

## 7. 联系不在你 LAN 上的机器人

**远程路径是到本地信令服务器的桥接，不是第二个设计。** 一个中继进程向外连接到会合服务并把同一协议代理到 `ws://127.0.0.1:8443`。机器人的信令服务器、会话模型、授权和控制通道不变；DTLS-SRTP 让媒体即使通过中继也端到端加密，值得向客户明说。

两个性质随之而来，且都是这个形态的原因：

- **本地模式从不依赖桥接。** 如果会合服务宕机，LAN 客户端仍连接。`architecture.md` 的不变量 1——本地恢复保持独立——延伸到媒体。
- **桥接什么都不解析。** 它代理 gst 信令协议，那与 LAN 客户端说的是同一协议。这是用 `webrtcsink` 而非 `webrtcbin` 的具体回报：协议已存在，所以桥接是中继而非翻译器。

- **桥接认证，所以机器人不必。** 中继向外连接持有账户令牌，服务只向客户端显示其账户拥有的机器人——所以桥接会话在到达前两边都被授权。§4 覆盖信任落在哪。

  一个有用的后果：因为中继是连接到环回的机器人侧进程，机器人*能*通过源地址区分桥接对端和 LAN 对端，即使它目前不对区别采取行动。如果那不再成立，没有东西被排除。

`reachy_mini` 正是对一个 Hugging Face Space 运行这种安排，机器人注册为 `producer`，Space 通过心跳保持 TTL 租约刷新。我们是否采用那个服务，以及机器人如何绑定到账户，超出这里范围且在本地模式工作前不进入。

### 信令协议，给写桥接的人

来自 `gst-plugins-rs` 0.15.3，`net/webrtc/protocol`——线上是带 `type` 标签的 JSON，camelCase：

| peer → server | server → peer |
|---|---|
| `setPeerStatus` (`roles`, `meta`, `peerId`) | `welcome` (`peerId`) |
| `startSession` (`peerId`, optional `offer`) | `sessionStarted` (`peerId`, `sessionId`) |
| `endSession` (`sessionId`) | `startSession`, `endSession` |
| `peer` (SDP `offer`/`answer`, or `ice`) | `peer`, `error` (`details`) |
| `list`, `listConsumers` | `list` (`producers`), `listConsumers` (`consumers`) |

角色是 `producer`、`listener`、`consumer`。机器人是 `producer`；`meta` 是自由形式 JSON，是机器人身份去的地方。

## 8. 通过 WebRTC 更新机器人：尚未，以及需要什么

不在第一个允许子集中，且**那是推迟而非原则**——手机通过 WebRTC 更新机器人是稍后想要的，所以本节是关于首先必须什么为真而非为什么不能。

今天让它别扭的是：应用更新重启 `mediad`，断开客户端正看进度的会话。`update-over-ble.md` 记录了"启动更新并看它"静默失败已经付出过一次代价，而更新中途消失的会话是同形态的问题。所以第一个子集把 `update.*` 变更排除，BLE 仍是能在重启中存活的传输。只读 `update.*` 调用从一开始就在——在远程会话上看版本和历史有用且无代价。

要让变更进来，两件事得变，且都小而具体：

- **客户端必须能在重启中存活。** 协议已经支持：进度作为 JSON-RPC *通知*推送，`duck-ipc-proto` 精确文档化它以便更新中途重连的客户端能重新订阅并继续接收。所以工作是一个重连并重新订阅的客户端，而非线格式变更。
- **`RobotRemoteSessionActive` 必须更具体。** `updater/src/preflight.rs::check_no_remote_session` 在远程会话存在时拒绝更新，当会话是*旁观者*时这是对的——有人在远程临场通话中，不应让机器人在他们脚下重启。当会话是*请求者*时这是错的。还没有东西把那个标志设为真，所以这个区分可以设计进去而非后装：检查需要知道这次更新是否通过它即将断开的会话请求。

值得现在写下来恰恰因为还没有东西设标志。`mediad` 诚实报告的那一刻，通过 WebRTC 请求的更新会拒绝自己，那看起来像更新路径的 bug 而非这里缺的区分。

## 9. 权限：本特性打破的前提，已记录未行动

`intents.rs` 说它的槽位是"实践中单写者，所以 last-writer-wins 名副其实"。一个手柄时这成立。**一个 pad 和一个远程对端同时驾驶的那一刻它就不成立了**，失败不是竞赛——是两个 50 Hz 写者交错进一个槽位，产生一个谁都不听的机器人。

**刻意不在此解决。** 记录下来以便令人困惑的机器人有一个等待的解释，且因为最终解决它的标志在有两个传输之前设计比之后便宜。`architecture.md` §6 负责需求——定义优先级和交接、本地物理能抢占远程——路线图把它放在 M6。

到时候，廉价答案是**单写者令牌**：一个对端持有写意图的权利，其他是观察者，交接是显式的。这远少于 §6 的完整仲裁，且它移除交错，那是产生无意义而非仅仅错误赢家的部分。优先级排序——物理不经询问抢占远程——可以之后来，在同一令牌之上。

本节在那之前的*用途*：知道两个同时驱动者是已知缺口而非谜团，且第一个症状是机器人忽略两个输入而非服从错误的那个。

## 10. 构建 `mediad`

`gstreamer-rs` crate 是 pkg-config crate，所以交叉编译它们需要开发者机器上有*目标*的头文件、`.pc` 文件和共享库。`cargo board` 用 `cargo-zigbuild` 从 macOS 交叉构建，而曾经供应它唯一 C 依赖的 multiarch 脚本说它"是那个一个例外的代价，且在加另一个之前值得一读"。这是第二个，且大得多——它彻底替换了那个脚本，因为 Ubuntu multiarch 能提供 libudev 却不能诚实地提供 GStreamer：它会给 Ubuntu 的而机器人跑 Debian trixie 的。

**`scripts/cross-sysroot.sh` 把机器人自己的 Debian 包解包到 sysroot**——已证明：整个工作区针对它交叉构建，`gstreamer`、`gstreamer-app` 和 `gstreamer-webrtc` 都解析为 1.26.2，与板子跑的版本相同。

碰它之前值得知道的三件事：

- **它服务整个工作区，不仅 `mediad`。** `PKG_CONFIG_LIBDIR` *替换* pkg-config 的搜索路径而非添加，所以只带 GStreamer 的 sysroot 会破坏 `padd`——它的 `gilrs` 需要 libudev——且破坏发生在 `libudev-sys` 内部，离任何关于媒体的东西都很远。替换仍是对的：`PKG_CONFIG_PATH` 是加到*宿主*的，这就是 pkg-config 会用 macOS 库回答并产生无法在机器人上运行的二进制的原因。
- **包列表是显式的，非解析的。** 从明显的根走 Debian `Depends` 会拉 543 个包，因为 `libgstreamer-plugins-bad1.0-dev` 声明了每个可选后端的 dev 包且闭包到达 Qt、Vulkan 和 OpenEXR。十九个包满足实际需要的东西。
- **单独 `-dev` 包对任何实际链接的东西都不够。** 它把 `libfoo.so` 作为符号链接发到运行时包里的 `libfoo.so.N`，所以 `-lfoo` 两者都要。只出现在 `Requires.private` 的库只需要 `-dev`。

替代方案是在 arm64 runner 上构建 `mediad`，像 [`media-bringup.md`](../project/media-bringup.md) 里的插件。被拒绝因为它把 daemon 构建一分为二且让没人能在笔记本上构建 `mediad`——对将需要最多针对真实硬件迭代的 crate 来说这是错误的权衡。

## 11. 推迟，附理由

- **服务端程序的 WebSocket 表面**（`architecture.md` §5.3）。同样的 JSON-RPC，无媒体栈，`get_frame` 返回 JPEG。一旦 §5 的路由存在它只要几十行，且它是让"LLM 驾驶机器人"变简单的东西——但它是第二个传输，第一个应先工作。
- **`teleop` 数据通道。** 不是近期优先；§6 覆盖推迟它移除了什么、期间代价是什么，以及它将需要的序列号。
- **多对端视频。** 一次一个媒体会话，加仅控制客户端。Simulcast 和一次编码多路发送是个真项目。
- **同意和推流指示器。** `architecture.md` §7 要显式逐会话同意和可见指示器，且说得对它们现在便宜以后贵。它们需要存在的硬件——软件可控的 LED——尚未确立。
- **TURN。** 仅 LAN 不需要。桥接需要，且耗真实带宽；那个决定属于会合服务，不在这里。
