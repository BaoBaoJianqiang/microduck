# robot-rescue

## 文件位置

`d:\microduck\scripts\robot-rescue`

## 核心设计决策

该脚本将板端切回 golden 版本，不经过 update daemon。

- **为什么需要独立救援脚本**：`robotctl update rollback` 和 `reset-to-golden` 是对 `updaterd` 的 IPC 调用。除了最关键的一种失败外都没问题——某个版本的 `updaterd` 无法启动。此时恢复能力恰好在无法启动的进程内部，`Restart=` 放弃后，磁盘上有完好旧版本的板端却无路可退，连控制台前的人也不行。此脚本就是那条退路。
- **为什么是 shell 脚本且从 release 中拷出**：作为制品中二进制发布的救援会被它要应对的版本破坏——错误架构、缺少共享库、启动 panic。`install.sh` 和 `hooks/postinstall` 将其拷到 `/usr/local/sbin`，与拷贝 unit 文件同理——运行的副本在 `current` 移动时不能变。
- **读符号链接不解析任何文件**：最可能救援的是 `updaterd` 拒绝板端 `updater.toml` 的版本（操作员文件，跨安装保留，而版本可自由更改期望）。解析同一文件的救援会死于它要治的病。`updaterd` 每次启动都将 golden 发布为符号链接（`Engine::refresh_golden_links`），答案只需一次 `readlink`。
- **循环守卫**：`robot-boot-check` 无人值守调用此脚本，"交换并重启"必须不能自行发生两次。两道独立防线：
  1. `current` 已是 golden 时拒绝（成功救援后即为此状态）。
  2. 前次尝试的 breadcrumb 仍在时拒绝（覆盖第一道漏掉的情况：更新又把 `current` 移离 golden 并再次失败）。
  breadcrumb 由 `updaterd` 下次启动时清除（`Engine::record_rescue`），守卫恰在板端证明能再次运行 update daemon 时释放。

## 常量/参数分析

### 环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `ROBOT_INSTALL_DIR` | `/opt/robot/daemon` | 安装目录 |
| `ROBOT_STATE_DIR` | `/var/lib/robot/updater` | 状态目录（breadcrumb 所在） |

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--dry-run` | 报告决策，不做任何更改 |
| `--reboot` | 交换后重启（默认不重启，因为所有 unit 通过 current exec，重启前不生效；站着的机器人 daemon 死亡会摔倒） |
| `--force` | 即使有前次 breadcrumb 也执行 |
| `--because <text>` | 原因，记录在 breadcrumb 和更新日志中，默认 `invoked by hand` |

### 退出码

- 0：已交换（或本应交换）
- 2：拒绝，无事可做
- 1：出错

### 关键路径

| 路径 | 说明 |
|---|---|
| `golden_link` | `${INSTALL_DIR}/golden` |
| `current_link` | `${INSTALL_DIR}/current` |
| `breadcrumb` | `${STATE_DIR}/rescued` |

## 核心逻辑

1. **循环守卫检查**：breadcrumb 存在且无 `--force` → 拒绝（updaterd 自上次尝试后未启动，golden 没修好）。
2. **无 golden 检查**：`golden_link` 不是符号链接 → 拒绝。
3. **golden 目标不存在** → 拒绝。
4. **current 已是 golden** → 拒绝（rollback 无法修复，可能是硬件故障）。
5. **dry-run**：报告并退出。
6. **写 breadcrumb**：`key=value` 格式（不是 JSON，因为 sh 中正确引用 JSON 易产生无人能读的记录；也是先给人看的证据），在交换前写入（与引擎启动计数器顺序一致：未发生的尝试记录可恢复，未记录的交换不可恢复）。
7. **原子交换**：不用 `ln -sfn`（有 current 不存在的窗口，daemon 在此窗口启动会 exec 失败），而是创建临时符号链接后用 `rename(2)` 原子覆盖。GNU `mv -fT` / BSD `mv -fh` 兼容。
8. **重启**（若 `--reboot`）：`systemctl reboot`。

## 关键要点总结

1. 用 `rename(2)` 原子交换符号链接，避免 `ln -sfn` 的窗口风险。
2. breadcrumb 用 `key=value` 而非 JSON，便于 shell 写入和人阅读。
3. `--reboot` 是 opt-in，默认不重启，因为站着的机器人 daemon 死亡会摔倒。
4. 循环守卫确保不会无限重启：只有 updaterd 成功启动后才清除 breadcrumb。
5. 所有决策都基于符号链接的 `readlink`，不解析 `updater.toml`。
