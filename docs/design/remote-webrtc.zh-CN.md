# 远程 WebRTC

WebRTC 会话、信令和控制通道如何工作。

本页面拥有会话结构、信令服务器、控制通道和遥操作通道。[`webrtc-console.md`](webrtc-console.md) 拥有客户端页面；[`architecture.md`](architecture.md) 拥有 `mediad` 在整体中的位置。

## 为什么是 WebRTC

机器人需要将视频和音频流式传输到远程客户端，并接收控制命令。选项：

| 方案 | 延迟 | 复杂度 | 穿透 NAT |
|---|---|---|---|
| RTSP | 低 | 中 | 差 |
| HLS | 高 | 低 | 好 |
| WebRTC | 低 | 高 | 好（ICE） |

WebRTC 是唯一同时提供亚秒级延迟、浏览器原生支持和 NAT 穿透的方案。对于一个需要从手机或笔记本电脑实时驾驶的机器人来说，这是正确的选择。

## 会话结构

一个 WebRTC 会话携带四个流：

| 流 | 方向 | 用途 |
|---|---|---|
| video | 机器人 → 客户端 | 摄像头画面 |
| audio | 机器人 → 客户端 | 麦克风音频 |
| control | 双向 | 现有 API 的管道 |
| teleop | 客户端 → 机器人 | 遥操作命令（驾驶） |

video 和 audio 是标准的 WebRTC 媒体流。control 和 teleop 是数据通道（`RTCDataChannel`）。

### 为什么是两个数据通道

control 和 teleop 分开是因为它们有不同的语义：

- **control** 是可靠的、有序的。它是现有 API 的管道——与 `robotctl` 在串行端口上执行的操作相同。命令需要按顺序到达，并且不能丢失。
- **teleop** 是不可靠的、无序的。它是驾驶命令——虚拟摇杆位置、速度目标。旧的遥操作命令是无用的；如果一个命令丢失，下一个命令会覆盖它。使用 `maxRetransmits=0` 意味着 SCTP 不会重传，这减少了延迟但允许重排。

将它们放在同一个通道上会迫使 teleop 变得可靠（增加延迟）或 control 变得不可靠（丢失命令）。分开是正确的。

### teleop 需要序列号

因为 `maxRetransmits=0` 允许重排，接收者必须能够检测并丢弃乱序的遥操作命令。每个 teleop 消息携带一个单调递增的序列号；接收者跟踪最后看到的序列号并丢弃任何更低的消息。

这不是可选的。没有序列号，一个延迟的"左转"命令可能在一个较新的"直行"命令之后到达，导致机器人在用户已经释放摇杆后突然转向。

## `webrtcsink` 而非 `webrtcbin`

GStreamer 有两个 WebRTC 元素：

| 元素 | 用途 | 复杂度 |
|---|---|---|
| `webrtcbin` | 通用 WebRTC 端点 | 高（手动管理 SDP、ICE、数据通道） |
| `webrtcsink` | 单向流媒体接收器 | 低（自动管理 SDP、ICE） |

`webrtcsink` 是正确的选择，因为机器人只发送媒体（video/audio 出站）。它不接收媒体，因此不需要 `webrtcbin` 的全部灵活性。`webrtcsink` 处理 SDP 协商、ICE 候选和 DTLS 握手，让 `mediad` 专注于信令和数据通道。

代价是 `webrtcsink` 不直接支持数据通道。`mediad` 在 `webrtcsink` 旁边手动创建数据通道，使用相同的 `WebRTCBin` 底层。这有效，但需要小心确保数据通道与媒体流共享同一个传输。

## 信令服务器

`mediad` 运行一个信令服务器，使用 axum（HTTP）+ tokio-tungstenite（WebSocket）。

```
客户端                    机器人
  |                        |
  |  GET /                 |  → 页面（HTML）
  |                        |
  |  GET /ws               |  → 信令 WebSocket
  |                        |
  |  SDP offer             |
  | ─────────────────────→ |
  |                        |  → webrtcsink.set_remote_description()
  |                        |  → webrtcsink.create_answer()
  |  SDP answer            |
  | ←───────────────────── |
  |                        |
  |  ICE candidates        |
  | ←────────────────────→ |
  |                        |
  |  (DTLS + SRTP 建立)    |
  |                        |
  |  video/audio 流        |
  | ←───────────────────── |
  |  control 数据通道       |
  | ←────────────────────→ |
  |  teleop 数据通道        |
  | ─────────────────────→ |
```

### 两个端口

| 端口 | 用途 |
|---|---|
| 8080 | 页面 + 静态资源（HTTP） |
| 8443 | 信令 WebSocket（WS） |

分开是因为页面可以从任何地方加载，而信令必须到达机器人。将它们放在同一个端口上会迫使页面和信令共享一个来源。

### 授权在机器人端无门控

信令服务器不需要认证。这是故意的：

- 机器人在家庭/实验室网络上，而不是在公共互联网上
- 桥接端（如果有）已经认证了用户
- 添加令牌会使初始设置复杂化——用户需要在连接之前获取令牌

如果机器人需要在不可信网络上可访问，信令将需要令牌认证和 TLS。这些不在 v1 中。

## 控制通道是现有 API 的管道

control 数据通道不实现新的 API。它是 `robotd` 现有串行端口 API 的管道——与 `robotctl` 使用的相同的 JSON-RPC 协议。

```
客户端 → control 通道 → mediad → unix socket → robotd
```

`mediad` 不解析或验证命令。它只是将它们从数据通道转发到 `robotd` 的 unix socket，并将回复转发回来。这意味着：

- 没有新的权限模型——control 通道具有与 `robotctl` 相同的权限
- 没有新的命令——如果 `robotctl` 能做到，control 就能做到
- 没有新的错误处理——错误直接从 `robotd` 传递

这是故意的。`mediad` 是一个管道，而不是一个 API 层。添加 API 逻辑会创建第二个必须与 `robotd` 保持同步的真相来源。

## 摄像头 35% 帧丢失

在开发期间，摄像头流显示约 35% 的帧丢失。这被证明有两个独立的原因，都需要修复：

### 原因 1：捕获池深度

GStreamer 的 `v4l2src` 默认分配一个 2 帧的捕获池。在 USB 摄像头上，这意味着如果用户空间在一帧被捕获时正忙，下一个帧就会被丢弃（驱动程序没有地方放它）。

修复：将捕获池深度增加到 4。这给了用户空间更多的喘息空间，而不会显着增加延迟。

```
v4l2src device=/dev/video0 ! video/x-raw,width=640,height=480,framerate=30/1 ! ...
```

加上 `io-mode=dmabuf` 和 `buffer-pool-size=4`。

### 原因 2：像素格式 NM12 vs UYVY

摄像头支持两种像素格式：

| 格式 | 布局 | 带宽 |
|---|---|---|
| UYVY | 4:2:2 交错 | 高（每像素 2 字节） |
| NM12 | 4:2:0 平面 | 低（每像素 1.5 字节） |

`v4l2src` 默认选择 UYVY，因为它是"更简单"的格式（交错而非平面）。但在 USB 2.0 总线上，UYVY 的带宽要求意味着 30fps 时大约 35% 的帧被 USB 控制器丢弃。

修复：强制使用 NM12。GStreamer 的 `videoconvert` 可以在下游将 NM12 转换为任何需要的格式，而 USB 带宽下降了 25%。

```
v4l2src device=/dev/video0 ! video/x-raw,format=NM12,width=640,height=480,framerate=30/1 ! videoconvert ! ...
```

两个修复都是必需的。只修复一个会留下约 15-20% 的帧丢失；两个都修复会将其降低到 <1%。

## 什么未被测试

- 在真实网络条件下的 ICE 连接时间——目前只在 LAN 上测试过
- NAT 穿透——没有测试过对称 NAT 或企业防火墙
- 多个同时会话——`mediad` 目前一次只支持一个 WebRTC 会话
- 控制通道上的大消息——JSON-RPC 响应可能超过单个数据通道消息的 64KB 限制
- teleop 通道上的延迟——目标是亚 100ms，但尚未在真实网络上测量
- 音频回声消除——机器人的麦克风可能拾取机器人的扬声器音频，导致回声
#（注：内容由AI生成）
