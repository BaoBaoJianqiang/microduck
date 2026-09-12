# robot-boot-check.service 文件解析

## 文件位置

`d:\microduck\updater\systemd\robot-boot-check.service`

## 定位

更新路径最后未覆盖的失败：刚启动的发布是否真的起来了？若没有，回退到 golden。

updaterd 无法观察自身替换启动，其启动计数器仅在 updaterd 自身启动时推进——所以守护进程启动不起来的发布会留板上有好发布、armed trial、但无运行进程能行动。

## 触发

仅由 `robot-boot-check.timer` 启动。**故意无 `[Install]` 段**：install.sh 与 postinstall 会 `enable --now` 每个有 Install 的 unit，在此做会在安装它的更新中途运行回滚检查（守护进程正当重启中，被误读为坏发布）。

## [Service]

| 字段 | 值 | 说明 |
|---|---|---|
| `Type` | oneshot | |
| `ExecStart` | `/usr/local/sbin/robot-boot-check` | 不在 `current` 下（恢复路径不能经被恢复的东西） |
| `Restart` | no | 非零退出是脚本 bug |

## `Conflicts=shutdown.target`

关机时守护进程被杀可能记为 failed，无此则 poweroff 时机不对会导致机器人重启进 golden。

## 关键摘要

robot-boot-check.service 是启动后回退检查：由 timer 触发，脚本判断发布是否起来，否则回退 golden；无 Install 段防在安装更新中途误触发；ExecStart 在 /usr/local/sbin 而非 current（恢复路径不经被恢复物）；与 shutdown.target 冲突防关机误触发。
