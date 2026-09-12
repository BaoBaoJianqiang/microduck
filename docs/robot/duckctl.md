# `duckctl` —— 每个命令

从笔记本与一个机器人对话，无网络且无 ssh。它是手机 app 的替身，也是到达一个从未见过 wifi 网络的机器人的方式。

蓝牙低功耗是它今天到达那里的方式，且名字故意不说明：`mediad` 给机器人一个到达一组不同方法的第二传输，因此工具以它对话的东西命名而非它当前使用的无线电。当 BLE 是唯一答案时它叫 `duck-btctl`。

**永不在机器人上。** 一个版本中没有任何东西依赖它 —— `robotctl` 是交付的工具，且 [`cheatsheet.md`](cheatsheet.md) 有它的命令，其中大多数在下面有一个 `duckctl` 等价物。

## 获取它

从这个仓库的一个克隆运行它：

```bash
cargo run -q -p duckctl -- --name <robot-name> info
```

或安装一次，代价是一个不再跟随分支的快照：

```bash
cargo install --path duckctl
```

```bash
duckctl --name <robot-name> info
```

下面每个命令都以安装形式书写。给它加前缀 `cargo run -q -p duckctl --` 以从克隆运行。

这个工具曾经把自己安装为 `btctl`。如果 `which btctl` 仍找到一个，它是你安装它时的构建且永远不会再变：

```bash
cargo uninstall btd --bin btctl
```

## 找到一个机器人

```bash
duckctl scan
```

```
1 robot(s) advertising the duck service:
  aa:bb:cc:dd:ee:ff duck-c51b — 192.168.1.42, 1 service(s)  ← DUCK_ROBOT

7 other device(s) in range, not listed. …
```

只有机器人，无线电范围内的其他一切被计数而非列出。`--verbose` 展开那个列表，且当你想要的机器人不在第一个里时值得一读。

每个机器人广播它的 IPv4 地址，因此这也是 ssh 地址的来源。不建立连接且不需要 PIN。行上的 `no address` 意味着机器人不在一个网络上；一个完全没有地址的行意味着一个早于机器人广播地址的版本，且 `duckctl wifi status` 仍报告它。

SSID 不在列表中 —— 它装不进一个广播。`duckctl wifi status` 有它，连同信号与两个地址。

单独要地址：

```bash
ssh radxa@$(duckctl ip)
```

`ip` 只打印地址，因此它可以替换。它读取广播，因此它不连接任何东西、不需要 PIN、且约一秒 —— 且答案不过期：`btd` 每五秒重读地址并在它移动时重新广播。

一个绑定到这台机器的机器人经常停止向它广播服务，然后 `ip` 连接并改问 `net.status`。那更慢且需要 PIN，且它总是应答。`--verbose` 说明发生了两者中的哪一个。

一个没有网络地址的机器人被告知该做什么而非报告为空，因为修复是经由无线电且必须是 —— `net.connect` 被设计为经 WebRTC 拒绝。

一个从未被重命名的机器人叫自己 `duck-` 加上从它的序列号派生的四个字符，因此 `duck-c51b`。一个机器人在两个名字下同时被报告的任一半 —— macOS 显示 `radxa-zero3 [duck-c51b]` —— 都作为 `--name` 工作。

一个扫描为 `duck-c51b` 然后在一次连接后只显示为 `radxa-zero3` 的机器人，在一个把名字给了广播但没给适配器的版本上，且客户端缓存了适配器的。更新机器人。那不会清除客户端已经缓存的东西，因此也清除它：Linux 上 `bluetoothctl remove <mac>`，或在 macOS 蓝牙设置中忘记机器人。

完全没有名字时 —— 无 `--name`、无 `DUCK_ROBOT` —— 第一个找到的机器人赢。有一个名字时，两个应答它的机器人是一个错误而非选择：

```
2 robots answer to "radxa-zero3": radxa-zero3, radxa-zero3
```

那发生在一块 bootloader 留空序列号的板子上，因为它随后以其主机名命名，且从同一镜像烧的每块板子都有同一个。从机器人自身重命名一个并使用新名字：

```bash
robotctl system set-name ducky
```

## 控制台

机器人服务一个流式传输其摄像头并驱动它的页面：

```bash
duckctl open
```

找到机器人，然后在浏览器中打开 `http://<address>:8080/`。`--print` 改为给出 URL，给一台没有浏览器的机器或一个脚本；`--port` 给一个用非默认 `mediad --web-port` 启动的机器人。

无需安装且无需服务任何东西 —— `mediad` 嵌入页面，因此一个运行那个守护进程的机器人就是一个有控制台的机器人。

上面有什么：摄像头，旁边带链路的比特率、帧率、丢失与往返；两个手柄与键 `W`/`A`/`S`/`D` 与 `Q`/`E` 来驾驶，在一个手柄的 0.3 m/s 与 1.5 rad/s；在画面上拖拽看向一个点；enable、init、relax、stop 与 shutdown；技能与音库作为菜单；以及 2 Hz 的状态流在 `robot.health` 旁边，那是一个热舵机、空电池或一个运行慢的循环被命名的地方。

`stop` 把页面正在发送的意图清零。**它不是一个紧急停止** —— 这个系统中没有任何东西从浏览器切断舵机电源 —— 且按钮因此是一个普通按钮。

原始 JSON 框、日志，以及一个 WebRTC 对端被拒绝的两个调用在底部的抽屉里。它们证明路由表被咨询而非驱动机器人。

涉及两个端口且只有这个被键入：页面自己在 8443 上到达信令服务器，使用它被服务自的主机。如果页面加载然后说它的信令端口没应答，机器人起来了且你与 8443 之间的某个东西没起来 —— 最常见是防火墙。

摄像头与驾驶控制在 WebRTC 上且不在别的上面，因此一个没有网络地址的机器人没有控制台。先通过无线电把它加入一个 —— 下面的 `duckctl wifi connect` 不需要它自己的网络。

## 永远是同一个机器人

把名字放进环境而非每个命令行：

```bash
export DUCK_ROBOT=duck-c51b
```

把那行放进 `~/.zshrc` 以保留它。下面每个命令然后无需 `--name` 工作：

```bash
duckctl info
```

`DUCK_PIN` 对 `--pin` 做同样的事，一个有自己 PIN 的机器人在每个命令上都需要它：

```bash
export DUCK_PIN=418299
```

对一个不同机器人的一个命令，`--name` 仍赢：

```bash
duckctl --name duck-ffff info
```

要对一个命令忽略默认 —— 一个上面有别人机器人的工作台 —— 把它设为空：

```bash
DUCK_ROBOT= duckctl scan
```

`scan` 标记 `DUCK_ROBOT` 命名的机器人并把它列第一，且每个去寻找它的命令在开始扫描之前说明。

## 身份

```bash
duckctl --name <robot-name> info
```

名字、序列号与运行时间。

```bash
duckctl --name <robot-name> name <new-name>
```

最多 24 字符。它几秒内生效且无需重启，但 Mac 继续服务它早先学到的名字，因此 `scan` 与 macOS 蓝牙设置都滞后。每个之后的命令使用新名字。

一次重命名不跟随 `DUCK_ROBOT`。工具事后说明；变量必须手动更改，否则每个之后的命令都找一个不再应答的名字。

```bash
duckctl --name <robot-name> reboot
```

## Wifi

```bash
duckctl --name <robot-name> wifi status
```

SSID、信号与地址。

```bash
duckctl --name <robot-name> wifi scan
```

花几秒 —— 机器人扫频无线电而非返回上一次扫描。

```bash
duckctl --name <robot-name> wifi connect <ssid> --psk <passphrase>
```

对开放网络省略 `--psk`。加入会把机器人从它所在的网络断开，因此经 wifi 的 ssh 会话断开；那是命令在工作。它可能需要最多 45 秒应答。

```bash
duckctl --name <robot-name> wifi forget <ssid>
```

## 它还好吗

```bash
duckctl --name <robot-name> health
```

控制循环是否健康。

```bash
duckctl --name <robot-name> status
```

版本握手与更新状态。

## 它在运行哪个版本

```bash
duckctl --name <robot-name> version
```

API 版本、版本，以及它构建自的 git 修订。一个 `revision` 为 `null` 意味着该版本是在某人的笔记本上而非由 CI 构建的。

## 更新

与 `robotctl update` 相同的词，因此在机器人上学的一个命令在这里工作。它们每个都接受 `--component <name>` 且默认为 `daemon`，那是机器人今天唯一的组件。

```bash
duckctl --name <robot-name> update check
```

```bash
duckctl --name <robot-name> update status
```

```bash
duckctl --name <robot-name> update versions
```

```bash
duckctl --name <robot-name> update log --limit 20
```

安装花几分钟并随进程打印进度行：

```bash
duckctl --name <robot-name> update apply
```

```
· daemon: preflight
· daemon: downloading 12%
· daemon: downloading 47%
· daemon: verifying
· daemon: swapping
· daemon: health_gate
{
  "outcome": "applied",
  "from": "0.5.1",
  "to": "0.6.0"
}
note: the robot restarts its daemons now, and `btd` about five seconds after this reply — so this
connection drops. That is the update working. Reconnect and run `duckctl update status`:
`last_attempt` carries the outcome of what just ran.
```

apply 后连接断开是更新在工作，非失败。重连并读 `update status`。

一个分支构建、一个精确版本，或 staging 候选：

```bash
duckctl --name <robot-name> update apply --ref my-branch
```

```bash
duckctl --name <robot-name> update apply --version 0.5.1
```

```bash
duckctl --name <robot-name> update apply --staging
```

`--dry-run` 验证一切并在交换前停止。`--ref` 与 `--version` 是替代；要求两者被拒绝。

回去 —— 前一个版本，或一个从 `update versions` 命名的：

```bash
duckctl --name <robot-name> update rollback
```

```bash
duckctl --name <robot-name> update select 0.5.1
```

两者像 apply 一样被门控，因此一个不起来的被回滚。两者都不丢弃任何东西。

一个别人启动的更新的进度，或一个由机器人自己触发的：

```bash
duckctl --name <robot-name> update watch
```

它打印进行中的更新到了哪里，然后跟随的一切。它永不收到一个应答，因此以 Ctrl-C 结束。

## 任何其他东西 —— `call`

```bash
duckctl --name <robot-name> call <method> '<json-params>'
```

参数默认为 `{}`。这些经蓝牙可达但没有自己的包装器，且不带前面的 `duckctl --name <robot-name>` 书写：

| | |
|---|---|
| `call system.services` | 哪些守护进程起来了，以及每个运行哪个版本。 |
| `call pad.status` | 一个手柄是否已绑定，且它是否连接？ |
| `call pad.pair '{"timeout_seconds":30}'` | 绑定一个处于配对模式的手柄。 |
| `call pad.forget '{"mac":"<address>"}'` | 丢弃一个绑定。 |

`call` 等 60 秒要一个应答。上面的更新命令改为等*静默* —— 三分钟完全没有东西到达 —— 这就是为什么它们是运行更新的方式而非 `call update.apply`。

## 全局选项

- `--name <robot-name>` —— 哪个机器人。没有它，`DUCK_ROBOT`；没有那个，第一个找到的赢。值得总是给：它跳过一个慢的回退层，那个层尝试 Mac 上每个已连接的外设，包括耳机。
- `--pin <six-digits>` —— 默认为 `DUCK_PIN`，然后 `000000`。机器人上的 `robotctl system pin` 显示真正的那个。
- `--verbose` —— 打印发送与接收的每一行，并让 `scan` 列出每个设备而非只有机器人。当有东西挂起时第一个该加的。

## 它打印什么

应答以漂亮 JSON 到 stdout，其他一切 —— 进度、诊断、无线电看到了什么 —— 到 stderr。因此 `duckctl ... info > reply.json` 把两者分开，且一个来自机器人的 JSON-RPC 错误仍以非零退出。进度行以 `·` 开头且每行一个，因此 `update apply > outcome.json` 把它们留在屏幕上并把结果留在文件中。

一个命令是一个连接：它找到机器人、必要时配对、证明 PIN、询问、然后断开。

每个命令在一段静默后而非一个固定总数后放弃，因此一个慢的更新永不被切断且一个停止应答的机器人在几秒内被报告。一个*断开*的链路被立即报告而非等完，且在一次 apply 后它说明：重启正是把连接拿下的东西。一个进行中的更新在任一情况下存活：机器人拉取，因此它在无人观看时继续，且之后的 `update status` 说明它进展如何。

## 什么被拒绝

电机控制（`robot.move`、`robot.head`、`robot.enable`、`robot.stop`、`robot.init`、`robot.relax`）、高速率遥测（`robot.subscribe`）、一个人必须有意为之的两个更新命令（`update.pin`、`update.resetToGolden`）以及配对 PIN（`system.pairingPin`、`system.setPairingPin`）被 `btd` 自己拒绝且永不到达守护进程。它们以错误代码 14 "not available over Bluetooth" 回来。

那是一个安全边界而非一个缺失特性，且每个拒绝在 `btd/src/route.rs` 中旁边都有它的原因 —— [`app-path-design.md`](../design/app-path-design.md) §3.1 是设计。
那些命令是机器人上的 `robotctl`。

## 当它找不到机器人

```bash
duckctl --verbose scan
```

一个空列表 —— 不是一对耳机 —— 指向 Mac 而非机器人：蓝牙关了，或终端从未被授予蓝牙权限。

一个机器人不在其中的列表指向机器人。它在一个可能自己被错过的扫描响应中广播它的名字，因此一个无名字无服务被报告的设备是一个可能的机器人；`--name` 仍连接到一个。如果 macOS 显示机器人已配对但一次连接或第一次读取挂起，绑定是半途而废的：

```bash
sudo pkill bluetoothd
```

在 macOS 蓝牙设置中忘记机器人做同样的事。在机器人自身上，`journalctl -u btd -b` 说明 GATT 应用是否注册过。
