# 什么重启，以及何时

每条移动 `current` 符号链接的路径以及启动的权威序列。写它是因为"哪个 daemon 重启了，在什么时刻"曾从三份不同文档得到三种不同答案，而答案决定了如何诊断不一致。

这里的一切都从代码读取，每一步都点名拥有它的函数。如果别处的叙述性注释与本页不一致，注释是 bug。

## 1. 七个 daemon，以及不是 daemon 的那个 unit

一个 release 发货七个 daemon——`robotd`、`configd`、`btd`、`padd`、`mediad`、`tofd`、`updaterd`——每个都 `ExecStart` 一个 `/opt/robot/daemon/current/bin/` 下的路径，所以每个在符号链接移动的瞬间就过时了，每个要么在更新中重启，要么在更新后重启。

它还发货 `robot-boot-check.service`，它**不是**其中之一，且更新绝不能重启它：它询问启动的 release 是否起来了，如果没有就交给 `robot-rescue`，所以在更新中途运行它会把回滚检查指向正处于合法重启中途的 daemon。它没有 `[Install]` 段，这正是让它不被重启的原因——见 §1.1。

| unit | restarted mid-update | restarted ~5 s after the reply |
|---|---|---|
| `robotd` | yes | — |
| `configd` | yes | — |
| `padd` | yes | — |
| `mediad` | yes | — |
| `tofd` | yes | — |
| `updaterd` | **never** — it is the process performing the update | yes |
| `btd` | **never** — it may be the transport the update arrived over | yes |

两半是一个决定，一个测试（`every_unit_held_back_is_restarted_once_the_answer_is_out`）断言 `engine.rs` 中的两个列表是同一个集合：`NEVER_RESTART` 和 `RESTART_AFTER_REPLYING`。

没有任何东西被推迟到重启。在 `RESTART_AFTER_REPLYING` 存在之前就是如此，这也是关于这个系统最常见的过时说法。

而因为调度一次重启不是它发生了的证据，下一次 `updaterd` 启动会对照活动 release 检查每个 unit 并重启任何过时的——§5。

### 1.1 什么算作更新启动的 unit

一个有 `[Install]` 段的 `systemd/*.service` 文件。没有的由其他东西触发——timer，或另一个 unit 拉入——所以它的生命周期不归更新驱动，`engine.rs::has_install_section` 是决定之处。

用规则而非名字列表，因为已经有两个地方在应用它且它们不一致：`hooks/postinstall` 拒绝对无 `[Install]` 的 unit 执行 `enable --now`，原话是——*"对它执行 `enable --now` 会在安装它的更新中途跑回滚检查，此时 daemon 正处于合法的重启中途"*——而引擎读取每个 `*.service` 并在片刻后仍重启它。以段为键，两者通过构造一致，下一个类似的 unit 不需要任何人记得。

不可读的 unit 文件算作要重启的。那更可能是权限问题而非刻意无触发的 unit，且重启大声失败胜过悄悄跳过。

### 集合如何计算

两个集合，差一个函数。`engine.rs::units_shipped` 是 release 提供的一切：

```
(the release's own systemd/*.service stems)  ∪  (on_apply.units from /etc/robot/updater.toml)
```

排序去重。`engine.rs::units_to_restart` 是它减去 `NEVER_RESTART`：

```
units_shipped(…)  −  {updaterd, btd}
```

更新重启 `units_to_restart`；§5 的启动检查读 `units_shipped`，因为更新不能碰的两个 unit 正是它存在要监视的两个。

在今天的 release 上 `units_to_restart` 恰好是：

```
configd, mediad, padd, robotd, tofd
```

按这个顺序——字母序，所以顺序在每块板和每个测试中都相同。没有向 `deploy/updater.toml` 添加任何东西来把 `mediad` 和 `tofd` 放进那个列表，也不需要：两者都发货带 `[Install]` 段的 unit，这就是全部规则。

从 release 而非板子推导它的两个后果：

- `padd` 被重启即使 `deploy/updater.toml` 从未提到它。配置列表是**附加的**，非权威；它只需要点名 release *不*发货的 unit。
- 一个 `updater.toml` 早于某个 daemon 的板子仍重启该 daemon。那个文件属于操作员且 `install.sh` 保留它，这就是 `configd` 曾一段时间未被重启的原因（`../project/install-path-gap.md` §4）。

每个 unit 由自己的 `systemctl restart <unit>` 调用重启，从不批量——单个 `systemctl restart a b` 在任一 unit 未知时整体失败，*不*重启存在的那个。systemd 报告为 `LoadState=not-found` 的 unit 被跳过并警告；存在但重启失败的 unit **使更新失败并回滚**。

## 2. `robotctl update apply` / BLE 上的 `update.apply`

`engine.rs::apply` → `apply_inner` → `stage_and_swap` → `post_swap`。第 8 步之前什么都不碰活动路径。

| # | step | restarts anything? |
|---|---|---|
| 1 | preflight: clock, robot stopped, no live session | no |
| 2 | fetch manifest, verify its signature, channel / pin / downgrade / compatibility checks | no |
| 3 | preflight again, now for disk space (the requirement comes from the manifest) | no |
| 4 | download to `releases/.staging-<ver>/dl/` | no |
| 5 | verify sha256, then verify the artifact signature | no |
| 6 | extract to `releases/.staging-<ver>/root/`, write `.updater-manifest.json` | no |
| 7 | **`hooks/preinstall`** (cwd = the staged tree) — installs ONNX Runtime if below the floor, then runs the release's `scripts/setup-gstreamer.sh` for `mediad`'s stack | no |

**钩子脚本来自传入的 release；运行它的代码不是。** `updaterd` 从不在更新中途重启自己（§1），所以本表每一步都由已经在运行的 `updaterd`——*前一个* release 的——执行。因此对钩子运行、日志或限制方式的变更会在一次 apply 之后、即 `updaterd` 重启到携带它的 release 之后的第一次 apply 才生效。这曾导致一次困惑的"钩子跑了但什么都没记"；`cat /run/updaterd/identity.json` 告诉你哪个 build 即将运行你的钩子。

| 8 | `rename` the staged tree to `releases/<ver>/` | no |
| 9 | arm the boot counter (`pending.json`), *before* the swap | no |
| 10 | swap `current` → `releases/<ver>` | no |
| 11 | **`hooks/postinstall`** (cwd = `releases/<ver>`) | **starts newly shipped units** |
| 12 | `on_apply` — `systemctl restart` each unit from §1, one at a time | **configd, mediad, padd, robotd, tofd** |
| 13 | `releases/<ver>/bin/updaterd --self-test` | no |
| 14 | health gate: poll `robotd` over its socket, every 500 ms, up to 30 s | no |
| 15 | confirm the boot counter, prune old releases | no |
| 16 | release the update lock, schedule the deferred restarts | **schedules updaterd, btd** |
| 17 | the reply goes out to `robotctl` / the app | — |
| 18 | ~5 s after step 16 | **updaterd, btd** |

第 11 步起的任何失败都会回滚：把 `current` 交换回去、对前一个 release 重跑 `on_apply`、确认启动计数器、记入 journal。第 1–10 步失败时旧 release 仍活动，无需撤销。

**第 2 步也可能结束它，且不总是不重启任何东西。** 如果 manifest 点名已安装的 release，操作在此停止并报告 `already_current`——不下载任何东西，不交换任何东西。回复前它对该 release 的 units 做 §5 的读取，并在 `stale` 中点名运行其他东西的那些，那些被精确地按第 16–18 步调度延迟对的方式调度。所以对一个最新的机器人 `apply` 是惰性的，而对 daemon 与已安装内容不一致的机器人 `apply` 不是。对板子已经活动的 release 执行 `select` 也一样，同理。

### 第 11 步详解 —— `hooks/postinstall`

钩子发货在签名 artifact 内，以 release 目录为 cwd 运行。按顺序：

1. 把每个 `systemd/sysusers.d/*.conf` `install` 到 `/usr/lib/sysusers.d/`，然后运行 `systemd-sysusers`。账户先于 unit，因为点名缺失 `User=` 的 unit 启动失败并读作坏 daemon。
2. 把每个 `systemd/*.service` `install` 到 `/etc/systemd/system/`，覆盖。**全部**，包括 `updaterd.service` 和 `btd.service`——钩子没有排除列表，也不需要。
3. `systemctl daemon-reload`。
4. 对每个有 `[Install]` 段的 unit（§1.1）执行 `systemctl enable --now`；没有的安装后不管，`.timer` 启用但不加 `--now`。

第 4 步是 §1 中排除项仍成立的原因：`--now` 意味着*启动*，启动已运行的 unit 是空操作。它不重启 `updaterd` 或 `btd`。它做的是启动板子从未有过的 unit——这正是钩子的全部意义，且意味着**新引入的 daemon 在引入它的 release 上启动两次**：一次在这里，然后在第 12 步再一次。

钩子内的每个失败都是警告，除了 unit 或 sysusers 文件的 `install` 失败，它非零退出并因此回滚更新。启动不了的服务刻意不致命：`btd` 在无线电尚未出现的板子上合法失败。

### 第 13 步 —— 自检

`updaterd` 不在更新期间重启自己，所以一个无法启动的替换二进制否则会在下一次启动、提交之后才被发现，而恢复住在正在失败的进程内部。取而代之的是用 `--self-test` 运行*新的* `bin/updaterd`：它加载 `/etc/robot/updater.toml`、构造引擎、在触碰任何状态之前退出。10 秒上限。

失败是携带二进制最后一行 stderr 的 `Error::SelfTest`，并回滚。没有 `bin/updaterd` 的 release（模型组件）平凡通过。除非 `on_apply` 是重启否则完全跳过，这让它不妨碍引导安装。

### 第 16–18 步 —— 延迟重启

`engine.rs::schedule_deferred_restarts`，对 `updaterd` 然后 `btd`：

```
systemd-run --on-active=5s --timer-property=AccuracySec=100ms -- systemctl restart <unit>
```

四个关键细节：

- **瞬态 unit，不是子进程。** `systemctl restart updaterd` 杀死 `updaterd` 的整个 cgroup，而 `updaterd` 的子进程*在*那个 cgroup 里——它会在重启自己父进程的中途被杀死。`systemd-run` timer 住在它外面。
- **先释放更新锁**（第 16 步，在派生之前）。fork 复制每个打开的描述符，所以持锁派生会把副本交给子进程。
- **在 `Applied` 时，以及在有过时 unit 的 `already_current` 时**——`engine.rs::restarts_owed` 决定是哪种，且它是唯一决定的。前者欠固定的一对；后者欠 §2 发现运行错误 release 的任何东西，通常是 `updaterd` 自己。回滚欠什么都不欠：它让驻留的 `updaterd` 已经匹配 `current`，所以在那里重启是无所修复的折腾。
- **每个 unit 都经过 timer**，包括更新会在第 12 步就地重启的那些。为 `configd` 和 `robotd` 加一条第二条立即路径能换来五秒却要付出一个机制。
- **失败被记录，从不返回。** 成功的更新不因重启无法调度而报为失败；代价是 daemon 在下次启动前停留在旧二进制上。

Timer 在回复写入*之前*调度（引擎在 `update.apply` 调用内运行），所以 5 秒从第 16 步算起，不是从客户端看到答案算起。客户端看到结果然后连接断开——对 BLE 来说是普通重连。

### 重启后的 `updaterd` 接着做什么

它是普通启动，所以完整运行 §4 的启动序列：`clean_staging`、`record_boot`、§5 的协调——那是没发生的 `btd` 重启被抓到的地方——然后服务。两个容易忽略的后果：

- **启动计数器计数 `updaterd` 启动，不是字面启动。** 成功的 apply 在第 15 步确认自己的 trial，所以没有未完成的要推进——但被*更早*中断的更新留下的已武装 trial 会被这次重启推进。
- **定期检查时钟重置。** `INITIAL_CHECK_DELAY` 是进程启动后 60 秒，`check_interval` 从那里计数。

## 3. 其他转换

`select`、`rollback` 和 `reset-to-golden` 共享 `engine.rs::transition_to`：武装、交换、`on_apply`、健康门控、失败时回退。它们**不**跑钩子且**不**自检——没有 artifact 被解压，所以没有新东西要安装或证明。

| | hooks | `on_apply` (§1) | self-test | deferred updaterd/btd restart |
|---|---|---|---|---|
| `update apply` | yes | yes | yes | **yes** |
| `update select <ver>` | no | yes | no | **yes** |
| `update rollback` | no | yes | no | **no** |
| `update reset-to-golden` | no | yes | no | **no** |
| auto-rollback inside a failed apply | no | yes | no | **no** |
| boot-counter revert at startup | no | yes | no | **no** |

底部两个 `no` 通过构造正确：在失败的 apply 和启动计数器回退中，`updaterd` 和 `btd` 从未重启，所以它们仍是属于被返回的 release 的二进制。

显式 `rollback` / `reset-to-golden` 的 `no` 是不同情况，值得知道：如果被离开的 release 在早先某个时候成功 apply 过，`updaterd` 和 `btd` *曾*重启到它上面，显式回退把它们留在那里。`robotd`、`configd` 和 `padd` 移回；那两个保持**领先于**活动 release。

这在下一次 `updaterd` 启动时而非重启时自行解决：§5 双向比较，所以运行比 `current` 新的 release 的 `btd` 同样被测试判为过时并被重启。`updaterd` 是设计上不在那里自愈的那个进程。

因为 `select` 和回退跳过 `hooks/postinstall`，它们也从不*删除* unit 文件。降级到早于某个 daemon 的 release 会留下该 daemon 的 unit 已安装，指向旧 release 不含的二进制；unit 失败，且因为它在重启集中，失败的重启使转换失败（`../project/install-path-gap.md`）。

## 4. 启动时

systemd 从 `multi-user.target` 启动 daemon，`robot-boot-check.timer` 为 180 秒后的恢复检查武装。daemon 之间**没有顺序**，除了 `padd` 在 `robotd` 之后（建议性——如果 socket 不存在 `padd` 退出并每 5 秒重试）和 `btd` 在 `dbus`/`bluetooth` 之后。没有东西等 `updaterd`，`updaterd` 除了 `network-online.target` 什么都不等，那是 `Wants` 而非 `Requires`。

在这块板上 `hci0` 在上电约 73 秒后才存在，所以 `btd` 为重试适配器而非失败。

`updaterd` 自己的启动，按顺序（`main.rs::serve`）：

1. 以 `warn` 记录启动身份行——版本、revision、**`exe` 路径**、pid。`exe` 路径告诉你进程实际来自哪个 release 目录。
2. 加载 `/etc/robot/updater.toml` 和受信任密钥。任一失败都是致命的。
3. 构造引擎。
4. `--self-test` 在此返回，在触碰任何状态之前。
5. `Engine::recover_on_start`，**在 socket 服务之前**，所以启动进坏 release 的机器人在任何东西能要求它做别的之前已经开始回退：
   - 对每个组件 `clean_staging`：删除 `releases/.staging-*`。
   - `record_rescue`：如果 `<state_dir>/rescued` 存在，启动恢复网在什么都没运行时把 `current` 交换到了 golden。把它作为回复制进更新日志，**清除该组件的 trial**，并删除文件。刻意在 `record_boot` 之前：救援已经走得比 trial 更远，留下武装的 trial 会在一两次启动后把 `current` 移离 golden。删除文件也是释放救援循环守卫的动作。
   - `refresh_golden_links`：把每个组件配置的 golden 作为 `golden` 符号链接发布在 `current` 旁边，这样救援能用一次 `readlink` 且无解析器找到它。
   - `record_boot`：对 `pending.json` 中每个已武装的 trial 递增 `boots`。
   - 对每个 `boots >= 2`（`MAX_BOOT_ATTEMPTS`）的 trial：回退到 `previous`，当 `previous` 缺失、磁盘上找不到或自身被记录为已回滚时升级到 `golden`。那意味着交换符号链接、确认 trial、重跑 `on_apply`——**所以启动计数器回退重启 `configd`、`padd` 和 `robotd`。** 它不跑钩子也不调度任何东西，所以 systemd 刚启动的 `updaterd` 和 `btd` 进程保持从被放弃的 release 运行。第 6 步然后抓到 `btd`，因为它在这之后运行并对照回退使其活动的 release 比较；`updaterd` 被报告并留下，直到下次启动都过时。
   - 记录结果。这里的失败被记录且服务继续：拒绝服务会移除修复它的唯一方式。
6. `Engine::reconcile_running_units` —— §5。**在**恢复之后，因为恢复能改变哪个 release 活动，在它之前 unit 会被对照机器人正在放弃的版本比较。这一步可以重启 unit。
7. `--check-only` 在此返回。注意它*执行*恢复和第 6 步的协调——它是操作员工具，不是探针，且它能回退 release 并重启 daemon。
8. 服务 `/run/updaterd.sock`。
9. 如果设了 `check_interval`，派生调度器：首次在启动后 60 秒，然后每个间隔。`auto_apply` 决定无客户端连接时它可安装什么。

Trial 仅当 apply 在 §2 的第 9 步和第 15 步之间被中断时存在——断电、`kill -9`，或交换后被取消的 RPC。已提交的更新不留任何武装的东西。且因为 `exhausted` 是 `boots >= 2` 而 `arm` 写 `boots = 0`，中断后*第一次* `updaterd` 启动记录"update still on trial"，*第二次*回退。

**上述每一步都预设一个能启动的 `updaterd`。** 启动时还有一件事为它都不运行的情况发生：`robot-boot-check.timer` 在 180 秒触发，询问 `robotd`、`configd`、`btd` 和 `updaterd` 是否起来，如果没有就交给一个 `/bin/sh` 救援，把 `current` 交换到 golden 并重启。它刻意在本页每个进程之外——[`boot-recovery-net.md`](boot-recovery-net.md) 负责它，包括为什么它是截止时间而非这些 unit 上的 `OnFailure=`。

## 5. 启动协调——重启发生了吗？

`systemd-run` 返回成功意味着创建了一个瞬态 timer。不意味着重启运行了，也不意味着新二进制启动了；§2 第 16 步刻意吞掉那些失败。这留下了一个让机器人最终运行它没安装的 release 的静默方式，且它落在没有别的东西监视的两个 unit 上。`updater/src/reconcile.rs` 关上它。

### 它读什么

每个 daemon 在启动时发布自己的身份——`duck_ipc_proto::publish_identity`，由每个 daemon 打开时调用的 `log_startup_identity!` 宏调用：

```
/run/<service>/identity.json    { service, version, revision, built_at, exe, pid }
```

`exe` 字段来自 `/proc/self/exe`，所以它通过 `current` 符号链接解析并点名进程实际启动的 release 目录。`Identity::release()` 从中解析出 `releases/<ver>` 组件。

自发布而非从外部读取：进程知道自己的版本和 revision，且能无需特权读自己的 `exe`，而读另一个用户的需要 root。每个 unit 中的 `RuntimeDirectory=<service>` 创建由该 unit `User=` 拥有的目录，在 `ProtectSystem=strict` 下存活，且**在 unit 停止时被 systemd 移除**——所以停止的 daemon 不能留下声称正在运行的身份。

### 它决定什么

按组件，对 `units_shipped`（§1）中的每个 unit——所以 `updaterd` 和 `btd` 被包含，这正是要点——把发布的 release 与组件的活动 release 比较。`verdict_for` 是纯函数；每个 syscall 都保持在它之外。

| verdict | when | action |
|---|---|---|
| `Current` | published release == active | nothing, silently |
| `Restarted` | they differ | `systemctl restart <unit>`, at `warn` |
| `ReportedOnly` | they differ **and the unit is `updaterd`** | logged, never acted on |
| `RestartFailed` | the restart was attempted and failed | logged at `error` |
| `Unknown` | no identity file | nothing |

三个刻意的不作为：

- **`updaterd` 从不在此重启自己。** 一个对哪个 release 活动有异议的后继者会重启、再次异议、循环——在拥有恢复的那个进程里，所以没东西能打破循环。只报告是安全的：进程刚启动，所以关于它的任何过时都在此代码运行前已决定。它从另一侧被修复——见下。
- **停止的 unit 保持停止。** 无身份文件意味着要么停止要么太旧无法发布，两者在此读起来一样。启动它会在每次 `updaterd` 启动时覆盖停止它的人。
- **什么都没发布的 daemon 不被当作过时。** 因旧而重启机器人的 daemon 是没人要求的决定；下次更新让它们能应答。

比较是方向无关的——任何差异都是过时——这让它覆盖被显式回滚留在*领先于* `current` 的 release，不仅是被留在后面的。

它在每次 `updaterd` 启动时运行，所以在启动时是空操作：一切都从同一个符号链接启动。代价是每个 unit 一次文件读取。

### 同样的读取，来自 `apply`

`reconcile::stale_units` 是对同样身份文件的 `verdict_for`，去掉了 acting，`apply` 和 `select` 在它们已经 current 的路径上调用它（§2）。它存在是因为上面的例外：过时的 `updaterd` 是本模块拒绝修复的唯一不一致，而之后唯一看它的是因为 daemon 似乎有问题而跑 `apply` 的人。那曾回答 `already_current` 且不调度任何东西，读作确认没有东西要修。

两个性质让它不会成为 §5 防范的循环。它在请求时而非每次启动触发，且通过 `systemd-run` 调度而非从被重启进程内部驱动 `systemctl`。它既报告也行动：unit 在 `already_current` 结果中被点名，所以客户端——`robotctl` 或 app——看到哪些 daemon 没在运行机器人已安装的 release。

两种过时判定在此都算作过时（`Restarted` 和 `ReportedOnly`）；它们之间的区别是谁可以执行重启，而非出了什么问题。表说的其他一切不变：停止的 unit 和什么都没发布的 daemon 仍不过时。

## 6. 首次安装

`scripts/install.sh` 在裸板上，按顺序（`main`）：

1. 写 `/etc/robot/updater.toml` 和受信任密钥。
2. `bootstrap_first_release`：获取独立的 `updaterd` 二进制并运行 `updaterd install [--from <dir>]`。那是普通的 `Engine::apply`，强制两个设置，因为在没有 release 的板子上它们是事实而非策略：`on_apply = none`（unit 住在正在安装的 release 内）和 `health = none`（没有 `robotd` 可探测）。所以 §2 运行时跳过第 12、13、14 步——但**第 11 步仍运行**，`hooks/postinstall` 正是安装 unit 并对每个有 `[Install]` 段的 unit 执行 `enable --now` 的地方。daemon 首次在那里启动，从钩子内部。第 16 步也没跳过：引导安装把 `updaterd` 和 `btd` 重启调度在 5 秒后，此时 `install.sh` 仍在运行。
3. `install_units`：再次复制 unit、`daemon-reload`，然后按 `updaterd`、`robotd`、`configd`、`btd`、`padd`、`mediad` 顺序 `enable --now`——`configd` 在 `btd` 之前因为 `btd` 向 `configd` 要配对 PIN，`mediad` 最后因为它的 unit `After=` 其他三个。`btd`、`padd` 和 `mediad` 可以失败而不导致安装失败：没有无线电、没有手柄或没有摄像头的机器人仍能更新和行走。然后 `robot-boot-check.timer` 被启用*不带* `--now`。与钩子冗余且无害；它也是早于钩子的 release 的路径。release 发货但此函数不知道的 unit 被安装、报告、不管。`tofd` 被点名为已知但没有自己的 `enable_unit`：钩子在一步前启用了它且没有东西依赖它，所以此函数没有可发表意见的顺序。
4. `install_token_dropin`：写 `GITHUB_TOKEN` drop-in、`daemon-reload`、`systemctl try-restart updaterd`——光 `daemon-reload` 会让*运行中的* `updaterd` 没有 token，那是每块板的情况。

`DUCK_FORCE_REINSTALL=1` 在第 2 步之前加 `stop_for_reinstall`：按 `padd`、`tofd`、`btd`、`configd`、`robotd`、`updaterd` 顺序 `systemctl stop`。交换发生时没有任何活动的东西，且后面没有健康门控。

## 7. 诊断不一致

两个版本号同时合法不同，是哪一对决定诊断。

| observation | means |
|---|---|
| `updaterd` or `btd` behind the installed release, for a few seconds after an update | expected — the deferred restart has not fired yet |
| `btd`, `robotd`, `configd`, `padd`, `mediad` or `tofd` disagreeing with `current` at all, persistently | the restart did not take effect *and* §5 did not fix it — so either `updaterd` has not restarted since, or its restart failed (journal, at `error`) |
| `updaterd` behind it, persistently | the deferred restart never landed, and §5 will not fix this one. `robotctl update apply daemon` repairs it — it reports `already_current` with `updaterd` in `stale` and schedules the restart. `systemctl restart updaterd` is the same fix by hand; either way the journal has why it did not land |
| any daemon reporting `build unknown (old)` | it predates the identity mechanism, so §5 leaves it alone. One update makes it answerable |

问机器人而非推断：

```bash
robotctl health
```

`units` 块为每个 daemon 打印一行，带其进程启动的 release，并在与已安装内容不一致时警告。

手动，从 daemon 发布的文件得到同样答案：

```bash
cat /run/configd/identity.json; readlink /opt/robot/daemon/current
```
