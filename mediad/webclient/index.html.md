# index.html 文件解析

**文件位置**：`d:\microduck\mediad\webclient\index.html`

## 核心设计决策

机器人的控制台。单文件、无构建步骤、无依赖——`mediad` 内嵌并提供此页面，访问就是一个地址：`http://<robot>:8080/`（或 `duckctl open` 自动找机器人）。

**机器人提供它，这才使信令目标可推导。** 页面来自它要对话的机器人，所以主机是 `location.hostname`，端口由 `mediad` 提供时填入（`SERVED_PORT`）。无需输入任何东西，也没有端口或 API 版本的第二份副本。

直接说 gst-plugins-rs 的信令协议，不用其 `gstwebrtc-api` JS 库——需要 npm 的客户端没人跑。协议很小，形状从 `net/webrtc/protocol` @0.15.3 读出。

**单文件是约束而非偶然**——这是它能运行的原因。若超过一个文件则拆成 index.html/app.js/app.css 三个 `include_str!`，仍无构建、无 npm。

HTTP 非 HTTPS：HTTPS 页面上的 `ws://` 是混合内容被直接拦截，LAN 机器人无证书可提供 `wss`。代价是麦克风、部分浏览器的游戏手柄（`webrtc-console.md` §1.3）。

从 checkout 打开仍可开发：无端口替换时回退 8443，主机来自提供它的地方。`file://` 页面完全无法到达机器人（Chrome Private Network Access 阻止不透明源到私有地址）。

## 关键机制

### Token 替换
`{{SIGNALLING_PORT}}` 与 `{{API_VERSION}}` 由 `mediad` 的 `web::page` 在提供时替换。`NaN` 表示未替换（从 checkout 打开），用于区分诊断。

### 信令流程
`ws://<hostname>:<port>` 连接 `webrtcsink` 信令服务器：`welcome` → `list`（取 producer 的 `meta`：name/release/api_version）→ `startSession` → `sessionStarted` → 建 `RTCPeerConnection` → 处理 `peer` 消息（sdp offer/answer、ice）。无 ICE 服务器（LAN 场景；远程走桥）。

### 数据通道
机器人创建数据通道（`mediad` 在每个 consumer 的 webrtcbin 上 `create-data-channel`），客户端 `ondatachannel` 接收。`control` 标签。

### 控制面（JSON-RPC over datachannel）
- `call(method, params, quiet)`：分配 id、注册 promise、10 秒超时。`quiet` 抑制日志（10 Hz 的 twist、2 秒的 health 轮询是页面在工作，不是人做的事）。
- `tell(method, params)`：fire-and-forget，用于连续 intents。
- 无 id 行是通知：`robot.state`（telemetry）、`media.video`（安装旋转角）、`media.detections`（检测框）。

### 视频旋转
摄像头安装偏四分之一圈，机器人不旋转像素。页面用 CSS `transform: rotate()` 在 GPU 上转，wrapper 保持直立（`aim` 拖拽坐标与检测框都在直立空间）。`onVideoInfo` 从 `media.video` 通知取旋转角并应用 `.turn90/180/270` 类。

### 驾驶
键盘（WASD/QE/方向键）与两个触摸板（移动+转向）。`MAX_LINEAR=0.3 m/s`、`MAX_ANGULAR=1.5 rad/s`（与 `padd` 默认一致）。连续 intent 以 10 Hz 重发（`safety.deadman_ms=500`，留余量），停止时发零。

### 检测框
`media.detections` 通知的框在直立帧像素内，用 SVG `viewBox` 绘制，浏览器缩放。2.5 秒无更新则清除（防最后一只鸭子永远留在画面上）。标签只写 "duck" 不写分数（量化模型输出张量与框坐标共享尺度，真实检测都约 1.3）。

### 遥测
`robot.subscribe` 2 Hz（服务端降采样）+ `robot.health`/`robot.mode` 每 2 秒轮询。`robot.health` 的 battery/temps 缺席渲染为 "not read" 而非空电池（避免误导）。

### 链路质量
每秒 `pc.getStats()` 算码率、丢包率、fps、rtt。

## 关键摘要

- 单文件无依赖，由机器人提供使信令目标可推导、Private Network Access 不拦截。
- 直接说 gst-plugins-rs 信令协议，不用 npm。
- 控制面是 JSON-RPC over datachannel，通知（无 id）承载 telemetry/video/detections。
- 旋转在浏览器 GPU 上做（CSS 变换），不在机器人上做（省 145% 核 + 22 fps）。
- 连续 intent 10 Hz 重发，deadman 保障掉线即停。
