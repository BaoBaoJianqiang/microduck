# reconcile.rs 文件解析

## 文件位置

`d:\microduck\updater\src\reconcile.rs`

## 问题

更新重启发布携带的 unit，然后在回复上线 5 秒后通过 `systemd-run` 重启自身与 `btd`。但**调度成功不代表重启发生**——`systemd-run` 成功仅创建了瞬态定时器。这留下了机器人运行未安装版本的静默可能。

## 解决方案

每次 `updaterd` 启动时，比较每个 unit 运行的二进制与其组件 active 的发布，任何过期的都重启。

## 读取方式

每个 daemon 启动时发布自己的 identity（`duck_ipc_proto::Identity`），所以此处**读文件**而非询问进程。未发布 identity 的 daemon 不视为过期（停止或太旧）。

## 为何启动是正确时机

1. 答案首次可信的时刻——updaterd 无法观察自身重启，检查必须在**后继**中运行
2. 捕获一切：失败的重启、无法启动的新二进制、被停的 unit、回滚遗留的 daemon

## `verdict_for(running, expected, is_self) -> Verdict`

| 情况 | 裁决 |
|---|---|
| `None`（停止） | `Unknown`（停止的 unit 不是过期的） |
| 匹配 | `Current` |
| 不匹配且是自身 | `ReportedOnly`（**updaterd 不得在此重启自身**，否则可能循环） |
| 不匹配 | `Restarted` |

## `check(systemctl, expected, self_unit, units)`

对每个 unit 调用 `running_release`（读 identity 文件），决定裁决，过期的调用 `systemctl restart`（`RestartFailed` 记录但不致命）。

## `stale_units(expected, units)`

只读半部分，供 `Engine::apply` 在发布已安装但 unit 过期时调度重启（因 apply 内不能重启 updaterd/btd）。

## 关键摘要

reconcile.rs 在启动时检查并重启过期 unit（运行版本≠active 版本）：读 daemon 发布的 identity 文件而非询问进程；updaterd 自身只报告不重启（防循环）；停止的 unit 不动；`stale_units` 供 apply 路径调度自身/btd 的重启。
