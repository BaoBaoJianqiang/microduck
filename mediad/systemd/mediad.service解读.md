# `mediad.service` 解读

## 概述

`mediad` 的 systemd service unit 文件，负责摄像头、麦克风、WebRTC 和远程网关。

**两条容易被忽略的承重线**，两者在运行时表现相同——编码器就是不存在，没有说明为什么：
- `SupplementaryGroups=`
- `Environment=GST_PLUGIN_PATH=`

各自有注释解释。

### 无连接时安全运行

> It is safe to have running with nobody connected: `webrtcsink` listens, produces no consumer, and the pipeline sits at PLAYING. What it does *not* do is authenticate — anyone who reaches the signalling port can drive the robot and see its camera. That is a decision, not an omission; see `docs/design/remote-webrtc.md` §4, whose short version is that the pairing PIN is a shared 000000, so a gate would add a step to every connection and prove nothing.

无人连接时运行是安全的：`webrtcsink` 监听、无消费者、管线停在 PLAYING。但它**不做认证**——任何到达信令端口的人都可以驱动机器人并看到摄像头。这是一个决定而非遗漏：配对 PIN 是共享的 `000000`，所以加门只会给每次连接增加一步，证明不了什么。

---

## [Unit] 段

```ini
Description=Robot media and WebRTC gateway
Documentation=file:///opt/robot/daemon/current/docs/design/remote-webrtc.md
```

```ini
After=robotd.service configd.service updaterd.service local-fs.target
Wants=robotd.service
```

### `After` 而非 `Requires`

> `After`, not `Requires`, exactly as padd has it: the services mediad forwards to are where calls go, so starting first is pointless — but any of them being down must not keep this unit from starting. A peer that connects while robotd is restarting gets "robotd is not answering", which is the honest answer and better than mediad refusing to run.

- **`After`**：mediad 在 robotd/configd/updaterd 之后启动，但不要求它们必须运行
- **`Wants=robotd.service`**：弱依赖，robotd 不在时尝试启动它但不失败
- **为什么不用 `Requires`**：mediad 转发到的服务是调用的目的地，先启动没有意义；但它们中的任何一个宕机都不能阻止这个单元启动。peer 在 robotd 重启时连接得到 "robotd is not answering"——这是诚实的答案，比 mediad 拒绝运行好。

---

## [Service] 段

### 执行入口

```ini
Type=exec
ExecStart=/opt/robot/daemon/current/bin/mediad
```

### 为什么没有视频标志

> **No video flags here, on purpose.** What this daemon streams — camera or test pattern, frame size, rate, bitrate — is `[media]` in `/etc/robot/robotd.toml`, edited with `robotctl configure` and applied by `systemctl restart mediad`.

视频参数（摄像头还是测试图案、帧大小、帧率、码率）在 `/etc/robot/robotd.toml` 的 `[media]` 段中，用 `robotctl configure` 编辑。

**曾经是这个行上的标志。** 发布安装器重写这个文件，所以改一个意味着 systemd drop-in——`systemctl edit --full` 把编辑放在会被覆盖的确切位置——没有人会为了回答"为什么视频模糊"去找 drop-in。配置文件是每板设置在更新*和*回滚后存活的唯一场所。

### 摄像头默认开启的后果

> **The consequence of the camera being on by default, stated because it is not obvious:** the control datachannel is bundled with the video track, so a robot whose camera is absent — unplugged, or the device-tree overlay not enabled — fails to start this service and loses its WebRTC control surface along with its video. BLE and the local pad are unaffected. The fix on such a board is `camera = false` in `[media]`, which streams a test pattern: the pipeline starts, so the control channel exists.

**控制数据通道绑定在视频 track 上。** 摄像头缺失（未插、设备树 overlay 未启用）的机器人会导致这个服务启动失败，同时失去 WebRTC 控制界面和视频。BLE 和本地 pad 不受影响。

修复：在 `[media]` 中设 `camera = false`，流测试图案——管线启动，控制通道存在。

### 用户和组

```ini
User=mediad
Group=mediad
SupplementaryGroups=video render robot
```

#### 三个补充组

**`video`** — `/dev/mpp_service`（VPU）、`/dev/rga`（2D 加速器，编码器用于格式和步幅转换）、`/dev/videoN`（采集节点）。

> All three matter and each fails differently: without the VPU node the encoders are *silently not registered*, because an MPP plugin probes MPP before registering them; without `/dev/rga` the element exists and the pipeline fails inside RGA; the capture nodes arrive root:video already. Three debugging rounds, one cause — `docs/project/media-bringup.md` records them.

三个都重要，各自失败方式不同：
- 没有 VPU 节点：编码器**静默不注册**（MPP 插件在注册前探测 MPP）
- 没有 `/dev/rga`：元素存在，管线在 RGA 内部失败
- 采集节点已经是 root:video

三轮调试，一个原因。

**`robot`** — robotd、configd、updaterd 的 0660 socket，和 btd 及 SDK 到达它们的方式相同。padd 和 tofd 的 socket 由它们自己交给 `robot` 组，所以 pad 触控和深度流通过同一组成员可达。

**`render` 是 NPU** — rknpu 驱动注册为 DRM 设备，`root:render` 模式 660，所以 duck 检测器没有这个组不能打开 NPU——失败是 `rknn_init` 返回一个数字而非关于权限的任何东西。

> Which `/dev/dri/renderD*` it takes is not fixed and is not worth writing down: on a Radxa Zero 3 the display and Mali already hold 128 and 129 before the NPU binds at all, so a number recorded here would name somebody else's device. Membership costs nothing on a board whose NPU is disabled; it is the difference between working and not on one where it is enabled.

哪个 `/dev/dri/renderD*` 不固定，不值得写下来：在 Radxa Zero 3 上显示和 Mali 在 NPU 绑定前已经占了 128 和 129，所以写死的数字会命名别人的设备。NPU 禁用的板子上组成员不花代价；NPU 启用的板子上它是工作与不工作的区别。

### 重启策略

```ini
Restart=always
RestartSec=5s
```

> Restart=always for padd's reason: mediad exits when it cannot start a pipeline, and a missing plugin or an unopenable device node is a state to retry rather than one to stay down in — a provisioning run that installs the plugins should be enough to bring it up without a reboot.

`Restart=always`，5 秒重试。mediad 在无法启动管线时退出，缺失插件或不可打开的设备节点是应该重试的状态而非应该保持宕机的状态——安装插件的配置运行应该足以让它启动而无需重启。

### 安全加固

```ini
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
RestrictNamespaces=yes
LockPersonality=yes
CapabilityBoundingSet=
```

#### 为什么不用 `PrivateDevices=yes`

> NOT PrivateDevices=yes: it would hide /dev/mpp_service, /dev/rga and the capture nodes, which are the point of the process. Which nodes it may open is still bounded by the `video` group.

`PrivateDevices=yes` 会隐藏 `/dev/mpp_service`、`/dev/rga` 和采集节点——这些是进程的全部目的。可以打开哪些节点仍由 `video` 组限制。

#### 为什么保留 AF_INET 和 AF_INET6

> AF_INET and AF_INET6 are load-bearing: the signalling server listens on a TCP socket and WebRTC is UDP. Without them the daemon starts and no peer can ever reach it, which is the same shape of failure as padd's missing AF_NETLINK.

信令服务器监听 TCP socket，WebRTC 是 UDP。没有它们守护进程启动但没有 peer 能到达——和 padd 缺失 AF_NETLINK 相同形状的失败。

#### 为什么不用 `MemoryDenyWriteExecute=yes`

> NOT MemoryDenyWriteExecute=yes, unlike padd. GStreamer dlopens its plugins, and a hardware codec stack is exactly the kind of thing that maps executable pages at load time. Denying it would make the daemon fail in the dynamic loader, some way from anything that mentions media.

GStreamer dlopen 它的插件，硬件编解码器栈正是那种在加载时映射可执行页的东西。拒绝它会让守护进程在动态加载器中失败，离任何提到 media 的东西都很远。

### 环境变量

```ini
Environment=GST_PLUGIN_PATH=/usr/local/lib/gstreamer-1.0
Environment=RUST_LOG=info
```

#### `GST_PLUGIN_PATH` 是必须的

> GStreamer does not search /usr/local/lib/gstreamer-1.0 by default — its built-in path is the distro's /usr/lib/aarch64-linux-gnu/gstreamer-1.0, which the plugins deliberately avoid so that an apt operation cannot replace or remove them. So this is not optional: without it mpph264enc and webrtcsink do not exist, and the daemon says so and exits.

GStreamer 默认不搜索 `/usr/local/lib/gstreamer-1.0`——它的内置路径是发行版的 `/usr/lib/aarch64-linux-gnu/gstreamer-1.0`，插件刻意避免它，以便 apt 操作不能替换或移除它们。所以这不是可选的：没有它 `mpph264enc` 和 `webrtcsink` 不存在，守护进程报告并退出。

**这是两条容易忽略的承重线之一。**

```ini
StandardOutput=journal
StandardError=journal
```

### 运行时目录

```ini
RuntimeDirectory=mediad
```

> /run/mediad/, for identity.json — what this daemon is running, read by `robotctl health` and by updaterd's startup check. `RuntimeDirectory=` rather than a granted path, for padd's reasons: it is the only mechanism that creates a directory owned by this unit's User= *and* survives ProtectSystem=strict, and systemd removes it on stop so a stopped daemon cannot leave behind an identity claiming to be running.

`/run/mediad/` 用于 `identity.json`——这个守护进程在运行什么，被 `robotctl health` 和 updaterd 的启动检查读取。

`RuntimeDirectory=` 而非授权路径：它是唯一创建由这个单元的 `User=` 拥有的目录*并且*在 `ProtectSystem=strict` 下存活的机制，systemd 在停止时移除它，所以停止的守护进程不能留下声称在运行的 identity。

---

## [Install] 段

```ini
WantedBy=multi-user.target
```

### 为什么有这个段

> **This section is what makes `mediad` a daemon rather than something to remember.**

`hooks/postinstall` 启用并启动每个带 `[Install]` 段的单元，`updater/src/engine.rs` 中的 `units_shipped` 在 apply 时重启每个这样的单元——一条规则，从单元文件读取而非任何人维护的列表。所以这个段是让发布安装成为工作摄像头的全部，也是让它捡起下一个的全部，它的缺失就是为什么每次推送都以有人输入 `systemctl restart mediad` 结束。

**它在 mediad 从未在板子上启动时不存在。它启动了，所以它在这里。**

### 两种无人看管时的失败方式

> Both are visible in `robotctl health` as a unit that is not running, and neither touches the control loop, BLE, the pad or an update:

两者在 `robotctl health` 中显示为未运行的单元，都不触及控制循环、BLE、pad 或更新：

1. **没有 GStreamer 栈的板子**：配置安装它，每个更新的 pre-install hook 也安装它（`hooks/preinstall` 运行发布版自己的 `setup-gstreamer.sh`），所以到达这种状态的板子两者都失败了，最可能是因为缺网络。管线不能启动，`Restart=always` 每 5 秒重试，手动重跑是 `sudo /usr/local/sbin/robot-setup-gstreamer`。

2. **没有摄像头的板子**，因为 `[media] camera` 默认开启。相同形状，修复是 `sudo robotctl configure` → `media.camera` off。

### `WantedBy=multi-user.target` 而非手工 symlink

> The release owns which units exist and systemd owns whether they are enabled, and a robot whose enablement was done by hand on one board is a robot nobody can reason about.

发布版拥有哪些单元存在，systemd 拥有它们是否启用。在一个板子上手做启用的机器人是没有人能推理的机器人。

---

## 与其他文件的关系

- **`robotd.toml` `[media]`**：视频参数（camera、帧率、码率等）的配置位置
- **`remote-webrtc.md` §4**：为什么不做认证的设计决策
- **`architecture.md` §3**：为什么配置文件是每板设置在更新和回滚后存活的唯一场所
- **`media-bringup.md`**：三个组各自失败方式的调试记录
- **`padd.service`**：相同的 `After`/`Wants` 模式和安全加固策略
- **`updater/src/engine.rs`**：`units_shipped` 在 apply 时重启所有带 `[Install]` 段的单元
- **`hooks/postinstall`**：启用并启动所有带 `[Install]` 段的单元
- **`hooks/preinstall`**：运行 `setup-gstreamer.sh` 安装 GStreamer 栈

---

## 关键踩坑点总结

1. **两条承重线在运行时表现相同**：`SupplementaryGroups=` 和 `GST_PLUGIN_PATH=` 缺失时，编码器就是不存在，没有说明为什么。分别花了三轮调试。

2. **`video` 组缺 VPU 节点 → 编码器静默不注册**：MPP 插件在注册前探测 MPP，失败了就不注册，没有错误消息。这是最隐蔽的失败模式。

3. **`render` 组是 NPU**：rknpu 驱动注册为 DRM 设备 `root:render` 模式 660。不能写死 `/dev/dri/renderD*` 编号（显示和 Mali 占了 128/129），组成员是正确方式。

4. **`GST_PLUGIN_PATH` 必须指向 `/usr/local/lib`**：插件刻意避免发行版路径以便 apt 不替换它们，但 GStreamer 默认不搜索 `/usr/local/lib`。没有这个环境变量插件不存在。

5. **摄像头缺失导致 WebRTC 控制通道也丢失**：控制数据通道绑定在视频 track 上。摄像头未插/设备树未启用 → mediad 启动失败 → 失去 WebRTC 控制界面。修复是 `camera = false` 流测试图案。

6. **不用 `PrivateDevices=yes`**：会隐藏 `/dev/mpp_service`、`/dev/rga`、`/dev/videoN`——这些是进程的全部目的。

7. **不用 `MemoryDenyWriteExecute=yes`**：GStreamer dlopen 插件，硬件编解码器栈在加载时映射可执行页。

8. **保留 AF_INET/AF_INET6**：信令 TCP + WebRTC UDP，没有它们守护进程启动但无人能到达。

9. **`[Install]` 段是让发布自动化工作的全部**：缺失时每次推送都要手动 `systemctl restart mediad`。有了它，postinstall 自动启用，updater apply 自动重启。

10. **视频参数不在 service 文件中**：曾经是标志，安装器重写这个文件导致 drop-in 被覆盖。改为 `robotd.toml [media]`，每板设置在更新和回滚后存活。
#（注：内容由AI生成）
