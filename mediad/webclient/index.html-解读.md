# index.html（duck console Web 控制台）解读与架构梳理

> 分析对象：`index.html`（973 行），microduck 机器人的远程控制台——mediad 嵌入并在 `http://<robot>:8080/` 服务的单文件 Web 应用。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 duck console——机器人的远程控制 Web 界面，由 mediad 通过 axum 服务（Cargo.toml 中提到的"the console's one route"）。它是**零依赖单文件**：无 npm、无构建步骤、无框架，纯 HTML+CSS+JS。页面直接讲 gst-plugins-rs 的 WebRTC 信令协议（不依赖其 gstwebrtc-api JS 库），通过 WebSocket 信令 + RTCPeerConnection 媒体流 + DataChannel JSON-RPC 控制。

**核心功能**：
- **视频**：WebRTC 收流，浏览器 GPU 旋转画面（摄像头装偏 90°，pipeline 旋转会烧板）。
- **检测框叠加**：SVG overlay，2Hz 鸭子检测框，2.5 秒无新帧自动清除。
- **拖拽注视**：在画面上拖拽 → robot.look IK → 机器人转头看。
- **驾驶**：虚拟摇杆 + 键盘 WASDQE，10Hz 重发意图，deadman 500ms。
- **姿势控制**：enable/init/relax/stop/shutdown。
- **技能/声音**：坐下/起立/捡东西/踢腿/滚动 + 7 种叫声。
- **遥测**：mode/policy/loop/safety/battery/temperature。
- **原始 JSON-RPC 控制台**：发送任意方法。
- **拒绝按钮**：net.connect 和 system.pairingPin 故意被路由表拒绝——证明权限表在生效。

**关键工程决策**：
- **单文件是约束而非意外**：outgrow 后变三个文件（html/js/css），但仍无 npm。
- **不用 gstwebrtc-api JS 库**："需要 npm 的客户端是没人运行的客户端"。
- **HTTP 不 HTTPS**：LAN 无证书，https 页面的 ws:// 被浏览器混合内容策略拦截。
- **浏览器旋转画面而非 pipeline**：pipeline 旋转消耗 145% CPU、97°C、降频到 408MHz、8fps。
- **两个模板占位符**：`{{SIGNALLING_PORT}}` 和 `{{API_VERSION}}` 由 mediad 替换——端口和 API 版本没有第二份副本。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 index.html |
| 行数 | 973 行 |
| 类型 | 自包含 HTML（内联 CSS + JS） |
| 服务者 | mediad（axum GET /） |
| 访问地址 | `http://<robot>:8080/` 或 `duckctl open` |
| 依赖 | 零（无 CDN、无 npm、无构建） |
| 已读范围 | 全文 973 行，完整阅读 |
| 协议 | WebSocket 信令 + WebRTC 媒体 + DataChannel JSON-RPC |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 单文件、无构建、无依赖 | L5–L6, L19–L21 | 设计约束 | confirmed |
| F2 | mediad 嵌入并服务此页面 | L5, L267–L273 | include_str! 模式 | confirmed |
| F3 | {{SIGNALLING_PORT}} 和 {{API_VERSION}} 模板占位符 | L272–L273 | mediad 替换 | confirmed |
| F4 | 直接讲 gst-plugins-rs 信令协议 | L15–L17 | 不用 JS 库 | confirmed |
| F5 | HTTP 不 HTTPS | L23–L25 | LAN 无证书 | confirmed |
| F6 | 从 checkout 打开时回退端口 8443 | L27–L30 | file:// 完全不可达 | confirmed |
| F7 | 摄像头装偏 90°，浏览器 GPU 旋转 | L70–L88, L820–L824 | pipeline 旋转烧板 | confirmed |
| F8 | pipeline 旋转：145% CPU, 97°C, 408MHz, 8fps | L822–L824 | 性能踩坑记录 | confirmed |
| F9 | 最大速度 0.3 m/s, 1.5 rad/s | L280–L281 | 与 padd 一致 | confirmed |
| F10 | 意图 10Hz 重发，deadman 500ms | L283–L284, L176–L177 | 连接断开安全停止 | confirmed |
| F11 | 无 ICE 服务器（LAN） | L414–L417 | 跨网走 bridge | confirmed |
| F12 | 机器人创建 datachannel（客户端接收） | L453–L456 | create-data-channel | confirmed |
| F13 | JSON-RPC 2.0 over datachannel | L471–L543 | id 匹配 promise | confirmed |
| F14 | 通知（无 id）：robot.state/media.video/media.detections | L519–L528 | 服务端推送 | confirmed |
| F15 | 遥测 2Hz 订阅 + 2 秒轮询 health | L599–L604 | 分离 | confirmed |
| F16 | 检测框 2.5 秒超时自动清除 | L864–L874 | 防止陈旧框 | confirmed |
| F17 | 不显示置信度数字 | L895–L898 | 量化模型共享 scale | confirmed |
| F18 | net.connect 和 system.pairingPin 故意拒绝 | L246–L250 | 证明路由表 | confirmed |
| F19 | stop/shutdown 有 confirm() 对话框 | L933–L935 | 防止误触 | confirmed |
| F20 | keyboard 焦点在 input/textarea 时不劫持 | L744 | 原始输入框不走路 | confirmed |
| F21 | 页面失焦时清空按键 | L752–L753 | 防止卡键 | confirmed |
| F22 | wheee 声音按住循环，松手停止 | L945–L959 | hold:true + 200ms 重发 | confirmed |
| F23 | 10 秒 RPC 超时 | L498–L500 | 防止 promise 泄漏 | confirmed |
| F24 | 日志最多 400 行 | L298 | 防止内存增长 | confirmed |

---

## 四、架构与协议详解

### 4.1 三层连接架构

```
浏览器（此 index.html）
  │
  ├─ HTTP GET /          → mediad axum（服务此页面，替换模板占位符）
  │
  ├─ WebSocket ws://host:port/  → 信令服务器（gst-plugins-rs protocol）
  │    ├─ welcome → list → startSession → sessionStarted
  │    ├─ peer {sdp}   ← 机器人 offer
  │    ├─ peer {sdp}   → 客户端 answer
  │    └─ peer {ice}   ↔ 客户端 ICE candidate
  │
  └─ RTCPeerConnection
       ├─ ontrack      → <video> （H.264/VP8 视频流）
       └─ ondatachannel → JSON-RPC 2.0 控制通道
            ├─ 推送：robot.state (2Hz), media.video, media.detections (2Hz)
            └─ 请求：robot.move, robot.look, robot.do, robot.sound, robot.health...
```

### 4.2 信令消息类型

| 方向 | type | 含义 |
|------|------|------|
| S→C | welcome | 服务器分配 id |
| C→S | list | 请求生产者列表 |
| S→C | list | 生产者数组（含 meta：机器人名/release/api_version） |
| C→S | startSession | 开始会话（peerId） |
| S→C | sessionStarted | 会话开始（sessionId） |
| S→C | peer | SDP offer 或 ICE candidate（含 sessionId） |
| C→S | peer | SDP answer 或 ICE candidate |
| S→C | endSession | 会话结束 |
| S→C | error | 服务器错误 |

**关键**：机器人是 offerer（webrtcsink 知道它在发什么），客户端是 answerer——这是 webrtcsink 期望的方向。

### 4.3 JSON-RPC 方法

| 方法 | 方向 | 用途 | 频率 |
|------|------|------|------|
| hello | C→S | 握手，交换 API 版本 | 连接时一次 |
| system.info | C→S | 机器人名/序列号 | 连接时一次 |
| robot.subscribe | C→S | 订阅状态流 | 连接时一次（2Hz） |
| robot.state | S→C 通知 | 策略/循环/safety/twist | 2Hz |
| robot.move | C→S tell | 速度指令 vx/vy/vyaw | 10Hz（非零时） |
| robot.look | C→S call | 注视点 IK | 10Hz（拖拽时） |
| robot.do | C→S call | 执行技能 | 按钮点击 |
| robot.sound | C→S call/tell | 播放叫声 | 按钮/按住 |
| robot.enable | C→S call | 使能切换 | 按钮 |
| robot.init | C→S call | 初始化 | 按钮 |
| robot.relax | C→S call | 放松 | 按钮 |
| robot.stop | C→S call | 零化意图（confirm） | 按钮 |
| robot.shutdown | C→S call | 关机（confirm） | 按钮 |
| robot.mode | C→S call | 查询模式 | 2 秒轮询 |
| robot.health | C→S call | 查询健康（电池/温度） | 2 秒轮询 |
| media.video | S→C 通知 | 视频尺寸/旋转 | 一次 |
| media.detections | S→C 通知 | 检测框 | ~2Hz |
| net.connect | C→S call | **被拒绝**（WiFi 重连会断当前会话） | 演示拒绝 |
| system.pairingPin | C→S call | **被拒绝** | 演示拒绝 |

---

## 五、UI 模块详解

### 5.1 头部状态栏

| 元素 | 内容 |
|------|------|
| #robot | 机器人名（meta.name → system.info.name） |
| #release | 版本号 + git short hash |
| #api | API 版本（hello 返回） |
| #state | 连接状态（idle/connecting/signalling/negotiating/connected/failed） |
| #dcstate | DataChannel 状态 |
| #signalling | WebSocket URL |
| #connect | 连接/断开按钮 |
| #banner | 版本不匹配警告（黄色条） |

### 5.2 视频区

- `<video autoplay muted playsinline>`：WebRTC 流。
- SVG overlay：检测框（viewBox 为直立帧像素）。
- 统计行：bitrate / ducks / fps / loss / rtt。
- 拖拽注视：pointerdown → aim() → robot.look。
- **旋转处理**：wrapper 保持直立，video 在内部 CSS transform 旋转（GPU 免费）。

### 5.3 驾驶区

- **移动 pad**（150×150）：上下=前后，左右=平移。
- **转向 pad**（150×44）：左右=旋转。
- **键盘**：W/S 前后，A/D 平移，Q/E 转向，方向键同义。
- pad 优先于键盘（pad 静止时为 0，键盘值才生效）。
- 最大偏转：0.3 m/s（线速度）、1.5 rad/s（角速度）。
- 非零时 10Hz 重发；归零后多发一次零。

### 5.4 姿势控制

| 按钮 | 方法 | 确认 |
|------|------|------|
| enable ⇄ | robot.enable {on:true,toggle:true} | 否 |
| init | robot.init | 否 |
| relax | robot.relax | 否 |
| stop | robot.stop | 是（"Zero the robot's intents?"） |
| shutdown | robot.shutdown | 是（"Power the robot off?"） |

**设计**：stop 不是物理急停（不断舵机电），所以不用红色紧急按钮样式。enable 用 toggle 语义，不会"卡住"。

### 5.5 技能与声音

**技能**：sit_toggle（坐/站）、ground_pick（捡东西）、kick_left（踢左）、kick_right（踢右）、roulade（滚动）。

**声音**：chirp（吱）、greet（问候）、inquire（询问）、alarm（警报）、peck（啄）、coo（咕咕）、wheee（按住骑行）。

wheee 特殊：pointerdown 时 hold:true，200ms 重发保持；pointerup/pointercancel/blur 时 hold:false。页面关闭后 hold 衰减，不会永远叫。

### 5.6 遥测表

| 行 | 来源 |
|----|------|
| mode | robot.mode 轮询（2s） |
| policy | robot.state 通知（2Hz） |
| loop | robot.state（Hz + missed count） |
| safety | robot.state（fallen/limp flags） |
| requested | robot.state.move.requested（twist 向量） |
| applied | robot.state.move.applied + limited_by 原因 |
| health | robot.health 轮询（2s） |
| battery | robot.health.battery（V + %） |
| temperatures | robot.health.motors + cpu_temp_c |

### 5.7 控制台抽屉（details）

- 两个"拒绝"按钮：net.connect 和 system.pairingPin——故意放在这里证明路由表在检查权限。
- 原始 JSON-RPC 输入框：发送任意 JSON，包括畸形消息看机器人怎么报错。
- 日志面板：最多 400 行，自动滚动。

---

## 六、关键设计决策深度解读

### 6.1 为什么不用 npm/框架

> "a client that needs npm is a client nobody runs"（L16）

**含义**：这是一个 LAN 上的机器人控制台。用户输入地址就能用。如果需要 `npm install` + 构建步骤 + 服务器托管 JS bundle，这个控制台就不存在了。单文件约束迫使一切内联。

### 6.2 为什么浏览器旋转而非 pipeline 旋转

> "Rotating in the pipeline cost 145% of a core: videoflip's buffers are ones the SoC's 2D engine refuses, so the encoder converted every frame in software — 97 °C, the CPU throttled to 408 MHz, 8 fps out of a 30 fps camera."（L822–L824）

**踩坑**：
- pipeline 中 videoflip 的 buffer 不被 SoC 2D 引擎接受。
- 编码器被迫软件转换每帧。
- 结果：CPU 145%、温度 97°C、降频到 408MHz、30fps 相机只输出 8fps。
- **解决**：CSS `transform: rotate(90deg)` 在 GPU 上做，零成本。
- **代价**：wrapper 必须保持直立（检测框坐标和拖拽映射都在 wrapper 坐标系），video 在 wrapper 内部旋转。

### 6.3 为什么 HTTP 不 HTTPS

> "ws:// from an https page is mixed content and blocked outright, and a robot on a LAN has no certificate to offer wss."（L23–L24）

**权衡**：
- HTTPS → ws:// 被浏览器拦截（混合内容）。
- wss:// 需要 TLS 证书——LAN 机器人没有。
- 所以 HTTP + ws:// 是唯一可行组合。
- 代价：麦克风在某些浏览器需要 HTTPS，游戏手柄 API 也是。docs/design/webrtc-console.md §1.3 记录了何时改变。

### 6.4 为什么两个模板占位符

> "Nothing is typed, and nothing here holds a second copy of the port — or of the API version the skew banner compares against."（L12–L13）

**单一事实源**：
- 端口由 mediad 服务时替换 `{{SIGNALLING_PORT}}`。
- API 版本由 mediad 替换 `{{API_VERSION}}`。
- 页面里没有硬编码端口或版本号。
- 如果页面从 checkout 打开（NaN），回退 8443 并显示警告——这是唯一能触发版本不匹配 banner 的场景。

### 6.5 为什么机器人创建 datachannel

> "The robot creates the datachannel, so we receive it. mediad calls create-data-channel on each consumer's webrtcbin rather than waiting for the peer, so a client that opens nothing still gets a control surface."（L453–L456）

**含义**：客户端不需要主动创建 datachannel——机器人主动创建，客户端 `ondatachannel` 接收。如果客户端自己开一个，会得到第二个未路由的通道。

### 6.6 为什么不显示检测置信度

> "The quantised model's output tensor shares one scale with the box coordinates, so every real detection reads about 1.3 and a 'confidence' of 1.3 is worse than no confidence at all."（L895–L898）

**量化模型的陷阱**：int8 量化后置信度和框坐标共享 scale，导致所有检测都显示约 1.3——这个数字毫无意义，不如不显示。数量和推理时间在统计行里。

### 6.7 为什么检测框 2.5 秒超时

> "two and a half seconds of silence means the detector stopped — and the alternative to clearing is the last duck it saw painted on the video for ever, which looks exactly like a duck that is still there."（L861–L863）

**防陈旧**：检测以 2Hz 推送。如果 mediad 重启或检测被关掉，最后一个框永远留在屏幕上——看起来像鸭子还在那里。2.5 秒无新帧自动清除。

### 6.8 为什么 stop 按钮不做红色紧急停止样式

> "It is not a physical emergency stop — nothing here cuts power to the servos — and it is a plain button for that reason."（L191–L192）

**诚实 UI**：stop 只是零化当前会话发送的意图，不断舵机电。做成红色 E-stop 会误导用户以为它能物理断电。shutdown 才是真关机（且有确认对话框）。

---

## 七、性能与安全考量

### 7.1 死人开关（deadman）

- 意图以 10Hz 重发，不是锁存。
- 机器人端 `safety.deadman_ms` = 500ms：500ms 没收到意图就自己停。
- 显式零在松开按键时立即发送——不等半秒。
- 连接断开 → onclose → stopTelemetry → inflight promises 全部 reject。

### 7.2 错误诊断

WebSocket onerror 区分两种情况：
- **页面来自机器人**（SERVED_BY_ROBOT=true）：页面能加载说明机器人在线，问题在信令端口——大概率防火墙，看 `journalctl -u mediad -b`。
- **页面来自 checkout**：可能 host 错或 mediad 没跑，建议 `duckctl open`。

### 7.3 键盘焦点保护

> "Keys are the robot's while nothing text-shaped has focus — the raw box in the drawer is a text field, and typing robot.mode into it should not walk the robot across the room."（L741–L743）

在 input/textarea/select 中按键不触发驾驶。

### 7.4 promise 泄漏防护

- 每个 RPC 10 秒超时。
- stopTelemetry 时 reject 所有 inflight——不是忘记。
- 否则重连后会有两套计时器和悬挂的 promise。

---

## 八、效果主张与责任闭合卡（EFFECT）

### 8.1 主张："此页面由 mediad 服务，端口和 API 版本由服务端注入"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | {{SIGNALLING_PORT}} 和 {{API_VERSION}} | L272–L273 |
| 触发者 | HTTP GET / | mediad web.rs |
| 当前装配/选择/开关 | mediad 替换模板占位符 | L267–L268 注释 |
| 实际执行者 | mediad include_str! 读取此文件并 string replace | — |
| 成功副作用与观察点 | SERVED_BY_ROBOT = Number.isFinite(SERVED_PORT) | L274 |
| 失败是否返回且被检查 | NaN → 回退 8443 + 警告 banner | L275, L571–L574 |
| 不能覆盖的对象 | 文件系统直接打开（file://）→ Chrome PNA 阻止访问私有地址 | L28–L30 |
| status | **confirmed** | — |

### 8.2 主张："net.connect 和 system.pairingPin 被路由表拒绝"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | 两个按钮的调用 | L246–L250 |
| 触发者 | 用户点击 | — |
| 当前装配/选择/开关 | mediad/src/route.rs 路由表 | L32–L34 注释 |
| 实际执行者 | 机器人端路由检查 | — |
| 成功副作用与观察点 | 返回 JSON-RPC error | — |
| 失败是否返回且被检查 | 错误显示在日志中 | onControl error 分支 |
| 不能覆盖的对象 | BLE 通道允许 net.connect（重连 WiFi 不会断 BLE） | L246–L247 注释 |
| status | **confirmed**（注释明确说明这两个调用在 BLE 允许但 Web 拒绝） | — |

---

## 九、边界与反例（BREAK）

1. **无 ICE 服务器**：纯 LAN 直连。跨网需要 bridge（§7）+ STUN——不是此页面的问题。
2. **HTTP 明文**：所有控制指令在 LAN 上明文传输。没有认证/加密。
3. **无 HTTPS**：麦克风在 Chrome 中需要安全上下文——在此页面不可用。
4. **无视频录制**：只显示，不保存。
5. **单机器人**：connect 只连一个 producer。
6. **检测框坐标在直立帧**：不随旋转变化——因为 wrapper 保持直立。
7. **拖拽注视是近似**：LOOK 常量是"机器人前方一米、半米盒子"的猜测，不是标定——IK clamp 会报告不可达。
8. **wheel event 未处理**：只有 pointer 和 keyboard。
9. **无多语言**：所有 UI 文本英文。
10. **无 PWA/service worker**：刷新需要重新连接。
11. **日志限制 400 行**：长会话早期日志丢失。
12. **无重连逻辑**：onclose 后需要手动点 connect。
13. **wheee 200ms 重发**：不是 10Hz，是 5Hz——因为它是 hold 衰减，不是速度意图。
14. **音频输出未实现**：虽然有声音按钮，但音频是否从扬声器输出还是只是 robot.sound 方法调用，取决于机器人端实现。

---

## 十、与其他 crate 的关联

```
index.html
  ↑ include_str! + string replace
mediad
  ├── axum GET /          → 服务此页面
  ├── axum GET /ws        → WebSocket 信令
  ├── gstreamer-webrtc    → webrtcbin + create-data-channel
  ├── gstreamer pipeline  → v4l2src → tee → [webrtcbin, appsink→duck-detect]
  └── duck-ipc-proto      → JSON-RPC 方法 schema
       ├── robot.move     → duck-control crate
       ├── robot.look     → kinematics crate（IK）
       ├── robot.do       → skills 执行器
       └── robot.health   → 各 daemon 聚合
```

**对应之前分析的文件**：
- `duck-ipc-proto`：定义所有方法的 schema（mediad Cargo.toml 依赖）。
- `duck-detect`：检测引擎，通过 media.detections 推送框。
- `kinematics`：robot.look 的 IK 求解（head.rs 凝视 IK）。
- `robotd-params`：[media]/[detect] 配置。

---

## 十一、结论（按状态分级）

### confirmed
- C1：这是 duck console，mediad 服务的单文件 Web 控制台（973 行）。
- C2：零依赖，无 npm，无构建步骤。
- C3：两层连接——WebSocket 信令 + RTCPeerConnection（视频+datachannel）。
- C4：JSON-RPC 2.0 over datachannel，10 秒超时，id 匹配 promise。
- C5：机器人创建 datachannel，客户端接收。
- C6：视频旋转在浏览器 GPU（pipeline 旋转烧板：145% CPU, 97°C, 408MHz, 8fps）。
- C7：最大速度 0.3 m/s, 1.5 rad/s；意图 10Hz 重发，deadman 500ms。
- C8：检测框 2.5 秒超时，不显示置信度（量化模型 scale 问题）。
- C9：net.connect 和 system.pairingPin 故意被路由表拒绝。
- C10：stop/shutdown 有 confirm 对话框；stop 不是物理急停。
- C11：HTTP 不 HTTPS（LAN 无证书，ws from https 被拦截）。
- C12：模板占位符 {{SIGNALLING_PORT}}/{{API_VERSION}} 由 mediad 注入。

### inferred
- I1：webrtcbin 配置为 webrtcsink 方向（机器人 offer，客户端 answer）。
- I2：检测框坐标在直立帧（wrapper 坐标系），不随旋转变化。
- I3：detector 约 2Hz 推送（"a couple of times a second"）。
- I4：bridge 模式（§7）用于跨网访问，需要 STUN——不在此页面实现。

### unknown
- U1：信令 WebSocket 的端口号（{{SIGNALLING_PORT}} 由 mediad 注入，可能是 8443）。
- U2：视频编码格式（H.264? VP8? AV1?）。
- U3：bridge 模式的具体架构。
- U4：麦克风/音频上行是否实现。
- U5：gamepad API 是否可用（HTTP 下可能不可用）。

---

## 十二、未知项与最小验证动作

| 未知项 | 最小验证动作 | 预期通过信号 |
|--------|-------------|-------------|
| U1 信令端口 | 读取 mediad/src/web.rs 或 config | 看到默认端口 |
| U2 视频编码 | 读取 pipeline 字符串 | 看到 x264/openh264 |
| U3 bridge | 读取 docs/design/webrtc-console.md §7 | 看到 TURN/STUN 配置 |
| U4 音频上行 | grep 'mic\|audio\|getUserMedia' index.html | 当前无——未实现 |
| U5 gamepad | grep 'gamepad' index.html | 当前无——未实现 |

---

## 附录 A　资料来源

1. 原文件：index.html（本地附件，973 行，全文已读）。
2. 关联文件：mediad/Cargo.toml（上一轮分析）、duck-ipc-proto、kinematics crate。
3. 外部参考：gst-plugins-rs WebRTC 信令协议（net/webrtc/protocol at 0.15.3）。
4. 设计文档：docs/design/webrtc-console.md（引用 §1.1, §1.3, §3, §7）。
5. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-11 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
