# 部署：操作系统级配置

状态：草稿 · 日期：2026-07-28 · 负责人：pierre

这里放的是属于*机器人镜像*、而不是某个具体服务的配置。各服务自己的 unit 文件就放在对应服务旁边（`updater/systemd/`、`robotd/systemd/`）；凡是机器人全局相关的，都集中在这里。

| | |
|---|---|
| `updater.toml` | 出厂时随机器人一起带的配置，装到 `/etc/robot/updater.toml` |
| `trusted_keys/` | 发布用的公钥——也就是信任锚点，装到 `/etc/robot/trusted_keys/` |
| `journald.conf.d/10-robot.conf` | journald 的持久化与大小上限 |

最后这一项要说明，而且下面这些是**实测**过的，不是想当然：本镜像的 `/var/log` 是一个 zram 设备，所以即便设了 `Storage=persistent`，journald 拿到的也只是一个内存里的目录。正常重启之后日志还在；但断电会丢掉最近的日志——而机器人实际上就是断电关机的。所以真正能留下来的只有 `/var/lib` 下那份更新历史——这正是 `architecture.md` §8.2 当初设计它要承担的角色。

> 只是想把开发板跑起来的话，看 [`docs/robot/install-dev.md`](../docs/robot/install-dev.md)，那是简短流程。下面讲的是信任链、各文件最终落到哪、以及日志去了哪儿。

## 快速开始

三种做法，按敲键盘的多少排序。这一节之后的内容是同一套事情的"带原因版"——只有当某处跟你想的不一样时才需要回去读，动手之前不用看。

### 开发板，仓库私有——也就是当前情况

从一份 clone 出发，一条命令搞定，分步说明见
[`docs/robot/install-dev.md`](../docs/robot/install-dev.md)：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
./scripts/provision-board.sh radxa@192.168.1.42
```

几个可选参数：`--no-dev-key` 表示这块板只收发布版；`--ref BRANCH` 从指定分支部署；`--local` 直接把当前 clone 里的 `provision.sh` 发过去而不是去拉，这样就能测对 `provision.sh` 的未推送改动。

`team.dev.pub` 提交在 [`dev-key/`](dev-key/)，不放进 `trusted_keys/`——把开发密钥带到板子上*本身就是*一次主动选择，所以它不能随每台机器人出厂。脚本默认发的是仓库里那份；`--dev-key PATH` 可以换。

### 在板子上操作，没有 clone

三条命令，第一条在你本机跑，也就是上面 `provision-board.sh` 帮你做的事：

```bash
scp deploy/dev-key/team.dev.pub radxa@192.168.1.42:/tmp/
```

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
curl -fsSL -H "Authorization: Bearer $DUCK_TOKEN" https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/provision.sh -o /tmp/provision.sh && sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/provision.sh
```

`provision.sh` 会按顺序跑 `setup-board.sh`、`migrate-network.sh`、`install.sh`，先给十秒警告，然后重启——你的 SSH 会话到此结束——之后它自己继续跑完。它会先把开发密钥从 `/tmp` 拷出来，因为 `/tmp` 在重启后就没了；要是密钥还留在那儿，你会得到一块"看着部署成功、其实并不是开发板"的板子，过几周才暴露——表现为 `--ref` 被拒，看起来像是发布版坏了。

重新登录之后：

```bash
robotctl health
```

如果等你回来它还在跑，盯着它：

```bash
sudo tail -f /var/lib/robot/provision.log
```

这份日志记的是"没人盯着的那半段"，而且刻意做成文件而不是 journal：journald 的持久化要靠被装的发布版里的 drop-in 来配，所以刚好在这段时间里 journal 可能还是纯内存的。只有密钥真的装上了，日志结尾才会出现 `DEV BOARD`——所以你拿到的是哪类板子是可以查的，不用靠记忆：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

无论哪条路径都不用 `newgrp robot`，这是故意的，不是漏了：`robot` 组在重启前就已经建好，所以你重新登录的会话里已经带着它了。

### 普通用户，仓库公开

不用 token，也不带开发密钥。

```bash
curl -fsSL https://raw.githubusercontent.com/pollen-robotics/microduck/main/scripts/provision.sh -o /tmp/provision.sh && sudo sh /tmp/provision.sh
```

```bash
robotctl health
```

即便是这种情况也用下载而不是管道：后半段是在重启之后才跑的，所以磁盘上必须留一个文件。它拒绝管道，省得你卡在半路。

### 手动一步步来

`DUCK_NO_REBOOT=1` 让 `provision.sh` 在重启前停下，并告诉你接下来该跑什么。如果你想看每一步的状态块依次刷过去，就用这个：

```bash
sudo DUCK_NO_REBOOT=1 DUCK_TOKEN="$DUCK_TOKEN" sh /tmp/provision.sh
```

```bash
sudo reboot
```

```bash
sudo /usr/local/sbin/robot-provision
```

这三个脚本也都能单独跑，`provision.sh` 自己只是个不带逻辑的薄编排。一个个来的话，顺序是：`setup-board.sh`、`migrate-network.sh`、重启、这两个再来一遍、`install.sh`——最后还要 `newgrp robot`，因为这条路径上没人替你在重启前把组建好。

## 这些命令到底在干什么

三个脚本分开，是因为它们管的事不一样。`setup-board.sh` 是 OS 层引导——设备树 overlay、ONNX Runtime——很少动，且需要重启。`install.sh` 装的是一个带签名的守护进程发布版，每次更新都走它；要是把两者混到一起，每次更新都得重新折腾一遍启动配置。`migrate-network.sh` 是个不会长久的过渡：它存在纯粹是因为 Armbian 官方镜像带的是 netplan，等哪天我们自己出一个内置 NetworkManager 的镜像，它就删掉、而不是继续维护。

**板子出厂不带 NetworkManager**，所以中间这一步省不掉。Armbian 的 headless 镜像跑的是 netplan + `systemd-networkd` + `wpa_supplicant`，而 `configd` 是通过 D-Bus 驱动 NM 的——所以在迁移跑完之前，`robotctl net status` 会报 `Unavailable`，蓝牙那边也没法配 wifi。为什么用 NM 而不是镜像自带那套，见 [`../docs/design/app-path-design.md`](../docs/design/app-path-design.md) §2。

`provision.sh` 按顺序串起这三个脚本，并且保管那些必须跨过重启的状态——token、开发密钥路径，还有它用来判断你是否真重启了的 boot id。它自己不带任何部署逻辑，这是故意的：三个生命周期不同的脚本，不该揉成一个没法分别拆掉的脚本。

`provision-board.sh` 再往外一层，跑在你本机、不在机器人上。它存在是为了填中间那段缝：`provision.sh` 要重启、然后自己跑完，这没错；但从外面看，就是 ssh 掉线、然后你得猜什么时候能连回去。它会等板子就绪、流式输出日志、最后停在健康报告上——于是部署成了一条命令、连续输出，而不是三条带空档的命令。

重启这一下是它做的，它调用的那几个脚本则刻意从不重启。这不矛盾：那些脚本是单一用途的，可能跑在一台正在干别的活的机器人上，所以它们谁都没法知道一次重启会打断什么。而 `provision-board.sh` 只会在一块正在被部署的板上跑，那时重启就是下一步、不是打断。

代价是后半段是无人值守的，那里可能出的问题不是"失败"而是"死循环"——所以有两道闸。恢复 unit 在动任何活之前先把自己禁用，于是最多只自动重试一次：跑挂了的阶段 2 会留下一块能去查的板子，而不是一块一遍遍撞同一面墙的板子。而 `migrate-network.sh` 只有当 NetworkManager 已经接管了 wifi 时才会再跑一次，意思是切换已经成了、这次只是为了撤掉那个兜底；如果兜底真的触发、把 netplan 恢回来了，再无人值守地切一遍就会重新武装它、再以同样方式失败、重启、再绕一圈——所以它会在日志里说明，并且不去动 wifi。unit 文件留在磁盘上、被禁用，作为"跑过什么"的留痕。

token 要用三次，但只有前两次是部署的一部分：拉这些脚本、拉发布版，然后是永久性的——`updaterd` 之后每次检查更新都从 systemd drop-in 里读 `GITHUB_TOKEN`。把 `DUCK_TOKEN` 传下去，是为了让更新在部署*之后*也能用，不只是部署期间能用；`provision.sh` 跑完会告诉你它把两个副本删了哪个、留了哪个。一块没 token 的板子装是装得上，但装完以后再也没法拉更新了——而那正是 `updaterd` 的大头用途。

它还在第一阶段就建了 `robot` 组，这就是上面流程里没有 `newgrp robot` 的唯一原因。`install.sh` 也会建这个组，做法没错，但时机太晚——等它跑的时候，你的 shell 是在组存在之前启动的，而进程的组在启动时就定死了。把它提前到重启之前，你重新登录时那个会话就已经带着组了。

**关于 token，以及为什么"URL 错了"是误诊。** `raw.githubusercontent.com` 对没带凭据的私有路径返回的是 **404**、不是 401，所以漏一个 header 看起来跟拼错 URL 一模一样。这里其实有两个不同的 token：一个是你 shell 里用来拉脚本的，另一个是 `updaterd` 用来拿发布资产的——[后者](#-while-the-repository-is-private-a-robot-needs-a-token) 是一个 systemd drop-in，比 shell 活得久。两者在 `provision.sh` 走到 `install.sh` 时碰头：把 `DUCK_TOKEN` 传下去，正是写那个 drop-in 的动作。

**`setup-board.sh`** 幂等，且从不自己重启。它修的一个否则非常难诊断的问题：Armbian 默认带 `overlay_prefix=rk35xx`，但 RK3566 跟 RK3568 共用设备树 overlay，文件名是 `rk3568-*.dtbo`。prefix 错的时候加载器什么都找不到，板子照常启动，于是压根没有 `/dev/ttyS2`。`armbian-config` 的 overlay 编辑器因为同一个原因会崩，所以这里直接改文件。

⚠ 一次会把 `/boot/{Image,dtb,uInitrd}` 重新指过去的内核升级可能把它撤掉。一台板在 `apt upgrade` 之后突然认不出电机了，就需要重跑这个。

⚠ 它的 `kernel console` 状态行在脚本已经修好、只是运行中的内核还旧的时候，会显示 `until the reboot`——`/proc/cmdline` 不重启就不会变。那一行上的别的字才是真发现。

**`migrate-network.sh`** 跑**两次**，重启前后各一次——`provision.sh` 两边都帮你跑了，第二次为什么重要值得说一说。第一次跑的时候会装一个开机兜底：如果 `wlan0` 拿不到 IPv4，就自动恢复 netplan，于是切坏了只搭一次重启、不搭一根串口线；第二次跑是为了撤掉这个兜底。不撤掉的话，以后哪次开机 wifi 只是慢了一点，板子就会被回退回去。它从 netplan 自己读 SSID 和密钥——netplan 生成了 `/run/netplan/wpa-wlan0.conf` 就用那个，否则读 `/etc/netplan/*.yaml` 里的 `access-points:` 段——这样走同一条 wifi 的 SSH 才不会断。两个都读不到的话，它**什么都不动**，只打印 `nmcli` 命令让你手动建档案。

**`install.sh`** 只依赖 `curl` 和 coreutils，别的都不要——`tar` 和 `zstd` 都已经链进 `updaterd` 了。幂等，且从不覆盖已经存在的 `/etc/robot/updater.toml`。仓库私有时它是两条命令、而不是 `curl … | sudo sh`，因为 header 要加在 fetch 上、`sudo` 自己不透传变量、管道两者都带不了。它接收的全部东西都是环境变量，因为它通常是从管道里跑的，那里用 flag 很别扭：

| | |
|---|---|
| `DUCK_TOKEN` | 私有仓库的 token——fetch 和发布资产都要用 |
| `DUCK_REPO` | 仓库，给 fork 或测试仓库用 |
| `DUCK_REF` | 读脚本和可信密钥的分支；想可复现就固定到一个 tag |
| `DUCK_CONFIG_REF` | `updater.toml` 从哪儿来。默认是正在装的发布版的 tag，`DUCK_REF` 不会改它——一个配置字段只有从它自己那个版本起的二进制才认，所以拿一个分支的配置配最后一个稳定版二进制，正是 `updaterd` 最后拒绝启动的来由。只有在你拿一个能读懂某配置变更的构建、去测那个配置变更时，才设它。 |
| `DUCK_DEV_KEY` | `team.dev.pub` 的路径——把这台变成开发板（见下文） |
| `DUCK_FORCE_REINSTALL` | 用发布版自带的 `updaterd` 重装到一个已经在线的发布版之上 |

**它自己怎么把自己装上去。** 更新需要更新器，而更新器是随更新一起来的。出路是一个裸 `updaterd` 二进制，作为 `updaterd-bootstrap-aarch64` 资产发布，因为一块全新的板连开 `.tar.zst` 的 `zstd` 都没有。然后它跑的是*普通*引擎——同样的校验、解压、原子替换、写 journal——所以 store 落地的状态正好是常驻守护进程期待的那样，不存在一条"只走 bootstrap"的路径跟以后更新的行为发生漂移。这段时间里 `on_apply` 和 `health` 被强制关掉，因为这些 unit 就在被装的那个发布版*里面*；`updaterd install` 在一个发布版已经*在线*之后就拒绝运行，所以永远不可能在一台正常工作的机器人上悄悄把自动回滚禁掉。`--from <dir>` 改成从本地文件装：离线和工厂路径，也是 CI 在发布前验证一个发布版用的手段。

### 把它变成开发板，好让 `--ref <branch>` 生效

从笔记本直接推上来的构建（`scripts/dev-push.sh`）受同样两个条件门控——它用同一把开发密钥签名，所以拒绝分支构建的板子也会拒绝那些。

一块板拒绝分支构建是两道关一起把：`allow_dev_keys` 是 false，且一把可信密钥只有当文件名以 `.dev.pub` 结尾才算开发密钥。两道都要，它们是独立检查，只做一道会留下一块仍然拒绝分支构建的板子——还带着一个看起来像"发布版损坏"的签名错误。`DUCK_DEV_KEY` 一把两道都过。

它在改任何东西之前先校验：文件得存在、得看起来像一把 minisign 公钥、`updater.toml` 里得已经有一行 `allow_dev_keys` 可供翻转——这键是顶级的，所以直接追加一行会落到"最后一个 `[table]` 里头"。不管源叫什么名字，它都把密钥装成 `team.dev.pub`，因为 `.dev.` 这个中缀才是分类依据；落到别的名字下的密钥会被当作*发布版*密钥信任，分支构建于是会被当作已评审过而被接受。

`team.dev.pub` 提交在 [`dev-key/`](dev-key/)，刻意放在 `trusted_keys/` 之外——[`dev-key/README.md`](dev-key/README.md) 说明了为什么这样是安全的。

收尾报告在这项打开时会输出 `DEV BOARD`，并给出撤销它的两条命令。
要出货的机器人，绝对不要这么干。


### ⚠ 仓库私有时，机器人需要 token

私有仓库的发布资产没凭据是拿不到的——`releases/download/...` 这个 URL 即便带了凭据也会 404，所以引擎改走 release API 去解析资产。`updaterd` 从环境里读 `GITHUB_TOKEN`，在板子上这意味着一个 systemd drop-in、而不是 shell 里 export 一下就完了。

开发者的板子上这么搞没问题；客户机器人上**不行**：镜像里塞一个全舰队通用的凭据，一旦泄漏就只能重新刷机才能轮换——而这正是分层签名密钥要避免的那种失败。

所以 `install.sh` **只在传入了 `DUCK_TOKEN` 时**才写 drop-in——mode 600，而且会特意大声报告这件事。客户机器人从公开的构件仓库装、不传 token，所以根本走不到那条路上。少了它，`updaterd` 会装上、会跑起来、却一个更新都拉不下来——而那正是它的大头用途。

所以构件托管是个开放问题、不是已经定了的——
[`../docs/design/updater-design.md`](../docs/design/updater-design.md) §6.1 列了几个选项。便宜的做法是另开一个只放已签名构件的公开仓库：让一个构件能安全地被分发靠的是签名、不是藏，源代码可以继续保持私有。

### 信任链

1. 走 TLS 到 `raw.githubusercontent.com` 拉 `install.sh`、`updater.toml` 和公钥。
   这些不能从发布版来：在密钥就位之前没法校验任何东西。
2. 走 TLS 到 `github.com` 拉 bootstrap `updaterd`。**还没校验。**
3. 这个二进制用 (1) 的密钥校验清单和构件，凡不是它们签的一律拒。
4. 安装器再把 bootstrap 二进制的 `sha256` 跟
   `current/bin/updaterd`（来自已校验的构件）比一遍。摘要相等就说明 (2) 那个二进制是真的。`release.yml` 断言两者是同一份字节，所以不匹配是真发现、不是打包的小毛病。

其余一切——两个 unit 文件、journald drop-in、`robot.conf`——都从*已安装的*发布版里取、不从仓库拉，所以它们都是对过签名的那一份。

剩下落在 GitHub 自身上的那点信任，也是脚本来源的地方；步骤 (4) 是把那扇窗缩小、而不是关掉。想一点都不沾的安装，可以用 `--from` 配上手工搬进来的文件。

### 无人值守更新

`updaterd` 本来就是个带计时器的常驻进程——`updater.toml` 里的 `check_interval`——所以不用往 cron 加任何东西。计时器允许装什么由 `auto_apply` 决定：

| | |
|---|---|
| `off` | 永不；可用性记日志，强制发布则大声报告 |
| `mandatory` | **出厂默认**——只装 `min_supported` 下限声称当前版本不该继续用的发布 |
| `all` | 每一个可用发布 |

一台要跟踪 `staging`、每个候选版都装的 canary 或台架机器人：

```bash
sudo sed -i 's/^auto_apply = .*/auto_apply = "all"/' /etc/robot/updater.toml
```

```bash
sudo sed -i 's/^tag_prefix     = .*/tag_prefix     = "daemon-staging-v"/' /etc/robot/updater.toml
```

```bash
sudo systemctl restart updaterd && journalctl -u updaterd -b | tail -20
```

启动后 60 秒做第一次检查，之后每 `check_interval` 一次。`auto_apply = "all"` 在启动时会以 `warn` 记一条，所以"为什么没人动它、它自己重启了"这种问题，任何日志级别下都能从 journal 里查出答案。

别去用 cron 或 systemd timer 顶替。`robotctl update apply` 刻意绕过那个"已知有问题"的守卫——重试某个发布版的运维者可能已经修好了原因——所以拿计时器去驱动它，会把那个绕过也继承下来、失去保护，一个坏发布版就变成一个每个间隔都重下重写 eMMC 的无限 apply/rollback 循环。详见 `updater-design.md` §8.1.1。

不需要维护窗口，这是故意的。无人值守 apply 就是一次普通 apply：预检在任何网络访问*之前*先问 `robotd` 现在重启是否安全、有没有远程会话在线，所以一台正在走或正在推流的机器人会拒绝、并等下一间隔再试。`safeToRestart` 是"现在是不是坏时机"的更好答案，胜过一个时钟。

### 各文件最终落到哪

```
/etc/robot/updater.toml                 配置；更新从不碰它
/etc/robot/trusted_keys/release-*.pub   信任锚点
/opt/robot/daemon/releases/<version>/   发布版树
/opt/robot/daemon/current -> releases/<version>
/etc/systemd/system/*.service           发布版自带的每个 unit，从中复制出来
/usr/lib/sysusers.d/robot.conf          建 `robot` 组
/var/lib/robot/updater/                 锁、更新日志、启动计数
/usr/local/bin/robotctl -> current/bin/robotctl
/usr/local/sbin/robot-provision         部署，重启后接着跑
/var/lib/robot/provision.log            无人值守那半段干了什么
/etc/systemd/system/robot-provision.service   部署完成后即被禁用
```

unit 文件是**复制**的、不是通过 `current` 软链的：要是走软链，每次更新它们都会在 systemd 眼皮底下变，而回滚之后 systemd 看到什么，取决于上次 `daemon-reload` 时碰巧在线的是哪个发布版。`robotctl` *是*软链，因为它是运维者手动调的工具、不是 systemd 缓存的文件。

变更操作只允许 root：`allow_uids`/`allow_gids` 在 `updater.toml` 里刻意留空。属于 `robot` 组只是让一个进程能"跟"`updaterd` 说话——查状态、看日志——仅此而已。`btd` 的 uid 在它存在时加入允许列表，因为"可以从 app 转发一个更新请求"是个比"在 robot 组里"更窄的主张。

⚠ **直接跑 `install.sh` 的话，第一次 `robotctl health` 会失败，但安装本身没问题。**
两个套接字都是 `root:robot`、mode 0660，`install.sh` 把运维者放进 `robot`——但进程的组在它启动时就定死了，所以跑安装的那个 shell 不在它刚刚加入的那个组里。同一个 shell 里一条命令即可：

```bash
newgrp robot
```

把一个组加到运行中的进程上没有 API，root 也不行，所以安装器做什么都修不了启动它的那个 shell——这就是为什么 `provision.sh` 改在第一阶段就建组、靠重启来生效。直接路径上，`install.sh` 会在收尾报告里打印这条命令，`robotctl` 失败时也会点名它、而不是把你引到 `systemctl status`——后者会显示两个完全健康的守护进程。

## 日志去了哪、什么能在重启后留下来

每个守护进程都把日志写到 **stderr**，由 systemd 收进 journal。级别由 `RUST_LOG` 决定，在每个 unit 里设（`info`）。

两份记录，持久性刻意不一样：

| | 在哪 | 重启后留 | 断电后留 | 上限 |
|---|---|---|---|---|
| 服务日志 | journald | 仅当配置过（见下文） | **不留**——`/var/log` 是 zram，见下文 | `SystemMaxUse=200M` |
| **更新历史** | `/var/lib/robot/updater/update-log.jsonl` | **留** | **留** | 200 条 |

更新历史刻意不放进 journal。它在引擎位于 `/var/lib` 下的 `state_dir` 里，每条在追加时都 `fsync`，重写走的是"临时文件加 rename"的原子操作、并对父目录 `fsync`
（`updater/src/journal.rs`、`updater/src/fsutil.rs`）。所以"这台机器人装过什么、它身上发生过什么"即便在 journal 是易失的机器人上也能留下来，可以用 `robotctl update log` 查、也可以直接以 JSON 行的形式从磁盘读。这一性质由测试验证、不是假设。

服务日志需要本目录下这个 drop-in。装上之后：

```bash
sudo systemctl restart systemd-journald
```

然后确认不止一本次启动被保留下来——这才是真正的验收，且只有在*真重启之后*才有意义：

```bash
journalctl --list-boots
```

两行或以上意味着上一次启动还能读得到。只有一行，或
`no persistent journal was found`，说明日志还是纯内存的。

读某个服务、上一次启动：

```bash
journalctl -u robotd -b -1
```

### RAM 日志这条告诫——实测过，并决定不要持久化

本镜像的 `/var/log` 是一个 **zram 设备**，所以 `Storage=persistent` 让 journald 拿到的是一个本身就在内存里的目录。这是在板子上实测确认的、不是推断：

```bash
findmnt /var/log
```

干净的 `reboot` 之后还在，因为关机会写回；断电会丢最近的——而机器人实际上就是断电关机的。所以服务日志在构造上就是尽力而为，而能留下来的持久记录是 `/var/lib` 下那份逐条 `fsync`、且从不经过 `/var/log` 的更新历史。

这是有意为之、不是缺口——`architecture.md` §8.2 当初就让更新日志做"那个能留下来的东西"。如果哪块板确实需要持久的服务日志，替代方案是一行命令加一个已知代价：

```bash
sudo systemctl disable --now armbian-zram-config
```

之后日志会落到 eMMC 上、磨损它。在 `info` 级别和上面的上限下量很小，这正是当初选这些级别的原因——但这是单板决定，不是该全网改的默认值。

## 版本，给支持用

问题永远是"当时跑的是什么？"，而在这台机器人上**同时有两个答案**：`updaterd` 没法在更新途中重启自己（`updater-design.md` §4.1），所以更新之后的几秒里，运行中的二进制合法地落后于已安装的发布版。任何只报一个版本号的东西因此都是误导。如果这个落差超出那几秒，那是故障、不是设计——见 `../docs/design/restart-order.md` §7。

`robotctl version` 两个都报、并标记不一致。它刻意在 `updaterd` **已下线**时也能工作，把那当成一行、而不是直接退出——那正是最可能有人去跑它的时候。`--json` 输出同样的内容，供支持包用。

四处独立位置都能恢复版本，所以丢一处不致命：

1. **启动日志行**，每个守护进程写下的第一行，级别 `warn`、所以在
   `RUST_LOG=warn` 下也留得住：版本、revision、`exe` 路径、pid。`exe` 路径告诉你运行中的进程到底来自哪个发布版目录。
2. **`robotctl version`**，走 IPC。
3. **`--version`**，每个二进制都有，用于什么都没在跑时。
4. **`version.toml`**，每个发布版目录里都有，以及 `robotctl update list`。

`revision` 在构建时从 `DUCK_REVISION` 编译进去（CI 设它；笔记本构建会老实报 `rev unknown, not a CI build`）。是编译时、不是运行时去查 git——出货的机器人没有仓库。
