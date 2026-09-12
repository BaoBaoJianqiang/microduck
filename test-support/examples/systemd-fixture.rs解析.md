# 解析：`test-support/examples/systemd-fixture.rs`

## 这是什么

一个 **Rust example 可执行程序**（290 行），为**唯一需要真实 systemd 的测试夹具** [`scripts/systemd-test.sh`](../../../scripts/systemd-test.sh) 铸造发布版——这些发布携带真实 systemd unit 与真实的 `updaterd` 二进制。

文件头注释解释了它与姊妹程序 [`fake-release.rs`](fake-release.rs解析.md) 的分工：后者刻意铸造**无 unit、`on_apply = none`** 的发布，适合几乎所有测试，但恰好答错唯一的问题：**重启机制真的工作吗？** 回答它需要"真的会启动的 unit、真的会触发的瞬时定时器、以及真的被后继版本替换的 `updaterd`"，所以它需要自己的夹具，而不是在那个夹具上加一个开关。

## 每个发布携带的内容（头注释清单）

| 文件 | 作用 |
|---|---|
| `bin/updaterd`、`bin/robotctl` | 被测二进制，为容器环境构建 |
| `systemd/updaterd.service` | 让更新能重启更新器自己——全部要点所在 |
| `systemd/fake-robotd.service` | 重启集合中一个守护进程的替身 |
| `systemd/btd.service` + `bin/fake-btd` | 唯一被排除在重启集合之外的守护进程的替身；脚本发布 identity 后睡眠 |
| `systemd/recovery-check.service` | 无 `[Install]` 段的 oneshot，任何东西都不许重启它 |
| `systemd/broken.service` | 仅 `:broken-unit` 变体携带：`ExecStart=/bin/false` |
| `systemd/needs-a-user.service` | 仅 `:sysusers` 与 `:missing-user`：带 `User=` |
| `systemd/sysusers.d/duck-test.conf` | 仅 `:sysusers`：创建该用户的配置 |
| `hooks/postinstall` | **真实的** postinstall 钩子，使 unit 被真的安装与 enable |

两个替身的设计意图（注释）：

- 用 `fake-robotd` 而非真 `robotd`：被测的是"引擎是否重启发布携带的东西"，一个 `sleep` 与电机控制一样能证明它，且不需要硬件、策略、socket；**它主 PID 的变化就是观察点**。
- **替身故意就叫 `btd`**：`NEVER_RESTART` 是代码中的 unit 名列表，只有真的叫 `btd` 的 unit 才会被排除在在途重启之外、改由延迟重启拾起——这是观察"启动时调和（startup reconciliation）存在的那个情形：一次从未发生的延迟重启"的唯一途径。与 fake-robotd 不同它还要发布 identity，因为调和读的正是 identity。

## CLI 参数（第 41-70 行）

| 参数 | 说明 |
|---|---|
| `root` | 要创建的目录，非空则拒绝（与 fake-release 同理由：每次新铸密钥） |
| `--updaterd` | 放进每个发布的 `updaterd`，为运行环境构建 |
| `--robotctl` | 配套的 `robotctl`，必须与 updaterd 同一发布，否则握手拒绝 |
| `--postinstall` | **真实的** `hooks/postinstall`，让夹具观察到发布自带钩子而非桩 |
| `--prefix` | 树被*读取*的位置（主机铸造、挂入容器，路径全部不同） |
| `releases`（必填） | `<version>[:broken-unit\|sysusers\|missing-user]` |

三种变体：`broken-unit` 增加一个装得上但起不来的 unit；`sysusers` 的 unit 所需 `User=` 由发布自带的账号配置创建；`missing-user` 是同一个 unit 但没有任何东西创建用户。

## 各 unit 常量逐段解析

### `updaterd_unit(prefix)`（第 77-88 行）

被测更新器自己的 unit，与出厂版一样**穿过 `current` 符号链接**指向 `ExecStart={prefix}/opt/daemon/current/bin/updaterd`。注释强调 `RuntimeDirectory=updaterd` 不是为整洁而是**承重**的：守护进程在那里发布"我在跑哪个发布"的文件，而那个文件是判断"延迟重启确实替换了本进程、而不只是被排程"的唯一途径。`Restart=on-failure`、`RUST_LOG=info`。

### `FAKE_ROBOTD_UNIT`（第 91-93 行）

`ExecStart=/bin/sleep infinity`、`Restart=always`——一个可被重启且可观察的重启集合成员。

### `FAKE_BTD`（第 100-105 行）

btd 替身运行的 sh 脚本：用 **`readlink -f "$0"`**（而非 `$0`）取得真实可执行路径，向 `/run/btd/identity.json` 写 `{service, version, exe, pid}` 后 `exec sleep infinity`。注释解释为何解析符号链接：unit 穿过 `current`，一个报符号链接名的 identity 说明不了跑的是哪个发布；真实守护进程读 `/proc/self/exe`，出于同样的原因它会被解析。

### `fake_btd_unit(prefix)`（第 108-116 行）

替身的 unit，**特意命名 `btd`** 以使 `NEVER_RESTART` 生效；`ExecStart` 同样穿过 `current`，`RuntimeDirectory=btd`。

### `NEEDS_A_USER_UNIT` / `SYSUSERS_CONF`（第 121-125 行）

`User=duck-test` 的 unit + sysusers 配置行 `u duck-test - "..." /nonexistent`。注释点明这验证的是 postinstall 的**承重顺序：账号先于 unit**——顺序反了 unit 起不来，且失败看起来像守护进程损坏、而非缺账号。

### `GHOST_USER_UNIT`（第 129-131 行）

`User=nosuchuser-duck` 且无任何东西创建它：`enable --now` 起不来，随后的重启也起不来。

### `RECOVERY_CHECK_UNIT`（第 139-141 行）

形状仿恢复网 oneshot：`Type=oneshot`，`ExecStart` 向 `/run/recovery-check.ran` 追加 `ran`。**没有 `[Install]` 段就是全部要点**：真实的那个在开机时询问本次启动的发布是否健康、不行则交给 `robot-rescue`，更新中若重启它会把一次回滚检查指向正在合法重启中的守护进程。它记录每次运行，使夹具能**断言它没运行过**，而不是假设。

### `BROKEN_UNIT`（第 145-147 行）

装得上、起不来（`/bin/false`）的 oneshot。注释：这是 `install-path-gap.md` 的 1 号 bug 的单文件重现——更新必须失败并点名它，而不是带着 `"not healthy: unreachable"` 回滚。

## main() 流程（第 149-278 行）

1. **空目录守卫**（154-156）；创建 `keys`、`published`、`opt/daemon`、`var`、`bin` 五个目录（158-160）。
2. 读入 `updaterd`/`robotctl` 字节与 `postinstall` 文本；确定 `prefix`（162-168）。
3. **引导副本**（172-174）：在任何发布之外向 `bin/updaterd` 放一份并加可执行位——模拟裸板引导：首次安装必须由一个"尚未装到任何地方的 updaterd"来执行。
4. 构造 `Publisher`，逐规格铸造发布（178-246）：公共部分为 updaterd/robotctl 两个二进制（0o755）、三个固定 unit（updaterd、fake-robotd、recovery-check，0o644）、fake-btd 脚本与 btd unit，以及**真实 postinstall 钩子**（`.hook(&postinstall)`）；再按变体追加 broken / sysusers（conf+needs-a-user）/ missing-user（ghost unit）；未知变体报错。
5. **生成 updater.toml**（254-270），与 fake-release 的关键差异：

```toml
on_apply = { action = "restart", units = [] }
health = { probe = "command", program = "/usr/bin/systemctl",
           args = ["is-active", "--quiet", "fake-robotd"], timeout = "20s" }
```

- `units = []` **不等于 `none`**：重启集合从发布携带的 unit 推导，空列表意为"重启你带的东西"，仅此而已；
- 健康探针用 **command 而非 socket**：这里没有 robotd；问 systemd"刚重启的守护进程是否 active"是一个真门，而 `/bin/true` 只会是走过场。

`make_executable`（280-289）按平台分叉：Unix 下设 0o755，非 Unix 为空操作（夹具实际只在 Linux 容器跑）。

## 与 postinstall 钩子的闭环

本夹具特意嵌入**真实的** [`hooks/postinstall`](../../../hooks/postinstall)：它在交换后安装并 enable 全部 unit、跑 `systemd-sysusers` 建账号、跳过无 `[Install]` 的 recovery-check。于是三类失败情形（unit 损坏、账号顺序、幽灵用户）与两类重启情形（在途重启 fake-robotd、延迟重启 btd）都在真实 systemd（pid 1）容器中端到端可观察。
