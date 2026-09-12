# robot-boot-check

## 文件位置

`d:\microduck\scripts\robot-boot-check`

## 核心设计决策

该脚本检查刚启动的版本是否真正起来了，若没有则交给 `robot-rescue`。由 `robot-boot-check.timer` 在启动 3 分钟后运行一次。

- **"没起来"的两个判定条件**（都是 systemd 试图运行 daemon 却无法保持其运行的正面证据）：
  1. unit 状态为 `failed`；
  2. unit 本次启动已重启至少 3 次（无论当前在做什么）。
- **不对称设计**：其他一切都不动。操作员手动停止的 unit 是 `inactive` 无重启，不能让板端丢版本；版本早于的 unit 未加载，读取结果相同；等待硬件的 daemon 是 `active`；单次崩溃恢复留下一两次重启而非三次。假阴性代价是操作员一次诊断，假阳性代价是一个好版本。
- **为什么是这四个成员**：`updaterd.service robotd.service configd.service btd.service`。unit 只有在等待硬件而非退出时才能加入，这样 unit 宕机意味着二进制坏了——这正是 rollback 能修的。`padd` 被排除，因为它在 robotd socket 缺失时按设计干净退出。`btd` 被包含是因为无网络时 BLE 是唯一入口，无 `btd` 且无已知 wifi 的机器人无法被触及。
- **MAX_RESTARTS=3 而非 1**：`Restart=always` 加 2-5s `RestartSec` 意味着真正的循环在期限内会多次超过三次，而因瞬时故障死亡一次又恢复的 daemon 不会。1 会误触发后者。
- **MAX_UPTIME=600（10 分钟）**：这是启动检查。启动后很久才被调用意味着不是 timer 启动的（手动 `systemctl start` 或安装器 enable unit）。无此守卫，`hooks/postinstall` 运行 `enable --now` 会在更新中途运行 rollback 检查。

## 常量/参数分析

| 常量/变量 | 值 | 说明 |
|---|---|---|
| `MEMBERS` | `updaterd.service robotd.service configd.service btd.service` | 检查的 unit 列表 |
| `MAX_RESTARTS` | `3` | 重启次数阈值 |
| `MAX_UPTIME` | `600` | 启动后秒数上限（10 分钟），超过则拒绝 |
| `RESCUE` | `${ROBOT_RESCUE:-/usr/local/sbin/robot-rescue}` | 救援脚本路径 |
| `UPTIME_FILE` | `${ROBOT_UPTIME_FILE:-/proc/uptime}` | uptime 读取源 |

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--dry-run` | 报告判定，不调用 robot-rescue |

### 退出码

- 0：无事可做，或已交给救援
- 2：拒绝
- 1：出错

## 核心逻辑

1. **uptime 守卫**：读取 `/proc/uptime` 第一字段，若超过 `MAX_UPTIME` 则拒绝（exit 2）。无文件时不强制守卫。
2. **逐个检查成员**：对每个 unit 调用 `property()` 获取 `ActiveState` 和 `NRestarts`。
   - `property()` 每次只查一个属性，因为 `systemctl show -p A -p B` 按 systemd 自己的顺序打印，顺序变化会导致两行解析静默出错。
   - `restarts` 非数字时归零（未加载的 unit 两个属性都返回空字符串）。
3. **判定失败**：`state == failed` 或 `restarts >= MAX_RESTARTS`。
4. **无失败**：exit 0。
5. **dry-run**：报告并 exit 0。
6. **救援脚本不存在**：exit 1。
7. **调用救援**：`"$RESCUE" --reboot --because "boot check: ${failed}"`。救援 exit 2（拒绝）时此脚本 exit 0（问题已被问答），否则透传退出码。

## 关键要点总结

1. 由 `robot-boot-check.timer` 在启动 3 分钟后运行，设计文档见 `docs/design/boot-recovery-net.md`。
2. 四个成员都等待硬件而非退出，使 unit 宕机等于二进制损坏。
3. 重启阈值 3 次平衡了真实崩溃循环和瞬时故障恢复。
4. uptime 守卫防止更新中途误触发。
5. 将所有决策权交给 `robot-rescue`，包括 golden 是否存在、current 是否已为 golden、前次尝试是否已试过。
