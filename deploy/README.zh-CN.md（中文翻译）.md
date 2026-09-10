# 部署：操作系统级配置

状态：草稿 · 日期：2026-07-28 · 负责人：pierre

属于*机器人镜像*而非任何单个服务的配置。服务单元位于其服务旁边（`updater/systemd/`、`robotd/systemd/`）；凡是机器人级的内容都在这里。

| | |
|---|---|
| `updater.toml` | 客户端机器人出厂携带的配置，安装到 `/etc/robot/updater.toml` |
| `trusted_keys/` | 发布公钥——信任锚，安装到 `/etc/robot/trusted_keys/` |
| `journald.conf.d/10-robot.conf` | journal 持久化与大小上限 |

关于最后一项的说明，现在是**实测**而非假设：此镜像上的 `/var/log` 是一个 zram 设备，所以 `Storage=persistent` 实际上是一个内存中的目录。它能在一次干净的重启后幸存，但会在断电时丢失最近的日志——而这正是机器人的实际关机方式。因此 `/var/lib` 下的更新历史是唯一持久的记录——这正是 `architecture.md` §8.2 将其设计成的样子。

> 只想让一块开发板跑起来的话，[`docs/robot/install-dev.md`](../docs/robot/install-dev.md) 是简短的
> 流程。下面讲的是信任链、什么东西装到哪里，以及日志去哪。

## 快速开始

三种进入方式，按你需要键入多少来排序。本节之后的内容是同一件事附上理由——在你觉得有什么不对劲时再读，而不是提前读。

### 开发板，仓库私有——这就是现状

克隆后一条命令，逐步讲解见
[`docs/robot/install-dev.md`](../docs/robot/install-dev.md)：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
./scripts/provision-board.sh radxa@192.168.1.42
```

`--no-dev-key` 用于只应接受发布的板卡，`--ref BRANCH` 用于从某个分支配置（provision），`--local` 用于发送本克隆的 `provision.sh` 而不是去抓取它——这正是能够测试对它的未推送变更的原因。

`team.dev.pub` 提交在 [`dev-key/`](dev-key/)，不在 `trusted_keys/`——把它带到板卡上*就是*选择加入（opt-in），所以它不能随每台机器人出厂。脚本发送已提交的副本；`--dev-key PATH` 覆盖它。

### 在板卡上，没有克隆

三条命令，第一条在你的机器上执行，以及上面 `provision-board.sh` 替你所做的事：

```bash
scp deploy/dev-key/team.dev.pub radxa@192.168.1.42:/tmp/
```

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
curl -fsSL -H "Authorization: Bearer $DUCK_TOKEN" https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/provision.sh -o /tmp/provision.sh && sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/provision.sh
```

`provision.sh` 依次运行 `setup-board.sh`、`migrate-network.sh` 和 `install.sh`，警告十秒，重启——你的 SSH 会话到此结束——然后自行完成。它先把开发密钥从 `/tmp` 复制出来，因为 `/tmp` 不会在该次重启后幸存；留在那里的密钥会产生一块"配置干净利落、却静默地*不是*开发板"的板卡，数周后以 `--ref` 被拒绝的形式浮出水面，看起来像发布损坏。

重新登录后：

```bash
robotctl health
```

如果你回来后它还在工作，观察它：

```bash
sudo tail -f /var/lib/robot/provision.log
```

该日志是"没人看着的那一半"的记录，而且它刻意是一个文件而非 journal：journald 持久化由正在安装的发布物内部的 drop-in 配置，所以在这个确切的时间窗口内 journal 仍可能只是 RAM 中的。只有密钥真正安装时它才会以 `DEV BOARD` 结尾，所以你最终得到哪种板卡是*可以查证*的，而不是*必须记住*的：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

两条路径上都没有 `newgrp robot`，这是刻意的而非遗漏：`robot` 组在重启前就已创建，所以你重新登录进的会话已经拥有它。

### 普通用户，仓库公开

没有 token，也没有开发密钥。

```bash
curl -fsSL https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/provision.sh -o /tmp/provision.sh && sudo sh /tmp/provision.sh
```

```bash
robotctl health
```

即使在这里也是"下载"而非"管道直通"：后半段在重启后运行，所以磁盘上必须留有一个文件供它使用。它拒绝管道，而不是把你半途撂下。

### 手工执行

`DUCK_NO_REBOOT=1` 让 `provision.sh` 在重启前停下并告诉你该运行什么——当你想看着每个步骤的状态块依次通过时，就用这个形态：

```bash
sudo DUCK_NO_REBOOT=1 DUCK_TOKEN="$DUCK_TOKEN" sh /tmp/provision.sh
```

```bash
sudo reboot
```

```bash
sudo /usr/local/sbin/robot-provision
```

这三个脚本也保持各自独立可运行，而 `provision.sh` 是一个没有自身逻辑的薄编排器。逐个执行的顺序是 `setup-board.sh`、`migrate-network.sh`、重启、两者再各跑一遍、`install.sh`——然后是 `newgrp robot`，因为在那条路径上，重启前没有任何东西创建该组。

## 那些命令实际上做了什么

三个脚本，彼此分开是因为它们对不同的东西负责。`setup-board.sh` 是操作系统级启动——设备树 overlay、ONNX Runtime——很少改动且需要重启。`install.sh` 安装一个已签名的守护进程发布物，每次更新都会发生；把两者混为一谈意味着每次更新都要重新争论启动配置。`migrate-network.sh` 是那个不会长久的一个：它存在仅仅因为 Armbian 的出厂镜像自带 netplan，而且在我们构建出带 NetworkManager 的镜像的那一天，它会被删除而不是被维护。

**板卡到货时并没有 NetworkManager**，这正是中间那个不是可选的原因。Armbian 的无头镜像运行 netplan + `systemd-networkd` + `wpa_supplicant`，而 `configd` 通过 D-Bus 驱动 NM——所以在迁移运行之前，`robotctl net status` 报告 `Unavailable`，蓝牙也无法配置 wifi。为什么用 NM 而不是镜像自带的，见 [`../docs/design/app-path-design.md`](../docs/design/app-path-design.md) §2。

`provision.sh` 按顺序运行它们，并持有必须跨过重启的状态——token、开发密钥路径，以及它用来判断你是否真的重启过的 boot id。它刻意没有任何自身的配置逻辑：三个生命周期不同的脚本不应变成一个各部分无法单独移除的脚本。

`provision-board.sh` 再往外一层，运行在你的机器上而非机器人上。它存在的意义在于中间那道缝：`provision.sh` 自行重启并完成，这没错，但从外面看，那像是一次 ssh 会话死掉，然后猜测该何时重新登录。它等待板卡、流式输出日志、以健康报告结束——所以配置是一次带连续输出的命令，而不是中间有空档的三条。

它确实承担了重启，而它调用的脚本刻意从不这么做。这不是矛盾：它们是单一用途的，可以在正忙着别的事的机器人上运行，所以两者都无法知道重启会打断什么。而这个只在正在配置的板卡上运行，在那里重启是下一步而非打断。

代价是后半段无人值守运行，而那里可能出错的是一个循环而非一次失败——所以有两道防护。恢复单元在任何工作*之前*先禁用自身，所以最多只有一次自动尝试：一个死掉的第二阶段留下的是"一块可以查看的板卡"，而不是"一块反复撞同一堵墙的板卡"。而且 `migrate-network.sh` 只在 NetworkManager 已经拥有 wifi 时才重跑，意思是切换已生效、这次运行只是为了退役那个后备方案。如果后备方案曾经触发并恢复了 netplan，无人值守地再次切换会重新武装它、以同样的方式失败、重启、再来一轮；它会在日志里说明这一点，并让 wifi 保持原样。单元文件保留在磁盘上，禁用状态，作为曾运行过的记录。

token 需要三次，而只有前两次以配置结束：抓取这些脚本、抓取发布物，然后永久地——`updaterd` 在每次后续更新检查时从一个 systemd drop-in 读取 `GITHUB_TOKEN`。传递 `DUCK_TOKEN` 是让更新在*配置之后*也能工作、而不只是在配置期间工作的原因，而且 `provision.sh` 结束时说明两份副本中它移除了哪一份、留下了哪一份。没有 token 的板卡能正常安装，但之后永远无法抓取更新——而这正是 `updaterd` 的大部分用途。

它还（在第一阶段）创建了 `robot` 组，这是上述流程中没有 `newgrp robot` 的唯一原因。`install.sh` 也正确地做了同样的事，但太晚了——等它运行时，你的 shell 是在该组存在之前启动的，而进程的组在它启动时就固定了。把它挪到重启之前，意味着你返回的登录会话已经拥有它。

**token，以及为什么错误的 URL 是错误的诊断。** `raw.githubusercontent.com` 对无凭据的私有路径回答 **404** 而非 401，所以缺失的请求头看起来和拼写错误一模一样。涉及两个独立的 token：你 shell 里的那个，用于抓取脚本；以及 `updaterd` 需要用来访问发布资产的那个——[那个](#-while-the-repository-is-private-a-robot-needs-a-token)是一个 systemd drop-in，比 shell 活得更久。两者在 `provision.sh` 到达 `install.sh` 时汇合：传递 `DUCK_TOKEN` 正是写入 drop-in 的方式。

**`setup-board.sh`** 是幂等的，且从不自行重启。它修复的、否则极难诊断的一件事：Armbian 自带 `overlay_prefix=rk35xx`，但 RK3566 与 RK3568 共享设备树 overlay，且它们命名为 `rk3568-*.dtbo`。前缀错误时加载器什么也找不到，板卡愉快地启动，但就是没有 `/dev/ttyS2`。`armbian-config` 的 overlay 编辑器也因同样的原因崩溃，所以文件是被直接修补的。

⚠ 重定向 `/boot/{Image,dtb,uInitrd}` 的内核升级可以撤销它。一块在 `apt upgrade` 后看不到电机的板卡需要重跑这一步。

⚠ 它的 `kernel console` 状态行在脚本已经修复、只有运行中的内核还是旧的时读作 `until the reboot`——`/proc/cmdline` 不重启就无法改变。该行上的任何其他内容才是真正的发现。

**`migrate-network.sh`** 运行**两次**，在重启的两侧——`provision.sh` 两者都做，值得知道为什么第二次重要。第一次运行武装一个启动时后备方案：如果 `wlan0` 没有得到 IPv4 地址就恢复 netplan，所以一次糟糕的切换代价是一次重启而不是一根串口线，第二次运行退役它。保持武装的话，任何后续启动中仅仅 wifi 较慢就会回退板卡。它从 netplan 本身取 SSID 和密钥——如果 netplan 生成了 `/run/netplan/wpa-wlan0.conf` 就用它，否则用 `/etc/netplan/*.yaml` 里的 `access-points:` 小节——所以同一 wifi 上的 SSH 会话能幸存。如果两者都读不到，它**什么也不改**并打印手工创建配置文件的 `nmcli` 命令。

**`install.sh`** 只需要 `curl` 和 coreutils，别的都不要——`tar` 和 `zstd` 已链接进 `updaterd`。幂等，且从不覆盖已存在的 `/etc/robot/updater.toml`。仓库私有期间它是两条命令而非 `curl … | sudo sh`，因为请求头要放在抓取上，而 `sudo` 不会自行传递变量；管道两者都带不了。它接受的一切都是环境变量，因为它通常通过管道运行、在那里 flag 很别扭：

| | |
|---|---|
| `DUCK_TOKEN` | 私有仓库的 token——抓取*和*发布资产 |
| `DUCK_REPO` | 仓库，用于 fork 或测试仓库 |
| `DUCK_REF` | 脚本和信任密钥的读取分支；固定到某个 tag 以获得可复现的运行 |
| `DUCK_CONFIG_REF` | `updater.toml` 的来源。默认为所安装发布物的 tag，且 `DUCK_REF` 不会改变它——配置字段只有其自身版本起的二进制才能理解，所以把某个分支的配置与最后一个稳定二进制配对，正是 `updaterd` 最终拒绝启动的原因。仅在你用能理解该配置的构建测试配置变更时设置它。 |
| `DUCK_DEV_KEY` | `team.dev.pub` 的路径——使这块板成为开发板（见下文） |
| `DUCK_FORCE_REINSTALL` | 使用发布物自己的 `updaterd` 在活动发布物上重装 |

**它如何安装自身。** 更新需要更新器，而更新器随更新而来。出路是一个裸的 `updaterd` 二进制，作为 `updaterd-bootstrap-aarch64` 资产发布，因为一块新板卡没有 `zstd` 来打开 `.tar.zst`。然后它运行*普通的*引擎——同样的验证、解压、原子交换、journal 条目——所以存储区以驻留守护进程期望的完全相同状态结束，且没有任何仅引导路径会偏离后续更新的行为。`on_apply` 和 `health` 在此期间被强制关闭，因为单元文件位于正在安装的发布物内部；一旦发布物*已*生效，`updaterd install` 就拒绝运行，所以那永远不会在一台正常工作的机器人上静默禁用自动回滚。`--from <dir>` 改为从本地文件安装：离线和工厂路径，也是 CI 在发布前验证发布物所用的。

### 让它成为开发板，让 `--ref <branch>` 可用

同样的两个条件也把关从笔记本电脑直接推送的构建（用 `scripts/dev-push.sh`）——它用同一个开发密钥签名，所以拒绝分支构建的板卡也同样拒绝那些。

一块板卡拒绝分支构建是双重的：`allow_dev_keys` 为 false，且一个受信任密钥只有在其文件名以 `.dev.pub` 结尾时才算开发密钥。两半都需要，它们是独立的检查，只做其一而不做另一，留下的是一块仍然拒绝分支构建的板卡——带着一个读起来像发布损坏的签名错误。`DUCK_DEV_KEY` 两者都做。

它在改变任何东西之前先验证：文件必须存在、必须看起来像 minisign 公钥，且 `updater.toml` 必须已有要翻转的 `allow_dev_keys` 行——该键是顶层键，所以追加一行会把它落进最后一个 `[table]` 内部。无论源文件叫什么，它都把密钥安装为 `team.dev.pub`，因为 `.dev.` 中缀才是分类依据；以任何其他名称落地的密钥会被当作*发布*密钥信任，分支构建随后会被当作已审查而接受。

`team.dev.pub` 提交在 [`dev-key/`](dev-key/)，刻意在 `trusted_keys/` 之外——[`dev-key/README.md`](dev-key/README.md) 解释为什么这样是安全的。

结束报告在此开启时输出 `DEV BOARD`，并打印撤销它的两条命令。永远不要对你交付的机器人这么做。

### ⚠ 仓库私有期间，机器人需要 token

私有仓库的发布资产没有凭据就无法访问——`releases/download/...` URL 即使有 token 也 404，所以引擎改为通过发布 API 解析资产。`updaterd` 从其环境读取 `GITHUB_TOKEN`，在板卡上这意味着 systemd drop-in，而不是 shell 导出。

这在开发者的板卡上没问题，在客户机器人上则**不行**：镜像中机群级凭据是会泄露、且不重刷就无法轮换的东西——这正是分级签名密钥存在所要避免的失败。

因此 `install.sh` **仅在提供了 `DUCK_TOKEN` 时**才写入 drop-in——mode 600，而且它会明确大声提示。客户机器人从公共制品仓库安装且不传 token，所以永远不会走到那条路径。没有它，`updaterd` 会被安装、运行，却连一个更新都抓不到——而这正是它的大部分用途。

制品托管因此是一个开放的决策，而非已定论的——[`../docs/design/updater-design.md`](../docs/design/updater-design.md) §6.1 列了选项。便宜的那个是第二个、只存放已签名制品的公共仓库：让制品可以安全分发的是签名，而不是隐蔽性，源码保持私有。

### 信任链

1. 到 `raw.githubusercontent.com` 的 TLS，用于 `install.sh`、`updater.toml` 和公钥。
   这些不能来自发布物：密钥就位前没有任何东西可被验证。
2. 到 `github.com` 的 TLS，用于引导 `updaterd`。**尚未验证。**
3. 该二进制对照 (1) 的密钥验证 manifest 和制品，并拒绝任何未被它们签名的东西。
4. 安装器随后将引导二进制的 `sha256` 与 `current/bin/updaterd` 比较，后者来自已验证的制品。摘要相等意味着 (2) 中的二进制是真的。`release.yml` 断言两者是相同字节，所以不匹配是真实发现而非打包怪癖。

其余一切——两个单元文件、journald drop-in、`robot.conf`——都取自*已安装的*发布物而非从仓库抓取，所以它是签名所校验的那份副本。

残余信任是 GitHub 本身，脚本也来自那里；步骤 (4) 收窄而非消除这个窗口。想要完全不要它的安装，应针对手工携带进来的文件使用 `--from`。

### 无人值守更新

`updaterd` 已经是一个带定时器的常驻进程——`updater.toml` 里的 `check_interval`——所以 cron 里没什么要加的。定时器被允许安装什么由 `auto_apply` 决定：

| | |
|---|---|
| `off` | 从不；可用性会被记录，强制发布则会被大声记录 |
| `mandatory` | **出厂默认**——只有其 `min_supported` 下限说运行版本不得再用的发布 |
| `all` | 每个可用发布 |

想要跟踪 `staging` 并安装每个候选的 canary 或 bench 机器人：

```bash
sudo sed -i 's/^auto_apply = .*/auto_apply = "all"/' /etc/robot/updater.toml
```

```bash
sudo sed -i 's/^tag_prefix     = .*/tag_prefix     = "daemon-staging-v"/' /etc/robot/updater.toml
```

```bash
sudo systemctl restart updaterd && journalctl -u updaterd -b | tail -20
```

第一次检查在启动后 60s，之后每 `check_interval` 一次。`auto_apply = "all"` 在启动时以 `warn` 级别记录，所以"为什么这台机器人在没人要求时重启了"在任何日志级别下都可以从 journal 得到答案。

不要改用 cron 或 systemd 定时器。`robotctl update apply` 刻意绕过已知不良防护——重试某个发布的操作员可能已经修复了原因——所以由定时器驱动它会继承该绕过并失去防护，一个坏发布会变成无休止的 apply/rollback 循环，每个间隔都重新下载并重写 eMMC。`updater-design.md` §8.1.1 有细节。

不需要维护窗口，这是刻意的。无人值守 apply 就是普通 apply：preflight 在*任何网络访问之前*先问 `robotd` 现在重启是否安全、是否有远程会话在线，所以行走中或直播中的机器人会拒绝并在下一个间隔重试。对"现在是不是坏时机"这个问题，`safeToRestart` 比时钟是更好的答案。

### 什么东西装到哪里

```
/etc/robot/updater.toml                 config; never touched by an update —— 配置；更新从不触碰
/etc/robot/trusted_keys/release-*.pub   trust anchor —— 信任锚
/opt/robot/daemon/releases/<version>/   the release tree —— 发布物树
/opt/robot/daemon/current -> releases/<version>
/etc/systemd/system/*.service           every unit the release ships, copied out of it —— 发布物携带的每个单元，从其中复制出来
/usr/lib/sysusers.d/robot.conf          creates the `robot` group —— 创建 `robot` 组
/var/lib/robot/updater/                 lock, update log, boot counter —— 锁、更新日志、启动计数器
/usr/local/bin/robotctl -> current/bin/robotctl
/usr/local/sbin/robot-provision         provisioning, resumed after its reboot —— 配置，在其重启后恢复
/var/lib/robot/provision.log            what the unattended half did —— 无人值守那一半做了什么
/etc/systemd/system/robot-provision.service   disabled once provisioning finished —— 配置完成后禁用
```

单元文件是**复制**而非通过 `current` 符号链接：通过符号链接读取的话，它们会在每次更新时在 systemd 眼皮底下变化，而回滚之后 systemd 的视图将取决于最后一次 `daemon-reload` 时恰好哪个发布物是活动的。`robotctl` *确实*是符号链接，因为它是操作员调用的工具，而不是 systemd 缓存的文件。

变更操作仅限 root：`updater.toml` 中的 `allow_uids`/`allow_gids` 刻意留空。`robot` 组成员资格把进程带到"能与 `updaterd` *对话*"为止——status 和日志——不再往前。`btd` 的 uid 在它存在时加入允许列表，因为"可以转发来自 App 的更新请求"是比"在 robot 组中"更窄的主张。

⚠ **直接运行 `install.sh` 时，第一次 `robotctl health` 会失败，而安装本身没问题。** 两个套接字都是 `root:robot` mode 0660，且 `install.sh` 把操作员放进 `robot`——但进程的组在它启动时固定，所以运行安装的那个 shell 并不在它刚获得的组里。一条命令，在同一个 shell 里：

```bash
newgrp robot
```

没有 API 能向运行中的进程添加组，即使 root 也不行，所以安装器做的任何事都无法修复启动它的那个 shell——这就是为什么 `provision.sh` 在其第一阶段就创建该组，而重启完成这项工作。在直接路径上，`install.sh` 在结束报告中打印那条命令，`robotctl` 在失败时指名道姓而非把你送去 `systemctl status`——后者会显示两个完全健康的守护进程。

## 日志去哪，以及什么能在重启后幸存

每个守护进程都记录到 **stderr**，systemd 将其捕获进 journal。级别是 `RUST_LOG`，在每个单元中设置（`info`）。

两份记录，持久性刻意不同：

| | 位置 | 能挺过重启 | 能挺过断电 | 上限 |
|---|---|---|---|---|
| 服务日志 | journald | 仅当配置了（见下文） | **否**——`/var/log` 是 zram，见下文 | `SystemMaxUse=200M` |
| **更新历史** | `/var/lib/robot/updater/update-log.jsonl` | **是** | **是** | 200 条 |

更新历史不在 journal 里是刻意的。它位于引擎的 `state_dir` 下、`/var/lib` 中，每条追加时都被 `fsync`，重写走带父目录 `fsync` 的原子临时文件加改名（`updater/src/journal.rs`、`updater/src/fsutil.rs`）。所以"这台机器人装了什么、后来发生了什么"即使 journal 易失也能幸存，且可以用 `robotctl update log` 或直接从磁盘作为 JSON 行读取。该性质由测试验证，而非假设。

服务日志需要本目录中的 drop-in。安装它，然后：

```bash
sudo systemctl restart systemd-journald
```

然后确认保留了不止一次启动——这是实际的验收检查，而且它只在*真正重启之后*才有意义：

```bash
journalctl --list-boots
```

两行或更多意味着上一次启动可达。一行，或
`no persistent journal was found`，意味着日志仍是仅 RAM 的。

读取特定服务、上一次启动：

```bash
journalctl -u robotd -b -1
```

### RAM 日志的注意事项——实测，并决定放弃持久性

此镜像上的 `/var/log` 是一个 **zram 设备**，所以 `Storage=persistent` 给 journald 的是一个本身就在内存中的目录。在板卡上确认过而非推断：

```bash
findmnt /var/log
```

它能挺过一次干净的 `reboot`，因为关机时写回，并会在断电时丢失最近的日志——而这是机器人的实际关机方式。所以服务日志按构造就是尽力而为的，持久的记录是 `/var/lib` 下的更新历史，它逐条 `fsync` 且完全不经过 `/var/log`。

这是有意的安排而非缺口——`architecture.md` §8.2 把更新日志设计成幸存下来的东西。备选方案，如果某块板子确实需要持久服务日志，是一条命令和一个已知代价：

```bash
sudo systemctl disable --now armbian-zram-config
```

日志随后落到 eMMC 上并磨损它。在 `info` 级别和上面的上限之下，体积很小，这正是选择这些级别的原因——但这是逐板决策，不是要机群级更改的默认值。

## 版本，供支持用

问题总是"当时在跑什么？"，而这台机器人上它有**两个答案同时存在**：`updaterd` 无法在更新中途重启自己（`updater-design.md` §4.1），所以更新后几秒内运行中的二进制合法地落后于已安装的发布物。任何报告单一版本号的东西因此都有误导性。如果滞后超过了那几秒，那是故障而非设计——见 `../docs/design/restart-order.md` §7。

`robotctl version` 报告两者并标记不一致。它刻意在 `updaterd` **停机**时也能工作，把这一项作为一行报告而非退出——那正是人们最可能运行它的时刻。`--json` 为支持包提供同样的内容。

版本可从四个独立位置恢复，所以丢失一个不是致命的：

1. **启动日志行**，每个守护进程写的第一件事，以 `warn` 级别以便挺过 `RUST_LOG=warn`：版本、revision、`exe` 路径、pid。`exe` 路径告诉你运行中的进程实际来自哪个发布目录。
2. **`robotctl version`**，通过 IPC。
3. 每个二进制上的 **`--version`**，用于什么都没在运行的时候。
4. 每个发布目录内的 **`version.toml`**，以及 `robotctl update list`。

`revision` 在构建时从 `DUCK_REVISION` 编译进去（CI 设置它；笔记本构建如实报告 `rev unknown, not a CI build`）。编译时，绝不在运行时查 git——交付的机器人没有仓库。
#（注：内容由AI生成）
