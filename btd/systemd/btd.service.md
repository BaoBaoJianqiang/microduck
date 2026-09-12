# `btd.service` 文件解析 —— systemd 服务单元

## 1. 文件定位

- 路径：`systemd/btd.service`
- 安装位置：`/etc/systemd/system/btd.service`
- 角色：定义 btd 守护进程如何被 systemd 启动、排序、降权与沙箱化。
- 文件头注释强调两条源自"btd 属于恢复路径"（docs/architecture.md §1.1）的、必须保持的属性：
  1. **对 robotd / updaterd 零依赖**（任一方向都没有）：机器人不工作时 btd 必须仍能应答——这是它存在的主要理由。它经 socket 联系二者，socket 缺失是被正常上报的应答，而非不启动的理由；
  2. **保持非特权**：btd 解析无线电范围内任何人发来的字节，以 root 运行的是 configd。把解析器放在该边界安全一侧比加固分发器更重要。

## 2. `[Unit]` 段

- `Description=Robot BLE transport adapter`；
- `Documentation=file:///opt/robot/daemon/current/docs/architecture.md`；
- `After=dbus.service bluetooth.service` 且 `Wants=bluetooth.service`：
  - bluetoothd 是硬需求——btd 经 D-Bus 与 BlueZ 交谈，没有它无事可做；
  - 用 `Wants` 而非 `Requires`：若 bluetoothd 缓慢或重启，btd 等待并重试适配器而不是失败（对应 `src/bluez.rs` 的 `ADAPTER_RETRY`）；
  - Radxa 上 hci0 在上电约 73s 后才出现（aic-bluetooth.service 很晚才挂接 AIC8800 UART，bluetooth.service 自身又被 dbus 阻塞约 26s），所以排在 bluetooth.service 之后"必要但远远不够"，真正让启动可用的是重试循环；
- `After=local-fs.target`：**无网络依赖**——BLE 本就必须在网络不可用时工作。

## 3. `[Service]` 段

### 3.1 启动

- `Type=exec`；
- `ExecStart=/opt/robot/daemon/current/bin/btd`。

### 3.2 身份与组（非特权）

```ini
User=btd
Group=btd
SupplementaryGroups=robot bluetooth
```

- 不像 robotd 那样用 root：btd 不直接碰硬件，只用 D-Bus 和几个 unix socket；
- 附加组 `robot`：使其能通过 updaterd、robotd 的 `0660` socket；
- 附加组 `bluetooth`：*应当*能通过 BlueZ 的 D-Bus 策略——但注释明确要求"上板验证"：Debian 的 `bluetooth.conf` 把 `send_destination="org.bluez"` 授予 root 与 `at_console`，组能否覆盖一个无会话守护进程随镜像而异。若启动报 D-Bus 权限错误，正解是增加一条授予该用户访问 org.bluez 的 drop-in 策略，**而不是以 root 运行**；
- 用户与组必须预先存在，否则单元启动失败（见 `sysusers.d/btd.conf`）。

### 3.3 重启策略

- `Restart=always`、`RestartSec=5s`；
- 与代码内的就地重试互补：无线电故障在进程内部自愈，进程级异常由 systemd 5s 后拉起。

### 3.4 沙箱加固

比 robotd 能收得更紧，因为本进程没有硬件要访问，只需要 D-Bus（一个 unix socket）和另外几个 unix socket：

```ini
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
RestrictAddressFamilies=AF_UNIX
RestrictNamespaces=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
CapabilityBoundingSet=
```

要点：文件系统严格只读、隐藏/home、私有 /tmp 与设备、禁止改内核参数/模块/cgroup、禁 suid、地址族仅限 AF_UNIX（无网络套接字）、禁命名空间、锁定 personality、禁可写可执行内存、系统调用过滤为 @system-service（违规返回 EPERM）、能力边界集合清空（无任何 capability）。

### 3.5 日志

- `Environment=RUST_LOG=info`，stderr/stdout 进 journal，级别可由 `RUST_LOG` 调整；
- 启动身份行（版本、revision、可执行路径）以 `warn` 级别记录，因此在长期运行的板卡上即使 `RUST_LOG=warn` 也会保留。

### 3.6 运行时身份文件

```ini
RuntimeDirectory=btd
```

- 守护进程把"正在运行什么"发布到 `/run/btd/identity.json`，供 `robotctl health` 与 updaterd 的启动检查读取；
- 用 `RuntimeDirectory=` 而非在单元里授予某条路径：它是唯一既能创建由本单元 `User=` 拥有的目录、又能在 `ProtectSystem=strict`（全盘只读）下成立的机制；
- systemd 在单元停止时自动删除该目录，因此已停止的守护进程不可能留下一份声称自己在运行的身份文件。

## 4. `[Install]` 段

- `WantedBy=multi-user.target`：随常规多用户目标启用。

## 5. 本文件要点小结

1. btd 是恢复路径的一环：不依赖 robotd/updaterd/网络，蓝牙缺失则等待重试而非失败；
2. 以专用非特权用户 `btd` 运行，仅靠附加组获得 socket 与（待上板验证的）D-Bus 访问；
3. `Restart=always` + 5s 与代码内重试构成两级自愈；
4. 沙箱收紧到只剩 AF_UNIX，无能力、无设备、只读文件系统；
5. 日志进 journal，`RUST_LOG` 可控；`RuntimeDirectory=btd` 安全地承载身份文件并随停随删。
