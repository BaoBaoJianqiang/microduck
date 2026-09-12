# session.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\session.rs`

## 核心设计决策

一个对等端的控制通道，作为通往持有答案的服务的管道。

刻意与传输无关：只接收行并返回行，对数据通道一无所知。这使得无需 WebRTC 对等端即可测试（测试用 channel 驱动它，对抗真实 unix socket 上的假守护进程），也让未来 WebSocket 表面可原样复用。

### 它不做什么

1. **从不解析回复。** 请求只读到足以路由的程度；服务发出的一切原样转发。两个推论：
   - 订阅无需特殊处理——它是开放连接上的通知流，每条都要到对等端；把回复关联到请求会破坏它（只留第一条）。
   - 给 API 加方法在此零成本，`duck-ipc-proto` 仍是唯一定义方法的地方。
2. **不认证。** `remote-webrtc.md` §4：机器人上无门禁；LAN 对等端可驱动，桥接对等端在到达前已在会合服务两侧认证。`route::permits` 直接拒绝 `system.authenticate`。

## 类型

### `Video`
`width/height/rotate`（摄像头安装偏离直立的顺时针角度）。会话持有它，让对等端能**问**（`media.video` 调用）而非仅靠推送——推送会在浏览器数据通道未开时被丢弃（实机发生过，画面横置且日志无解释）。推送仍保留作只监听客户端的礼貌。

### `run(inbound, outbound, pool, video)`
驱动一个对等端的控制通道直到入站流结束。

### `handle(line, pool, video) -> Option<String>`
路由一行。只在本传输自己应答时返回回复（即拒绝时）。

处理流程：
1. 解析 `Request`；失败返回 `PARSE_ERROR`（null id，因无可回显的 id）。
2. 若 `method == "media.video"`，直接返回 `Video` 的成功应答（不路由，无服务拥有它）。
3. `as_call()`：未知方法或参数形状不匹配则返回该错误（版本偏差在无法服务的那个调用上失败，而非握手时）。
4. `route_for`：`Refused` 返回 `route::refusal`；`To` 则 `pool.send`，失败返回 `INTERNAL_ERROR` + "service is not answering"。

## 辅助函数

- `video_notification(video)`：`media.video` 无 id 通知（与 `robot.state` 同机制）。
- `error_line(id, error)`：通过 `proto::Response` 构造错误行，信封只有一份定义。

## 单元测试

| 测试 | 意图 |
|---|---|
| `a_permitted_call_reaches_its_service` | 允许的调用到达正确服务且回复返回，不到错服务 |
| `a_refused_call_is_answered_here_and_forwarded_nowhere` | 拒绝的调用在此应答、不到达 socket；以 `net.connect` 为例（BLE 允许、WebRTC 拒绝） |
| `the_pad_tap_reaches_padd` | pad 输入到达 `btd` 故意不持有的 `padd` socket |
| `every_notification_in_a_stream_reaches_the_peer` | 流中每条通知都到对等端（若把回复关联到请求会破坏此） |
| `an_absent_service_is_reported` | 服务缺席要报告而非沉默（`robotd` 更新重启时最可能缺席） |
| `an_unknown_method_names_itself` | 未知方法按名拒绝 |
| `the_video_notification_carries_the_mount_rotation` | 通知携带安装旋转角 |
| `the_page_can_ask_what_the_video_is` | `media.video` 由会话应答，不路由到服务 |
| `garbage_gets_a_parse_error` | 垃圾得到 parse error，id 为 null |

## 关键摘要

- 管道语义：入站一行 → 路由 → 出站原样转发回复/通知。
- 不解析回复，所以订阅流与加方法都零成本。
- `media.video` 在此应答（无服务拥有），让对等端在就绪时主动问，避免推送竞态。
- 缺席服务要报告，区分"在想"与"永远不回"。
