# 速查表 —— dev 板

只在由 [`install-dev.md`](install-dev.md) 设置的板子上有意义的命令。机器人日常需要的一切在 [`cheatsheet.md`](cheatsheet.md) 中。

## dev 通道

安装一个分支最后在 CI 上构建的东西：

```
sudo robotctl update apply --ref <branch> daemon
```

```
sudo robotctl update apply --ref main daemon
```

`--version` 固定一个精确版本。给它们之一，除非你真的是指"去 stable"。

**无 `--ref` 的 `apply daemon` 安装最新的 *stable* 版本，在 dev 板上通常是一次降级。** 它不是"安装最新的东西"；它是"安装 stable 通道提供的东西"。一个分支合并后，那个 stable 版本仍比你一直在测试的一切都旧 —— 且如果它早于一个现在在板子上有 unit 文件的守护进程，它的 `ExecStart` 指向一个旧版本不包含的二进制，重启失败，更新回滚。那是门在工作，但导致它的命令看起来像明显的那个。

标签 `daemon-dev-<branch>` 随分支移动，因此没有版本号要复制。*内部*的版本每个构建保持唯一 —— `0.1.0-dev.42.c719ec8` —— 因此同一分支的两个构建永远不会混淆。`--ref main` 是一块板子回到主线而不离开 dev 通道的方式；一个普通的 `apply daemon` 离开它，因为一个预发布排序低于其发布，且没有单独的退出步骤。

一次合并不立即发布：CI 必须构建 `main`，`--ref main` 才能解析到它。

```
gh run list --branch main
```

## 发布候选

`release.yml` 发布到 staging 且还没人晋升的东西 —— 一只金丝雀机器人在晋升前应该运行的：

```
sudo robotctl update apply --staging daemon
```

```
sudo robotctl update apply --staging --version 0.3.0 daemon
```

一个候选像任何版本一样用发布密钥签名，且携带它将被晋升到的版本。让它无该标志不可达的是它被标记为预发布，且一个普通 `apply` 跳过那些，因此没有机器人漂移到一个没人验证过的构建上。`--staging` 是那个过滤器的唯一选择加入，它适用于这一个命令，且之后不留下任何开启的东西。

## 更新之后 —— 咬人的那部分

- **`robotd`、`configd` 与 `padd` 在更新期间重启。`updaterd` 与 `btd` 在它应答后 5 秒重启** —— 第一个不能在更新中途重启自己，第二个可能正在承载应答。因此一个 `btd` 修复几秒后就活了，无手动步骤。重连它就在那里。
- **如果那两个重启中有一个没发生，下一次 `updaterd` 启动修复它。** 除了 `updaterd` 自己，它报告不一致而非重启自己。对那个再跑一次 apply：它应答 `already_current`，命名没在运行它的守护进程，并调度重启。`sudo systemctl restart updaterd` 手动做同样的事。
- **一块运行早于 0.4.0 的 `updaterd` 的板子没有这些**，且把两者都留在旧二进制上直到你重启它们。一次更新修复它，且只有那次更新之后的行为正常。
- **`robotctl update apply` 报告 `already_current` 且什么都不安装**，如果你要求一块板子已经有的版本 —— 但它不再是惰性的。它检查哪些守护进程在运行那个版本并重启那些不在的，在 `stale` 中命名它们。因此当一个修复看起来缺席时，它*确实*是该求助的命令：要么它修复了，要么 `stale` 为空且修复从未在那个版本中。

症状是一个肯定安装了且肯定不工作的修复。问每个守护进程在运行哪个版本：

```
robotctl health
```

`units` 块为每个守护进程打印一行，带其进程被启动自的版本，且当与安装的不一致时命名重启的警告。`build unknown (old)` 意味着那个守护进程早于教它说话的版本 —— 重启它它就会应答。

如果一个守护进程真的过时了，重启它 —— 这不该是必要的，因此值得读日志看为什么：

```
sudo systemctl restart configd
```

`updaterd` 是那个永不修复自己的：

```
sudo systemctl restart updaterd
```

无需编辑板子的 `updater.toml` —— 重启集来自版本交付的 unit。`../design/restart-order.md` 是完整序列，一步步。

## 从笔记本 —— 在这里构建，在板子上安装

完全跳过 CI：在你的机器上构建并通过 ssh 安装，约一分钟。

```bash
scripts/dev-push.sh radxa@<board>
```

结果是一次普通的门控更新，因此上面的重启陷阱仍适用。[`dev-push.md`](dev-push.md) 有设置、容器构建、`--dry-run`、对低于 0.5.0 的板子的第一次推送，以及失败时该做什么。

## 从笔记本 —— 用手中的手柄驾驶

手柄在你手中，机器人在工作台上，且两者上都没安装任何东西：`padd` 是一个普通客户端，因此它可以从一个克隆对一个转发的 socket 运行。

先停止机器人上的那个，否则两个进程争抢摇杆：

```bash
sudo systemctl stop padd
```

转发 socket 并保持打开：

```bash
ssh -L /tmp/robotd.sock:/run/robotd.sock radxa@192.168.1.42
```

然后从这个克隆，在另一个终端：

```bash
cargo run -p padd -- --socket /tmp/robotd.sock
```

`systemctl start padd` 把机器人自己的放回去。这也是 `padd` 的标志值得有的地方 —— `--max-linear`（m/s）、`--max-angular`（rad/s）、`--max-head`（弧度）与 `--deadzone`，它存在因为模拟摇杆很少精确停在零，没有它机器人会蠕动。unit 用默认值运行，因此尝试其他值意味着自己运行二进制，在这里或板子上：

```bash
sudo -u padd /opt/robot/daemon/current/bin/padd --max-linear 0.25
```

## 从笔记本 —— `duckctl`

经蓝牙低功耗到达机器人，无网络且无 ssh：[`duckctl.md`](duckctl.md) 有每个命令。
