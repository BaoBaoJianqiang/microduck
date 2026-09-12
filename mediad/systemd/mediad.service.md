# mediad.service 文件解析

**文件位置**：`d:\microduck\mediad\systemd\mediad.service`

## 核心设计决策

`mediad` 的 systemd unit。两条负载的配置容易被忽略，且运行时表现完全相同——编码器干脆不存在，毫无原因说明：`SupplementaryGroups=` 与 `Environment=GST_PLUGIN_PATH=`。

无人连接时运行是安全的：`webrtcsink` 监听、不产生消费者、管线停在 PLAYING。**它不认证**——能到达信令端口的任何人都能驱动机器人、看摄像头。决策而非遗漏（`remote-webrtc.md` §4，配对 PIN 是共享 000000，门禁无意义）。

## [Unit]

- `Description=Robot media and WebRTC gateway`
- `Documentation=file:///opt/robot/daemon/current/docs/design/remote-webrtc.md`
- `After=robotd.service configd.service updaterd.service local-fs.target`、`Wants=robotd.service`——`After` 非 `Requires`：转发到的服务挂掉不能阻止本 unit 启动；对等端在 `robotd` 重启时连接会得到"robotd is not answering"，诚实答案好过 `mediad` 拒绝运行。

## [Service]

- `Type=exec`
- **`ExecStart=/opt/robot/daemon/current/bin/mediad`**——无视频参数。流化什么（摄像头/测试图案、帧尺寸、码率）全在 `/etc/robot/robotd.toml` 的 `[media]`，由 `robotctl configure` 编辑。曾是此命令行上的 flag，release 安装器重写本文件，改一个就要 systemd drop-in——没人会为"视频为什么糊"去碰 drop-in。
  - **摄像头默认开的后果**：控制数据通道与视频轨捆绑，无摄像头（未插或 overlay 未启用）的板子起不来本服务，连 WebRTC 控制面一起丢。BLE 与本地 pad 不受影响。修复：`[media] camera = false` 流化测试图案。
- `User=mediad`、`Group=mediad`、`SupplementaryGroups=video render robot`：
  - **`video`**：`/dev/mpp_service`（VPU）、`/dev/rga`（2D 加速器）、`/dev/videoN`（采集节点）。三者都重要且失败方式不同：无 VPU 节点编码器**静默不注册**；无 `/dev/rga` 元素存在但管线在 RGA 内失败；采集节点本就是 root:video。
  - **`robot`**：robotd/configd/updaterd 的 0660 socket；padd/tofd 也把 socket 交给 `robot` 组。
  - **`render` 即 NPU**：rknpu 驱动注册为 DRM 设备 `root:render` 660；不加入此组检测器打不开 NPU，失败是 `rknn_init` 返回数字而非权限错误。哪个 `renderD*` 不固定（Radxa Zero 3 上显示与 Mali 已占 128/129）。
- `Restart=always`、`RestartSec=5s`——缺失插件或设备节点是应重试的状态。
- 硬化：`NoNewPrivileges`、`ProtectSystem=strict`、`ProtectHome`、`PrivateTmp`、`ProtectKernelTunables/Modules`、`ProtectControlGroups`、`RestrictSUIDSGID`、`RestrictNamespaces`、`LockPersonality`、`CapabilityBoundingSet=`。
  - **不设 `PrivateDevices=yes`**：会隐藏 `/dev/mpp_service`、`/dev/rga`、采集节点。
  - **`RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6`**：信令服务器 TCP + WebRTC UDP 必需。
  - **不设 `MemoryDenyWriteExecute=yes`**：GStreamer dlopen 插件，硬件编解码栈会映射可执行页。
- `Environment=GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0`——GStreamer 默认不搜此路径（默认是发行版路径，插件刻意避开以防 apt 操作替换移除）。无此则 `mpph264enc`/`webrtcsink` 不存在。
- `Environment=RUST_LOG=info`
- `RuntimeDirectory=mediad`——`/run/mediad/` 放 `identity.json`。`RuntimeDirectory=` 是唯一能创建由本 unit `User=` 拥有且在 `ProtectSystem=strict` 下存活的目录的机制；stop 时 systemd 移除，停止的守护进程不能留下声称在运行的 identity。

## [Install]

- `WantedBy=multi-user.target`——`hooks/postinstall` 对每个带 `[Install]` 的 unit 启用并启动；`updater` 的 `units_shipped` 在 apply 时重启所有此类 unit。此段是让 release 安装出可用摄像头并拾取下一个 release 的全部。
- 两种无人看时的失败：1) 无 GStreamer 栈（`robot-setup-gstreamer`）；2) 无摄像头（`robotctl configure` 关 `media.camera`）。两者都在 `robotctl health` 可见，不影响控制循环/BLE/pad/更新。

## 关键摘要

- `SupplementaryGroups=video render robot` 与 `GST_PLUGIN_PATH` 是两条负载配置，缺一则编码器不存在。
- 流化参数全在 `robotd.toml`，不在 ExecStart。
- 硬化到硬件编解码进程能承受的极限，但保留 `/dev` 访问、网络族、可执行内存映射。
- `[Install]` 段让 release 自动启用/重启本服务。
