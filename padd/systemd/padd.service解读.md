# `padd.service` 解读

## 概述

`padd` 的 systemd service unit——gamepad 意图客户端。

### 为什么要有这个 unit

> Why this is a unit at all. Driving used to mean: ssh to the board, be a member of two groups (having logged out and back in since), and run a binary by its full path — which died with the ssh session. So the pad worked when someone had done all of that, and the failures were invisible in three separate places. Now pairing a pad (`robotctl pad pair`) is the only step, and this runs from boot waiting for one.

以前驾驶意味着：ssh 到板子、是两个组的成员（自那以后登出再登入过）、用完整路径运行二进制——随 ssh 会话死亡。所以手柄只在有人做完所有那些事时才工作，失败在三个不同地方不可见。现在配对手柄（`robotctl pad pair`）是唯一步骤，这个单元从启动就运行等待。

### 无手柄时安全运行

> It is safe to have running with no pad, which is what makes an always-on unit the simple answer rather than udev rules or device-unit binding: `padd` sends *nothing* when it sees no pad, and robotd's deadman holds the robot on its own. A pad that connects is picked up within a tick.

无手柄时运行安全，这让 always-on unit 成为简单答案而非 udev 规则或设备单元绑定：padd 看到无手柄时**什么都不发送**，robotd 的 deadman 自己停止机器人。连接的手柄在一个 tick 内被捡起。

### 连接时也不动任何东西

> Nothing moves on connection either — the policy is off until someone presses Start. What a paired pad grants is the authority to enable it, and *pairing* is the gate on that: it takes a pad held in pairing mode next to the robot, and `robotctl pad forget` revokes it.

连接时也不动任何东西——策略在有人按 Start 之前是关的。配对手柄授予的是启用它的权威，*配对*是门控：它需要一个在机器人旁边保持配对模式的手柄，`robotctl pad forget` 撤销它。

---

## [Unit] 段

```ini
Description=Robot gamepad intent client
Documentation=file:///opt/robot/daemon/current/docs/architecture.md
```

### After 而非 Requires

```ini
After=robotd.service local-fs.target
Wants=robotd.service
```

robotd 是意图去的地方，先于它启动没有意义——但 robotd 宕机不能阻止这个单元启动。如果 socket 还不存在，padd 退出，systemd 五秒后重试，这就是它在慢速启动时追上的方式。

### 蓝牙 73 秒

```ini
After=bluetooth.service
```

> The pad arrives over Bluetooth, and on this board hci0 does not exist until roughly 73 seconds after power-on. Ordering after bluetooth.service is nowhere near sufficient for that and is not meant to be: `padd` polls for a pad forever, so the radio can appear whenever it likes.

手柄通过蓝牙到达，在这个板子上 hci0 直到上电后约 73 秒才存在。在 bluetooth.service 之后排序远不足以解决这个问题，也不打算解决：padd 永远轮询手柄，所以无线电可以随时出现。

---

## [Service] 段

### 执行入口

```ini
Type=exec
ExecStart=/opt/robot/daemon/current/bin/padd
```

### 用户和组

```ini
User=padd
Group=padd
SupplementaryGroups=input robot
```

### 为什么非 root 是进程的目的

> Unprivileged, and this one is not a hardening choice — it is the point of the process. `padd` has no privileged access to the robot, which is what makes it an honest exercise of the intent API every day (see the crate docs).

非特权，这一个不是加固选择——它是进程的目的。padd 对机器人没有特权访问，这正是它每天成为意图 API 诚实练习的原因。

| 组 | 权限来源 | 访问什么 |
|----|----------|----------|
| `input` | `/dev/input/event*`（root:input 0660） | 手柄事件节点 |
| `robot` | robotd 的 0660 socket | 发送意图，和 btd/SDK 相同方式 |

用户和组必须存在否则单元启动失败；见 `sysusers.d/padd.conf`。

### 重启策略

```ini
Restart=always
RestartSec=5s
```

`Restart=always` 而非 `on-failure`：padd 在 robotd socket 不存在时干净退出，这是应该重试的状态而非应该保持宕机的状态。五秒因为等待的是启动完成或守护进程重启，自旋没有帮助。

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
RestrictAddressFamilies=AF_UNIX AF_NETLINK
RestrictNamespaces=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
CapabilityBoundingSet=
```

#### 为什么不用 PrivateDevices=yes

> NOT PrivateDevices=yes: that hides /dev/input, which is the one thing this process exists to read. The device nodes it may open are still limited by the `input` group.

`PrivateDevices=yes` 会隐藏 `/dev/input`——这是这个进程存在来读的唯一东西。可以打开的设备节点仍由 `input` 组限制。

#### AF_NETLINK 是承重线

> AF_NETLINK is load-bearing and easy to miss: gilrs enumerates and *watches* devices through libudev, which is a netlink socket. Without it a pad connected after startup is never noticed, which presents as "the pad works only if it was on before the robot booted".

**AF_NETLINK 是承重线且容易忽略**：gilrs 通过 libudev 枚举并*监视*设备，libudev 是一个 netlink socket。没有它，启动后连接的手柄永远不被注意，表现为"手柄只在机器人启动前开着时才工作"。

这与 padd.service 中注释的另一个坑呼应——蓝牙 73 秒后 hci0 才出现，如果没有 AF_NETLINK，padd 永远看不到这个事件。

#### MemoryDenyWriteExecute=yes

与 mediad 不同，padd 可以设 `MemoryDenyWriteExecute=yes`——padd 不 dlopen 插件，没有硬件编解码器栈在加载时映射可执行页。

#### SystemCallFilter

```ini
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
```

只允许 `@system-service` 类别的系统调用，其他返回 EPERM。这限制了 padd 能做的系统调用集合。

### 环境变量和输出

```ini
Environment=RUST_LOG=info
StandardOutput=journal
StandardError=journal
```

padd 在开始驾驶、策略切换、手柄出现/消失时记 `warn`——有人读 journal 来找的事件——不记每 tick 的事件。

### 运行时目录

```ini
RuntimeDirectory=padd
```

`/run/padd/` 持有两样东西：
1. **`identity.json`**：这个守护进程在运行什么，被 `robotctl health` 和 updaterd 的启动检查读取
2. **`pad.sock`**：只读原始输入 tap，`robotctl monitor` 显示手柄自己的事件流（`src/tap.rs`）

#### 为什么用 RuntimeDirectory= 而非授予路径

> `RuntimeDirectory=` rather than a path this unit is granted: it is the only mechanism that both creates the directory owned by this unit's User= *and* survives ProtectSystem=strict, which otherwise leaves the whole filesystem read-only. systemd also removes it when the unit stops, so a stopped daemon cannot leave behind an identity claiming to be running — nor a socket nothing is listening on, which is the difference between "padd is not running" and a client hanging.

唯一既创建由 `User=` 拥有的目录又在 `ProtectSystem=strict` 下存活的机制。systemd 在单元停止时移除它，所以停止的守护进程不能留下声称在运行的 identity——也不能留下没人监听的 socket，这是"padd 没在运行"和客户端挂起的区别。

#### tap socket 的组分配

> The tap's socket is 0660 and handed to the `robot` group by `padd` itself, so it reaches exactly who may already drive the robot. That is done in code rather than with `Group=robot` here, the way robotd does it: this process keeps its own primary group, and reaches the robot group as a supplementary one.

tap socket 是 0660 并由 padd 自己交给 `robot` 组。在代码中完成而非用 `Group=robot`——padd 保持自己的主组，通过补充组到达 robot 组。

---

## [Install] 段

```ini
WantedBy=multi-user.target
```

padd 随系统启动自动启用，从 boot 就运行等待手柄连接。

---

## 与其他文件的关系

- **`sysusers.d/padd.conf`**：创建 padd 用户和组
- **`padd` crate**：主程序（main.rs）和 tap（tap.rs）
- **`robotd.service`**：意图目标，padd 通过其 socket 发送命令
- **`bluetooth.service`**：蓝牙服务，手柄通过蓝牙连接
- **`robotctl pad pair`**：配对流程（在 configd 中）
- **`docs/architecture.md`**：架构文档（Documentation 字段指向）
- **`mediad.service`**：类似的安全加固模式，但 mediad 不能用 `PrivateDevices`（需要 /dev/mpp_service 等）和 `MemoryDenyWriteExecute`（GStreamer dlopen）

---

## 关键踩坑点总结

1. **AF_NETLINK 是承重线**：gilrs 通过 libudev 枚举和监视设备，libudev 是 netlink socket。没有 AF_NETLINK，启动后连接的手柄永远不被注意——表现为"手柄只在机器人启动前开着时才工作"。这和蓝牙 73 秒延迟叠加，padd 必须永远轮询。

2. **蓝牙 hci0 73 秒才出现**：在 Radxa Zero 3 上，上电后约 73 秒蓝牙控制器才就绪。`After=bluetooth.service` 远不够——padd 永远轮询手柄，无线电随时出现都能被捡起。

3. **不用 PrivateDevices=yes**：会隐藏 `/dev/input`——这是 padd 存在来读的唯一东西。

4. **MemoryDenyWriteExecute=yes 可以设**：与 mediad 不同，padd 不 dlopen 插件，可以安全禁止 W^X。

5. **padd 必须非 root 运行**：不是加固选择，是进程目的——padd 对机器人没有特权访问，这是意图 API 的诚实练习。

6. **无手柄时什么都不发送**：让 robotd 的 deadman 自己停止机器人。不发零命令（那会把断开的手柄伪装成有意停止）。

7. **连接时不启用策略**：配对手柄只授予按 Start 启用策略的权威。配对本身是门控——需要在机器人旁边保持配对模式的手柄。

8. **tap socket 组分配在代码中而非 unit 文件**：padd 保持自己的主组（padd:padd），通过补充组到达 robot 组。socket 的 chown 在 tap.rs 中用 libc::chown 完成。

9. **RuntimeDirectory 移除时清理 socket**：systemd 在单元停止时移除 /run/padd/，包括 pad.sock。这避免了"padd 没在运行"和客户端挂起之间的区别——没有残留 socket 让客户端以为有人在监听。

10. **Restart=always 而非 on-failure**：padd 在 socket 不存在时干净退出（exit code 0），on-failure 不会重启它。always 确保它在 robotd 慢启动时自动追上。
#（注：内容由AI生成）
