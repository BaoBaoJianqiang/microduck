# 速查表 — 开发板

只在由 [`install-dev.md`](install-dev.md) 设置的板上才有意义的命令。机器人日常需要的一切都在 [`cheatsheet.md`](cheatsheet.md) 中。

## 开发通道

安装分支最近在 CI 上构建的内容：

```
sudo robotctl update apply --ref <branch> daemon
```

```
sudo robotctl update apply --ref main daemon
```

`--version` 改为固定一个确切的 release。除非你真的是指"去 stable"，否则给出其中一个。

**没有 `--ref` 的 `apply daemon` 安装最新的 *stable* release，在开发板上这通常是降级。** 它不是"安装最新的东西"；它是"安装 stable 通道提供的东西"。在分支合并后立即，那个 stable release 仍然比你一直在测试的一切都旧——而且如果它早于现在板上有单元文件的守护进程，其 `ExecStart` 指向旧 release 不包含的二进制文件，重启失败，更新回滚。那是门在工作，但导致它的命令看起来像是显而易见的那个。

标签 `daemon-dev-<branch>` 随分支移动，因此没有版本号可复制。版本*内部*每个构建保持唯一——`0.1.0-dev.42.c719ec8`——因此同一分支的两个构建永远不会混淆。`--ref main` 是板回到主线而不离开开发通道的方式；普通的 `apply daemon` 离开它，因为预发布排序在其 release 之下，并且没有单独的退出步骤。

合并不是立即发布的：CI 必须先构建 `main`，然后 `--ref main` 才能解析到它。

```
gh run list --branch main
```

## 发布候选

`release.yml` 发布到 staging 且还没有人晋升的内容——canary 机器人在晋升前应该运行的：

```
sudo robotctl update apply --staging daemon
```

```
sudo robotctl update apply --staging --version 0.3.0 daemon
```

候选像任何 release 一样用 release 密钥签名，并携带它将被晋升的版本。使其在没有标志的情况下不可达的是它被标记为预发布，而普通的 `apply` 跳过那些，因此没有机器人会漂移到没有人验证过的构建上。`--staging` 是该过滤器的唯一选择加入，它适用于那一个命令，之后不会留下任何开启的东西。

## 更新之后 — 咬人的部分

- **`robotd`、`configd` 和 `padd` 在更新期间重启。`updaterd` 和 `btd` 在它回复后 5 秒重启**——第一个不能在更新中途重启自己，第二个可能正在携带回复。因此 `btd` 修复在几秒钟后生效，无需手动步骤。重新连接它就在那里。
- **如果这两个重启中有一个没有发生，下一次 `updaterd` 启动会修复它。** 除了 `updaterd` 本身，它报告不一致而不是重启自己。对那个再运行一次 apply：它回答 `already_current`，命名没有运行它的守护进程，并安排重启。`sudo systemctl restart updaterd` 手动做同样的事情。
- **运行早于 0.4.0 的 `updaterd` 的板没有这些**，并保持两者都在旧二进制文件上，直到你重启它们。一次更新修复它，只有之后的更新才会表现正常。
- **如果你要求板已经有的版本，`robotctl update apply` 报告 `already_current` 且不安装任何东西**——但它不再是惰性的。它检查哪些守护进程正在运行该 release，并重启那些没有运行的，在 `stale` 中命名它们。因此当修复看起来不存在时，它*是*要找的命令：要么它修复它，要么 `stale` 为空且修复从未在那个 release 中。

症状是一个肯定已安装且肯定不工作的修复。询问每个守护进程正在运行哪个 release：

```
robotctl health
```

`units` 块为每个守护进程打印一行，带有其进程启动的 release，以及当这与已安装的不一致时命名重启的警告。`build unknown (old)` 意味着该守护进程早于教会它说话的 release——重启它就会回答。

如果守护进程真的过时了，重启它——这不应该是必要的，因此值得阅读日志了解为什么：

```
sudo systemctl restart configd
```

`updaterd` 是永远不会修复自己的那个：

```
sudo systemctl restart updaterd
```

不需要编辑板的 `updater.toml`——重启集来自 release 携带的单元。`../design/restart-order.md` 是完整的序列，逐步进行。

## 从笔记本电脑 — 在这里构建，在板上安装

完全跳过 CI：在你的机器上构建并通过 ssh 安装，大约一分钟。

```bash
scripts/dev-push.sh radxa@<board>
```

结果是一个普通的门控更新，因此上面的重启陷阱仍然适用。[`dev-push.md`](dev-push.md) 有设置、容器构建、`--dry-run`、对低于 0.5.0 的板的第一次推送，以及失败时该怎么做。

## 从笔记本电脑 — 用手中的手柄驾驶

你手中的手柄，台架上的机器人，两者上都没有安装任何东西：`padd` 是一个普通客户端，因此它可以从克隆针对转发的 socket 运行。

先停止机器人上的那个，否则两个进程会争夺摇杆：

```bash
sudo systemctl stop padd
```

转发 socket 并保持打开：

```bash
ssh -L /tmp/robotd.sock:/run/robotd.sock radxa@192.168.1.42
```

然后从这个克隆，在另一个终端中：

```bash
cargo run -p padd -- --socket /tmp/robotd.sock
```

`systemctl start padd` 把机器人自己的放回去。这也是 `padd` 的标志值得拥有的地方——`--max-linear`（m/s）、`--max-angular`（rad/s）、`--max-head`（弧度）和 `--deadzone`，它存在是因为模拟摇杆很少精确停在零，没有它机器人会爬行。单元用默认值运行，因此尝试其他值意味着自己运行二进制文件，在这里或在板上：

```bash
sudo -u padd /opt/robot/daemon/current/bin/padd --max-linear 0.25
```

## 从笔记本电脑 — `duckctl`

通过蓝牙低功耗到达机器人，没有网络也没有 ssh：[`duckctl.md`](duckctl.md) 有每个命令。
#（注：内容由AI生成）
