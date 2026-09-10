# `duckctl` — 每个命令

从笔记本电脑与机器人交谈，没有网络也没有 ssh。它是手机应用的替身，也是到达从未见过 wifi 网络的机器人的方式。

蓝牙低功耗是它今天到达那里的方式，名字故意不说明这一点：`mediad` 给机器人一个到达不同方法集的第二传输，因此该工具以它交谈的东西命名，而不是它当前使用的无线电命名。当 BLE 是唯一答案时，它是 `duck-btctl`。

**永远不要在机器人上。** release 中没有任何东西依赖它——`robotctl` 是随附的工具，[`cheatsheet.md`](cheatsheet.md) 有它的命令，其中大多数在下面有 `duckctl` 等价物。

## 获取它

从这个仓库的克隆运行它：

```bash
cargo run -q -p duckctl -- --name <robot-name> info
```

或者安装一次，代价是一个不再跟随分支的快照：

```bash
cargo install --path duckctl
```

```bash
duckctl --name <robot-name> info
```

下面的每个命令都以安装形式编写。用 `cargo run -q -p duckctl --` 作为前缀，改为从克隆运行它。

这个工具曾经将自己安装为 `btctl`。如果 `which btctl` 仍然找到一个，它是你安装它时的构建，并且永远不会再改变：

```bash
cargo uninstall btd --bin btctl
```

## 找到机器人

```bash
duckctl scan
```

```
1 robot(s) advertising the duck service:
  aa:bb:cc:dd:ee:ff duck-c51b — 192.168.1.42, 1 service(s)  ← DUCK_ROBOT

7 other device(s) in range, not listed. …
```

仅限机器人，无线电范围内的其他一切被计数而不是列出。`--verbose` 展开那个列表，当你想要的机器人不在第一个列表中时值得阅读。

每个机器人广播其 IPv4 地址，因此这也是 ssh 到的地址的来源。没有建立连接，也不需要 PIN。行上的 `no address` 意味着机器人不在网络上；根本没有地址的行意味着来自机器人广播地址之前的 release，而 `duckctl wifi status` 仍然报告它。

SSID 不在列表中——它不适合广告。`duckctl wifi status` 有它，连同信号和两个地址。

单独获取地址：

```bash
ssh radxa@$(duckctl ip)
```

`ip` 打印地址而不打印其他东西，因此它可以替换。它读取广告，因此它不连接任何东西，不需要 PIN，大约需要一秒——而且答案不是过时的：`btd` 每五秒重新读取地址并在它移动时重新广告。

绑定到这台机器的机器人经常停止向它广告服务，然后 `ip` 连接并改为询问 `net.status`。那更慢且需要 PIN，并且它总是回答。`--verbose` 说明两者中发生了哪一个。

没有网络地址的机器人被告知该怎么做，而不是报告为空，因为修复是通过无线电的，而且必须是——`net.connect` 被设计为通过 WebRTC 拒绝。

从未被重命名的机器人称自己为 `duck-` 加上从其序列号派生的四个字符，因此是 `duck-c51b`。在两个名称下同时报告的机器人的任何一半——macOS 显示 `radxa-zero3 [duck-c51b]`——都可以作为 `--name`。

扫描为 `duck-c51b` 然后在一次连接后仅为 `radxa-zero3` 的机器人在一个给广告但不给适配器名称的 release 上，而客户端缓存了适配器的。更新机器人。那不会清除客户端已经缓存的东西，因此也清除它：在 Linux 上 `bluetoothctl remove <mac>`，或者在 macOS 蓝牙设置中忘记机器人。

根本没有名字——没有 `--name`，没有 `DUCK_ROBOT`——第一个找到的机器人获胜。有了名字，两个回答它的机器人是错误而不是选择：

```
2 robots answer to "radxa-zero3": radxa-zero3, radxa-zero3
```

这发生在引导加载程序留下其序列号空白的板上，因为它然后以其主机名命名，而从一个镜像刷新的每个板都有相同的名字。从机器人本身重命名一个并使用新名字：

```bash
robotctl system set-name ducky
```

## 控制台

机器人提供一个流式传输其摄像头并驾驶它的页面：

```bash
duckctl open
```

找到机器人，然后在浏览器中打开 `http://<address>:8080/`。`--print` 改为给出 URL，用于没有浏览器的机器或脚本；`--port` 用于以非默认 `mediad --web-port` 启动的机器人。

没有什么要安装的，也没有什么要服务的——`mediad` 嵌入页面，因此运行那个守护进程的机器人就是有控制台的机器人。

上面有什么：带有链接的比特率、帧率、丢失和往返的摄像头；两个手柄和按键 `W`/`A`/`S`/`D` 和 `Q`/`E` 来驾驶，在游戏手柄的 0.3 m/s 和 1.5 rad/s 下；在画面上拖动来看一个点；enable、init、relax、stop 和 shutdown；技能和语音库作为菜单；以及 2 Hz 的状态流旁边的 `robot.health`，那是热舵机、扁平电池或运行缓慢的循环被命名的地方。

`stop` 将页面正在发送的意图归零。**它不是紧急停止**——这个系统中没有任何东西从浏览器切断舵机电源——并且按钮因此是一个普通按钮。

原始 JSON 框、日志，以及 WebRTC 对等方被拒绝的两个调用在底部的抽屉中。它们证明路由表正在被咨询，而不是驾驶机器人。

涉及两个端口，只有这个被键入：页面自己到达 8443 上的信令服务器，使用它被服务的主机。如果页面加载然后说其信令端口没有回答，机器人起来了，你和 8443 之间的东西没有——最常见的是防火墙。

摄像头和驾驶控制在 WebRTC 上，没有别的，因此没有网络地址的机器人没有控制台。先通过无线电加入一个——下面的 `duckctl wifi connect` 不需要它自己的网络。

## 永远是同一个机器人

把名字放在环境中而不是每个命令行上：

```bash
export DUCK_ROBOT=duck-c51b
```

把那行放在 `~/.zshrc` 中以保持它。下面的每个命令然后在没有 `--name` 的情况下工作：

```bash
duckctl info
```

`DUCK_PIN` 对 `--pin` 做同样的事情，有自己 PIN 的机器人在每个命令上都需要它：

```bash
export DUCK_PIN=418299
```

对于针对不同机器人的一个命令，`--name` 仍然获胜：

```bash
duckctl --name duck-ffff info
```

要为一个命令忽略默认值——一个上面有别人机器人的台架——将其设为空：

```bash
DUCK_ROBOT= duckctl scan
```

`scan` 标记 `DUCK_ROBOT` 命名的机器人并首先列出它，每个去寻找它的命令在开始扫描之前都会说明。

## 身份

```bash
duckctl --name <robot-name> info
```

名字、序列号和正常运行时间。

```bash
duckctl --name <robot-name> name <new-name>
```

最多 24 个字符。它在几秒钟内生效，不需要重启，但 Mac 继续提供它早先学到的名字，因此 `scan` 和 macOS 蓝牙设置都滞后。以后的每个命令使用新名字。

重命名不跟随 `DUCK_ROBOT`。工具之后说明；变量必须手动更改，否则以后的每个命令寻找一个不再回答的名字。

```bash
duckctl --name <robot-name> reboot
```

## Wifi

```bash
duckctl --name <robot-name> wifi status
```

SSID、信号和地址。

```bash
duckctl --name <robot-name> wifi scan
```

需要几秒钟——机器人扫描无线电，而不是返回上一次扫描的结果。

```bash
duckctl --name <robot-name> wifi connect <ssid> --psk <passphrase>
```

开放网络省略 `--psk`。加入会将机器人从它所在的网络断开，因此通过 wifi 的 ssh 会话会掉线；那是命令在工作。它可能需要长达 45 秒才能回答。

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

版本握手和更新状态。

## 它在运行哪个 release

```bash
duckctl --name <robot-name> version
```

API 版本、release，以及它构建自的 git 修订。`revision` 为 `null` 意味着 release 是在某人的笔记本电脑上构建的，而不是由 CI 构建的。

## 更新

与 `robotctl update` 相同的词，因此在机器人上学到的命令在这里工作。它们中的每一个都接受 `--component <name>` 并默认为 `daemon`，这是机器人今天拥有的唯一组件。

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

安装需要几分钟，并在进行时打印进度行：

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

apply 后连接掉线是更新在工作，不是失败。重新连接并读取 `update status`。

分支构建、确切版本或 staging 候选：

```bash
duckctl --name <robot-name> update apply --ref my-branch
```

```bash
duckctl --name <robot-name> update apply --version 0.5.1
```

```bash
duckctl --name <robot-name> update apply --staging
```

`--dry-run` 验证一切并在 swap 之前停止。`--ref` 和 `--version` 是替代方案；要求两者都被拒绝。

回去——上一个 release，或从 `update versions` 命名的一个：

```bash
duckctl --name <robot-name> update rollback
```

```bash
duckctl --name <robot-name> update select 0.5.1
```

两者都像 apply 一样被门控，因此一个不起来的会被还原。两者都不丢弃任何东西。

别人开始的更新的进度，或由机器人自己触发的一个：

```bash
duckctl --name <robot-name> update watch
```

它打印进行中的更新到达了哪里，然后是接下来的一切。它永远不会收到回复，因此以 Ctrl-C 结束。

## 其他任何东西 — `call`

```bash
duckctl --name <robot-name> call <method> '<json-params>'
```

参数默认为 `{}`。这些可以通过蓝牙到达但没有自己的包装器，并且在编写时没有在它们前面加上 `duckctl --name <robot-name>`：

| | |
|---|---|
| `call system.services` | 哪些守护进程起来了，以及每个运行的 release。 |
| `call pad.status` | 游戏手柄是否绑定，以及它是否连接。 |
| `call pad.pair '{"timeout_seconds":30}'` | 绑定一个处于配对模式的手柄。 |
| `call pad.forget '{"mac":"<address>"}'` | 丢弃绑定。 |

`call` 等待 60 秒以获得答案。上面的更新命令改为等待*沉默*——三分钟完全没有任何东西到达——这就是为什么它们是运行更新的方式，而不是 `call update.apply`。

## 全局选项

- `--name <robot-name>` — 哪个机器人。没有它，`DUCK_ROBOT`；没有那个，第一个找到的获胜。总是值得给出：它跳过一个缓慢的回退层，该层尝试 Mac 上每个已经连接的外围设备，包括耳机。
- `--pin <six-digits>` — 默认为 `DUCK_PIN`，然后是 `000000`。机器人上的 `robotctl system pin` 显示真实的那个。
- `--verbose` — 打印发送和接收的每一行，并让 `scan` 列出每个设备而不是仅机器人。当有东西挂起时首先添加的东西。

## 它打印什么

回复作为漂亮的 JSON 转到 stdout，其他一切——进度、诊断、无线电看到了什么——转到 stderr。因此 `duckctl ... info > reply.json` 保持两者分开，并且来自机器人的 JSON-RPC 错误仍然以非零退出。进度行以 `·` 开头，每行一个，因此 `update apply > outcome.json` 将它们留在屏幕上并将结果保存在文件中。

一个命令是一个连接：它找到机器人，必要时配对，证明 PIN，询问，然后断开连接。

每个命令在一段沉默后放弃，而不是在固定的总数后，因此缓慢的更新永远不会被切断，而停止回答的机器人在几秒钟内被报告。*掉线*的链接立即被报告，而不是等待出来，并且在 apply 之后它说明：重启是使连接断开的原因。进行中的更新在两者中都幸存：机器人拉取，因此它在没有人观看的情况下继续，之后的 `update status` 说明它进行得如何。

## 什么被拒绝

电机控制（`robot.move`、`robot.head`、`robot.enable`、`robot.stop`、`robot.init`、`robot.relax`）、高速率遥测（`robot.subscribe`）、一个人必须有意的两个更新命令（`update.pin`、`update.resetToGolden`）以及配对 PIN（`system.pairingPin`、`system.setPairingPin`）被 `btd` 本身拒绝，永远不会到达守护进程。它们以错误代码 14 返回，"not available over Bluetooth"。

那是一个安全边界，而不是缺失的功能，并且每个拒绝在 `btd/src/route.rs` 中旁边都有其原因——[`app-path-design.md`](../design/app-path-design.md) §3.1 是设计。那些命令是机器人上的 `robotctl`。

## 当它找不到机器人时

```bash
duckctl --verbose scan
```

空列表——不是一副耳机——指向 Mac 而不是机器人：蓝牙关闭，或者终端从未被授予蓝牙权限。

机器人不在其中的列表指向机器人。它在扫描响应中广告其名字，而扫描响应本身可能被遗漏，因此报告为没有名字和没有服务的设备是一个可能的机器人；`--name` 无论如何连接到一个。如果 macOS 将机器人显示为已配对但连接或第一次读取挂起，绑定是半成品：

```bash
sudo pkill bluetoothd
```

在 macOS 蓝牙设置中忘记机器人做同样的事情。在机器人本身，`journalctl -u btd -b` 说明 GATT 应用是否根本注册了。
#（注：内容由AI生成）
