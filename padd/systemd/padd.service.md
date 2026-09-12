# padd.service 文件解析

**文件位置**：`d:\microduck\padd\systemd\padd.service`

## 核心设计决策

为什么是一个 unit：过去驾驶意味着 ssh 到板、加入两个组（还得登出再登入）、用全路径跑二进制——ssh 会话一断就死。现在配对手柄（`robotctl pad pair`）是唯一步骤，此进程从开机起等待手柄。

无手柄运行是安全的，所以常驻 unit 比 udev 规则或设备-unit 绑定更简单：`padd` 看不到手柄时什么都不发，robotd 的 deadman 自己稳住机器人。手柄连接后一个 tick 内被拾起。

连接时也不动——策略关着直到按 Start。配对的手柄获得启用它的权限，*配对*是门控。

## 关键配置

- `After=robotd.service local-fs.target` + `Wants=robotd.service`：After 不是 Requires——robotd 挂了不应阻止此 unit 启动；socket 不在时 padd 退出，systemd 5 秒后重试。
- `After=bluetooth.service`：hci0 约开机 73 秒后才存在，padd 永久轮询手柄，无线电随时出现即可。
- `User=padd` `Group=padd` `SupplementaryGroups=input robot`：非特权。input 组读 `/dev/input/event*`（root:input 0660），robot 组达 robotd 的 0660 socket。
- `Restart=always` `RestartSec=5s`：padd 在 robotd socket 不在时干净退出，这是应重试的状态。
- 硬化：**NOT `PrivateDevices=yes`**（会隐藏 `/dev/input`）；`RestrictAddressFamilies=AF_UNIX AF_NETLINK`（AF_NETLINK 是 gilrs 通过 libudev 枚举/监视设备的负载项，缺它手柄开机后连接永不被发现）。
- `RuntimeDirectory=padd`：唯一能在 `ProtectSystem=strict` 下创建属主目录的机制；systemd 停止时删除它，避免留下声称在运行的 identity 或无人监听的 socket。
- `Environment=RUST_LOG=info`：padd 在开始驾驶、策略切换、手柄出现/消失时 warn 记录——有人读 journal 找的事件，非每 tick。

## 关键摘要

常驻非特权 unit，靠 systemd 重试与 deadman 实现"配对即驾驶"。硬化中两条负载例外：不禁 PrivateDevices（要读 /dev/input）、保留 AF_NETLINK（gilrs 经 libudev 监视热插拔）。
