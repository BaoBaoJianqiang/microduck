# 解析：`journald.conf.d/10-robot.conf`

## 这是什么

systemd-journald 的** drop-in 配置**（51 行，其中 24 行是注释）。文件头注释写明了用法：安装到 `/etc/systemd/journald.conf.d/10-robot.conf`，然后 `systemctl restart systemd-journald` 生效。它回答一个问题：机器人的日志存多久、存多大、断电后还剩什么。

## 逐项解析

### `[Journal]` 段

| 配置 | 值 | 作用与理由（来自文件内注释） |
|---|---|---|
| `Storage=persistent` | 显式持久化 | systemd 默认 `Storage=auto`：只有当 `/var/log/journal` 已存在才落盘，否则日志全在内存（`/run/log/journal`）、重启即丢——而"重启前它说了什么"恰恰是支持排障必问的东西。设为 `persistent` 后行为不再依赖"碰巧有没有别的包建过那个目录"，目录本身也会被创建 |
| `SystemMaxUse=200M` | 磁盘上限 | eMMC 小、机器人一跑几个月，日志必须有顶。200M 配合 20M 单文件 ≈ 保留约十轮轮转，足以跨越多次启动 |
| `SystemMaxFileSize=20M` | 单文件上限 | 与上一条配合，使"崩溃前的日志"可达 |
| `SystemMaxFiles=10` | 最少文件数 | 保证触顶轮转时不会把历史压缩到只剩本次启动 |
| `MaxRetentionSec=3month` | 最长保留 | 放了几个月没动的机器人不该还留着当时的日志 |
| `Compress=yes` | 压缩 | 默认行为，显式写出 |
| `ForwardToSyslog=no` | 不转发 syslog | 本镜像上没有任何东西消费 syslog，转发纯属浪费 CPU 和 IO（顺带也跳过有自己小环形缓冲的 `/dev/kmsg`） |
| `RateLimitIntervalSec=30s` | 限流窗口 | 限流保留但放宽——见下 |
| `RateLimitBurst=10000` | 窗口内配额 | 默认每服务 1000 条/30s 低到会在崩溃循环时丢消息，而那正是每一行都最要紧的时刻。**是调高、不是关掉**：真正失控的服务仍然不能把磁盘填满 |

### 文件头注释里的关键告诫

⚠ **单装这个文件并不保险**——如果 `/var/log` 本身是 tmpfs。Armbian 镜像自带 RAM 日志机制（`armbian-ramlog` / `log2ram`），会把 `/var/log` 挂到内存、定期和在干净关机时同步回磁盘。在那种机制下，journald *以为*自己在持久化，而非正常断电仍会丢最近的日志——机器人恰恰经常非正常断电。板上的验证方法（文件注释给出）：

```bash
findmnt /var/log                  # 显示 tmpfs 即日志在内存
systemctl status armbian-ramlog   # 或 log2ram
```

按注释的说法，这是一个**尚未收口的板上验证项**（`docs/roadmap.md` 的 M4），处理办法见 `deploy/README.md`——README 已实测本镜像 `/var/log` 是 zram 设备，并决定接受"服务日志尽力而为、以 `/var/lib` 下逐条 fsync 的更新历史作为持久记录"这一安排。
