# 启动恢复网

当机器人启动的 release 无法启动其 daemon 时，什么把它放回一个已知良好的 release。

本页面负责启动截止时间及其谓词、它监视的 units 的成员规则、`robot-rescue`、`golden` 符号链接，以及循环守卫。[`restart-order.md`](restart-order.md) 负责它所处的序列——包括 `updaterd` 在启动时何处读取面包屑——而 [`updater-design.md`](updater-design.md) 负责存储、健康门控和启动计数器。

## 它填补的缺口

`updaterd` 在更新期间从不重启自己，因为它是执行更新的进程。三层缩小了这一代价，且三层都在板子完全启动时运行：

- `self_test_updaterd` 在提交前运行 release 自己的 `updaterd --self-test`，所以错误架构的二进制、缺失的库或被拒绝的 `updater.toml` 会在旧 release 仍活动时使更新失败；
- 延迟重启在更新回复后五秒把 `updaterd` 和 `btd` 放到新二进制上；
- 启动协调从后继者侧检查那些重启是否发生。

一个在运行中的板子上能启动的二进制仍可能在**启动时**失败，那时网络还没起来、外设还没枚举、unit 顺序也不同。而协调在*`updaterd` 启动时*运行，所以它预设了一个能启动的 `updaterd`。

最后一点正是这一切存在的原因。`BootCounter::record_boot` 从 `updaterd` 内部的 `Engine::recover_on_start` 运行，计数 `updaterd` **启动次数**而非字面意义的启动——所以一个不启动的 `updaterd` 不被计数，永不变砖的保证通过那个可能成为伤亡者的进程关闭。留在那里，板子磁盘上有一个好 release、有一个已武装的 trial 要回退到它，但没有任何运行中的东西能对两者采取行动。

启动计数器处理*不同*的失败：一个能启动并运行但从不报告健康的 release。这里处理的是 daemon 根本起不来的 release。

## 组成部分

| | |
|---|---|
| `robot-boot-check.timer` | fires 180 s into every boot |
| `robot-boot-check.service` | the oneshot: asks whether the release came up, hands over if not |
| `scripts/robot-boot-check` | the predicate. Installed at `/usr/local/sbin/robot-boot-check` |
| `scripts/robot-rescue` | swaps `current` to golden, records it, reboots. `/usr/local/sbin/robot-rescue` |
| `<install_dir>/golden` | symlink to the release with a standing guarantee, published by `updaterd` |
| `<state_dir>/rescued` | breadcrumb: the last attempt, and the loop guard |

两个脚本都是 `/bin/sh`，且都由 `scripts/install.sh` 和 `hooks/postinstall` **复制**到 `/usr/local/sbin`——与那两个复制而非通过 `current` 读取 unit 文件是同一个决定、同一个理由。作为 release payload 中二进制发货的救援会被它存在所要应对的那些 release 破坏：错误架构、缺失共享库、启动时 panic。通过 `current` 读取的救援会把恢复路由经过正在被恢复的东西。

## 触发器：截止时间，而非 `OnFailure=`

`OnFailure=` 看起来是自然之选——systemd 恰好在 unit 耗尽重启次数进入 `failed` 时触发它，无需轮询、无常驻看门狗。它不适合*这些* units，它们被配置为不进入那个状态：

| unit | policy | `RestartSec` |
|---|---|---|
| `btd` | `Restart=always` | 5 s |
| `padd` | `Restart=always` | 5 s |
| `configd` | `Restart=always` | 2 s |
| `robotd` | `Restart=always` | 2 s |
| `updaterd` | `Restart=on-failure` | — |

每种情况下 `Restart=always` 都是刻意的，且在 unit 文件中有论证。没有 unit 覆盖 `StartLimitIntervalSec`/`StartLimitBurst`，所以 systemd 默认的十秒内五次启动适用——而每 5 秒重启一次的 `btd` 永远碰不到它。它无限崩溃循环却从不进入 `failed`，那正是 `OnFailure=` 等待的状态。让 `OnFailure=` 触发意味着重新调整那些 unit 让 daemon *放弃*：为启用救援场景而降级正常场景。

截止时间把问题反过来，无需改动 unit，且能捕获永远重启者。`OnBootSec=180` 因为板子比看起来慢——`hci0` 在上电约 73 秒后才存在，其中 `bluetooth.service` 有 26 秒阻塞在 `dbus` 后面——且 daemon 等待硬件而非退出，所以慢启动不是失败启动。

### 谓词

两个条件，都是 systemd 试图运行 daemon 却无法保持其运行的正面证据。任一成员满足其一即可：

- `ActiveState` 为 `failed`；
- `NRestarts` 为 3 或更多。

其他一切都不碰，不对称正是关键。操作员停止的 unit 是 `inactive` 且无重启，绝不能让板子丢掉 release——与启动协调在让停止的 unit 保持停止时做的判断相同。release 早于它的 unit 未加载且不应答，读起来一样。仍在等硬件的 daemon 是 `active`。而恢复过来的一次崩溃留下一两次重启，不是三次。**假阴性让操作员丢一次诊断；假阳性让一个好 release 被丢。**

三次重启而非一次，因为 `Restart=always` 加 2–5 秒 `RestartSec` 意味着真正的循环在截止时间内轻松超过三次，而瞬态死一次的 daemon 不会。

### 它拒绝触发的三种方式

- oneshot 上的 **`Conflicts=shutdown.target`**。在关机途中被杀死的 daemon 可能被记录为 `failed`；没有这个，在错误时刻 `systemctl poweroff` 最终会把机器人重启进 golden。
- **运行时间守卫。** 超过十分钟，说明是别的东西启动了它，答案是这个问题不归这个脚本管。
- oneshot 上**无 `[Install]` 段**，两个安装器都把这读作"不要启用这个"。没有它，`hooks/postinstall` 对刚写的 units 运行 `enable --now` 会在安装它的更新中途跑回滚检查——此时 daemon 正处于合法的重启中途，那正是回滚检查读作 release 无法启动的情况。Timer 被启用但从不启动也是同理：过了截止时间才启动的 `OnBootSec=` timer 会立即触发。

## 成员规则

对"任何 daemon 失败 → 回滚"的反对是，daemon 可能因回滚无法修复的原因而失败——缺失无线电、未上电的电机总线——而为硬件故障回退一个好 release 比什么都不做更糟。

这消解为一条接纳 unit 进入集合的规则：**unit 只有在它等待硬件（缺失或行为不端）而非退出时才可加入。** 那么一个宕掉的成员意味着*二进制*坏了——错误架构、缺失库、启动 panic、它拒绝的配置——这正是回滚能修复的。

| daemon | in the set | when its dependency is absent |
|---|---|---|
| `robotd` | yes | waits for the motor bus, "waiting, not commanding anything" |
| `updaterd` | yes | serves regardless; it is the recovery path and must be reachable when things are broken |
| `btd` | yes | retries for an adapter every 5 s, and re-runs the whole bring-up if one appears and then stops answering |
| `configd` | yes | serves `system.*` and `pad.*`, and reports `net.state=unavailable` |
| `padd` | no | exits cleanly with no robotd socket, by design |

`btd` 凭实力而非对称获得位置：没有网络时 BLE 是唯一入口，所以一个启动时没有 `btd` 且没有已知 wifi 的机器人完全无法联系。`configd` 同理获得位置——`system.pin` 是 `btd` 获取手机用来认证的 PIN 的地方，所以宕掉的 `configd` 会带走手机路径。

如果将来有 daemon 因缺失硬件而退出，要么修复 daemon 要么把它排除在集合外。不要削弱规则。

## 救援

`robot-rescue` 读两个符号链接，什么都不解析。

这不是节省。它最可能救援的是一个 `updaterd` 拒绝板子 `updater.toml` 的 release——操作员的文件，跨安装保留，而 release 可以自由改变它期望的东西。一个解析那个同一文件的救援会死于它正在治疗的疾病。

Golden 是那个文件里的一个版本（`ComponentConfig::golden`），所以 `Engine::refresh_golden_links` 在每次 `updaterd` 启动时把它作为 `golden` 符号链接发布在 `current` 旁边：一次 `readlink`，没有可拒绝的东西。是缓存而非第二个真相来源，且它安全失败——如果 `updaterd` 停止启动，链接仍指向上次写入时是 golden 的 release，那是一个已知良好的 release。一个已配置但*未安装*的 golden 不发布任何链接，因为悬空链接会告诉救援它有目标而实际上没有。

**Golden，不是 previous：** 当坏掉的正是恢复路径本身时，previous 可能也坏了。Golden 是唯一有持续保证的 release，且 `prune` 从不删除它。

当没有 golden 链接、golden 指向一个未安装的 release、以及 `current` 已经是 golden 时，它拒绝并说明是哪种。最后一种最重要：此时 daemon 在承载持续保证的 release 上失败，所以这不是 release 故障，交换除了增加一次重启外什么都不会改变——这正是硬件故障变成重启循环的方式。

交换通过把临时符号链接 rename 到 `current` 上完成，匹配 `Store::link_to`，因为 `ln -sfn` 会先 unlink 再重建，留下一个 daemon 无法 exec 的窗口。`rename(2)` 不跟随符号链接但 `mv` 的路径解析会，所以普通的 `mv tmp current` 会把链接*移进* `releases/<version>` 而 `current` 不动；GNU 的修复是 `-T`，BSD 是 `-h`。

`--reboot` 是可选的。每个 unit 通过 `current` exec，所以交换在它们重启前什么都不做，而一个 daemon 死掉时还站着的机器人是一个会摔倒的机器人——由扶着它的人决定。启动检查传递该标志；控制台前的人则得到打印出的命令。

## 循环守卫，以及什么打开它

两个独立的东西阻止"交换并重启"自行发生两次：

1. 当 `current` 已是 golden 时拒绝，覆盖普通情况，因为成功的救援恰好留下那个状态；
2. 面包屑，覆盖第一个漏掉的情况——一次更新把 `current` 从 golden 移走又再次失败。

`Engine::record_rescue` 在下一次 `updaterd` 启动时清除面包屑，在把它复制进更新日志之后。所以守卫恰好在板子证明它又能运行其更新 daemon 时打开，在不能时保持关闭：一个静静坐着、journal 干净的机器人胜过一个永远重启的机器人。`robot-rescue --force` 是手动越过它的方式。

那个清除在**启动计数器之前**运行，顺序是关键的。已武装的 trial 会回退到 `previous`；救援已经走得更远，到了 golden。如果保持武装，`record_boot` 会推进 trial 并最终把 `current` 移*离*救援选择的 release——恢复网和永不变砖保证互相抵消，相隔一次启动。

面包屑是 `key=value` 而非 JSON，尽管 `updaterd` 解析它：

```text
at=1786453421
install_dir=/opt/robot/daemon
from=1.2.0
to=1.0.0
because=boot check: robotd.service (failed, 7 restarts)
```

写入者是一个在板子上事情已经出错时运行的 shell 脚本，在 `sh` 中正确引用 JSON 是产生一个无人能读的记录的方法。`journal::Breadcrumb` 出于同样原因很宽松——每个字段可选——因为无法解析的面包屑是一个永远打不开的守卫。`install_dir` 是把条目归属于某个组件的方式，而无需第二个地方必须在名称 `daemon` 上与 `updater.toml` 一致。

## 证据

静默回滚是一天变成"在我板子上能用"的方式，所以交换留下三条痕迹：

- **journal**，来自两个脚本，有 `logger` 时通过它，总是通过 stderr；
- **更新日志**，作为 `Outcome::RolledBack`，原因中点名失败的 unit——这是 `robotctl update log` 显示的，且它是永久的，不像面包屑会被清除；
- **`robotctl version`**，间接但有用：运行的 release 比已安装的旧正是它报告的情况。

## 什么未被测试

决策已测：`xtask/tests/rescue.rs` 对两个脚本针对临时树问每个问题，用 stub `systemctl` 提供 unit 状态并记录是否请求了重启，而 `scripts/board-test.sh` 在真实 ARM64 Linux 上覆盖安装器那一半。

**systemd 没测。** timer 是否在 `OnBootSec=180` 触发、`Conflicts=shutdown.target` 是否在关机途中让 oneshot 不运行、以及 `NRestarts` 在崩溃循环 unit 上是否按谓词假设的方式读取，全都未验证。除了真实 systemd——`systemd-nspawn`，或以它为 pid 1 的特权容器——没有任何东西能回答其中任何一个，[`install-path-gap.md`](../project/install-path-gap.md) 是论证这个的地方。这是第二个论据：恢复网是必须在其他一切都不工作时工作的代码，也是最不适合测试的代码。

在那之前，这个机制要么在实验板上验证，要么根本不验证，方法是给板子一个 `robotd` 无法启动的 release。

这一切都没在硬件上运行过，那里有趣的路径需要一个 release 无法启动的板子。
