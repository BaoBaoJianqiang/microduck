# 速查表

`robotctl`，在机器人上运行。这里的每个命令都来自随附它的分支上的 `--help`，而不是来自记忆。

只读命令不需要权限。任何**改变**机器人的东西都需要 `sudo`（或者对于 `configd` 是 `--allow-user`/`--allow-group` 中的用户，对于 `updaterd` 是 `updater.toml` 中的 `allow_uids`/`allow_gids`）。

分支构建、发布候选和更新后的重启陷阱在 [`cheatsheet-dev.md`](cheatsheet-dev.md) 中——它们需要开发板。通过蓝牙从笔记本电脑到达同一个机器人，没有网络也没有 ssh，是 [`duckctl.md`](duckctl.md)。

## 在机器人上 — `robotctl`

### 要运行的第一件事

```
robotctl version
```

每个守护进程正在*运行*什么对*已安装*什么，加上当它们不一致时的警告。在相信任何其他诊断之前运行这个——更新后一个提供旧代码的守护进程看起来和你刚刚发布的修复中的 bug 完全一样。参见下面的"更新之后"。

```
robotctl health
```

硬件和软件在一个报告中。当机器人不健康或不可达时以非零退出，因此它可以门控脚本——热舵机或被固定的组件被报告，而不是被判断，并且不影响退出代码。`--json` 用于支持包。

### 观看循环

```
robotctl monitor
```

客户端要求的东西旁边是实际应用的东西，当它们不同时命名原因——安全不断地夹紧东西，而"摇杆向前且机器人仍然不动"在没有那个的情况下是不可读的。限制被拼写出来而不是命名：`deadman — no intent arrived recently, velocity zeroed`。

在帧上还有：每个关节针对它被命令的内容进行测量，IMU 的投影重力和从中得出的跌倒判定，以及作为轨迹的已实现循环速率，因此已经恢复的卡顿仍然可见。投影重力是这个流上唯一的 IMU 量——直立大约是 `[0, 0, -1]`，它是 `fallen` 被决定的依据。陈旧读取计数器和它们有意义所针对的比率住在 `robotctl health` 中。

标题的最后一行是机器人的状况而不是其行为：电池组的电压（伏特和分数）、最热的舵机和板自己的温度。它来自 `robot.health`，每两秒轮询一次，因为这些都不在状态流上——而且它是任何出错的东西被命名的地方，无论是 `unhealthy: control loop at 43.9 Hz`、`degraded: no robot on the motor bus after 3 attempts` 还是 `orientation frozen — 25 stale reads`。最后一个在这一行上，在帧上其他任何地方都没有：一个已经停止融合的板继续回答总线，因此没有任何东西出错，上面的重力向量无限期地保持一个看似合理的姿态。

0% 是 `BATTERY_EMPTY_V`，那是 `robotd` 让机器人坐下并切断电源的地方，因此这个数字是倒计时而不是量表——30% 时黄色，15% 时红色。尚未获取的读数显示 `batt not read yet` 而不是 `0.00 V`，这是正常运行时间的第一秒和无法回答的总线两者看起来的样子。即使根本没有状态，这一行也被绘制，而这是它最重要的情况：一个舵机电源关闭的板永远不会完成一个控制 tick，因此没有任何东西到达流上，原因只在健康回答上。

底部边框命名已加载的策略——`.onnx` 文件，以及是否配置了站立网络——因为 `walk` 是两个具有不同步态的 release 都报告的模式。没有策略的机器人说明这一点，策略无法加载的机器人改为说明那一点，这是流的 `held` 无法区分的。

在右侧，**机器人按它站立的样子被绘制**——与策略训练所针对的相同视觉模型，由测量的关节角度摆姿势，并由 IMU 的重力向量倾斜。一条腿以错误方式折叠、一个头俯冲到地板上和一只鸭子侧躺都只是关节表中的数字；它们中的每一个在这里都是显而易见的。它默认开启，只要终端足够宽（大约 110 列——表先来，机器人拿走剩余的）就会出现。**`d`** 关掉它；**`[`** 和 **`]`**，或 `←`/`→`，环绕它。

每当 ToF 正在传递帧时，它看到的东西被绘制到同一个场景中——黄色表示命中，绿色表示地板——针对机器人自己的身体进行深度测试，因此喙后面的点被它隐藏。这就是使"它看到的是我的手，还是它自己"可回答的原因。它不需要按键：点在帧到达时出现，在它们停止时消失。

在它下面，当列足够高时，**机器人去过哪里的地图**：来自足部接触和 IMU 的里程计轨迹，用盲文。面板永远不会增长——世界随着轨迹缩放，因此整个路径保持在帧内。`+` 是它开始的地方，`●` 是带有短射线表示其航向的机器人，屏幕向上是它启动时的航向。没有磁力计，因此这是相对运动并且会漂移；它回答"它是否走了一个圈"而不是"它在哪里"。

`q` 退出；`↑`/`↓` 在太短而无法容纳所有关节的窗口上滚动关节列表；`u` 在度和弧度之间切换角度；`t` 打开 [ToF 矩阵](#the-tof-sensor-tofd)；`d` 切换机器人视图，`[` / `]` 环绕它；`p` 打开手柄的原始输入流——来自游戏手柄的每个 evdev 报告，带有它们之间的间隔，这是停滞的无线电可见的唯一地方（[配对游戏手柄](pair-a-gamepad.md#when-it-drops-while-you-are-driving)）。屏幕上的角度是度——关节、头部和偏航率。重定向或管道传输时，它改为每个 tick 打印一行，因此 `> run.log` 和 `| grep FALLEN` 表现正常，并且无论屏幕设置为什么，那些数字都保持弧度。关节向量在 `--json` 中，它携带整个状态，每行一个对象：

```
robotctl monitor --json --hz 50 > run.jsonl
```

### 配置机器人

```
sudo robotctl configure
```

`/etc/robot/robotd.toml` 上的交互式编辑器：守护进程知道的每个键，首先是功能开关（策略开/关、walk/roller、limp-fall、音频、宠物检测、电池关机、摄像头和视频质量……），当前值对默认值，一行文档。SPACE 切换，ENTER 键入值，`u` 将键恢复为其默认值。黄色的值（标记为 `•`）是这个机器人偏离默认值的键；其他一切是内置默认值，`unset` 的可选项显示它们解析为什么 `(auto)`。

三个值得信任的属性：

- **它不能与守护进程不一致。** schema、默认值和验证来自 `robotd` 解析文件所用的同一个 crate，键列表由测试固定完整——守护进程中的新 `[section]` 会在这里显示，否则构建失败。
- **它不能吃掉你的文件。** 来自其他 release 的注释、排序和键原封不动地幸存；只有你更改的键被写入。恢复一个键会移除它（以及附加到它的注释），而不是固定默认值，因此文件保持为*决定*的列表，而不是默认值的副本。
- **它不能写入 `robotd` 拒绝启动的文件。** 每次保存首先通过守护进程自己的加载器验证，原子地（临时文件 + 重命名），并带着原因被拒绝。

守护进程在启动时读取文件一次，因此保存提供重启——重启那些读取你更改的内容的：`[media]` 是 `mediad`，其他一切是 `robotd`。需要 `sudo`，因为文件是 root 拥有的——没有它编辑器以只读打开并在第一次写入时说明。`--file` 将它指向别处用于台架副本。随附的 `deploy/robotd.toml` 保持为*为什么*每个旋钮存在的参考；这是用于翻转它们的。

#### 视频质量

```
sudo robotctl configure
```

设置 `media.quality`——`1080p30`、`720p30`、`720p15` 或 `360p30`——并接受它提供的重启。`media.camera` 关闭改为流式传输测试图案，这是没有摄像头的板想要的：WebRTC *控制*通道骑在视频轨道上，因此无法启动的管道会两者都失去。`media.bitrate` 跟随质量，除非你设置它；单位是比特每秒。

`media.congestion_control` 是该部分中的另一个旋钮，它是移动 CPU 的那个：`disabled` 去掉带宽估计器，它是 `mediad` 中最大的单一消费者（一个核心的 7.6%，而捕获是 0.3%），并使 `media.bitrate` 成为速率而不是起点。它失去了适应性——在降级的链路上，画面停滞而不是速率下降。

720p30 是管道被测量的梯级；不成立的梯级运行得更慢而不是失败。`robotctl monitor` 在底部边框报告已实现的速率，当它低于要求的 90% 时以黄色显示，旁边有 `of <target>`。应用了什么：

```
journalctl -u mediad -b | grep streaming
```

#### 你自己的策略

你不需要切割 release 来尝试网络。在板上的 `/etc/robot/robotd.toml` 中将 `robotd` 指向你自己的 `.onnx`：

```toml
[policy]
walk = "/home/radxa/my_walking.onnx"
stand = "/home/radxa/my_stand.onnx"
```

```
sudo systemctl restart robotd
```

你的路径在更新中幸存——release 替换二进制文件和它随附的策略，而不是指向别处的文件。删除这些行以回到 release 携带的那些。

无法加载的策略报告**不健康**，`robotctl health` 和 `monitor` 的底部边框都命名原因。策略必须具有的形状，以及加载时还检查什么，在 [`../design/robotd-design.md`](../design/robotd-design.md) §2.3 中。

### 关节通电 (`robotd`)

```
sudo robotctl robot init
```

```
sudo robotctl robot relax --yes
```

`init` 给关节通电并在大约两秒内斜坡到起始姿势——**它移动每个关节**，因此让机器人在其支架上。它不需要策略，它是游戏手柄的 Start 在前往驾驶的路上所做的，因此手动它是一个台架的事情。

`relax` 切断电源，如果没有东西支撑它，**机器人会坍塌**，这就是它想要 `--yes` 的原因。它是除了拔插头之外回到 limp 的唯一方式：再次按 Start 停止策略并保持机器人站立，`robot.stop` 在仍然站立的同时将速度归零。

两者都通过 `robotd`，它拥有电机总线。`robotd init`——子命令——对于守护进程没有运行的机器人仍然存在，并且它需要守护进程停止，因为一个 UART 上的两个写入者会破坏彼此的回复：

```
sudo systemctl stop robotd && sudo /opt/robot/daemon/current/bin/robotd init && sudo systemctl start robotd
```

`init` 无论机器人是否跌倒都工作——默认情况下跌是一个*报告*（在 `robotctl monitor` 中可见），而不是门，与原型匹配。在 `robotd.toml` 中设置 `[safety] fall_limp` 或 `fall_recover` 的板武装门：在那里跌倒的机器人进入 limp 并拒绝 `init`/`enable`/技能，直到它被扶起来。

### 游戏手柄 (`configd`)

```
robotctl pad status
```

```
sudo robotctl pad pair
```

```
sudo robotctl pad pair 78:86:2E:BB:13:28
```

```
sudo robotctl pad forget 78:86:2E:BB:13:28
```

配对是每个手柄一次，有自己的页面——[`pair-a-gamepad.md`](pair-a-gamepad.md)：哪个按钮将手柄置于配对模式，添加第二个手柄而不忘记第一个，以及当它无法绑定时该怎么做（`/etc/bluetooth/main.conf` 中的 `Privacy` 设置比其他任何东西更经常是答案）。

`padd.service` 从启动运行并驱动任何连接的手柄，因此配对是唯一步骤。映射是原型的，因此肌肉记忆延续：

| 控制 | 做什么 |
| --- | --- |
| 左摇杆 | 驾驶：前进/后退和横移 · 头部：头部偏航和俯仰 · 身体姿势：向上和蹲下 |
| 右摇杆 | 驾驶：转弯 · 头部：颈部俯仰和头部翻滚 · 身体姿势：俯仰和翻滚 |
| **Start** | 切换策略——在它开启之前没有东西移动 |
| **Y** / 三角形 | 头部模式：摇杆摆姿势头部（身体保持不动） |
| **B** / 圆形 | 身体姿势模式：摇杆倾斜和蹲下站立的机器人 |
| **A** / 十字 | 地面拾取 |
| **X** / 方形 | 翻滚——一次前滚；按住以链接翻滚 |
| **LB / RB** | 左/右踢 |
| **DPad-Down** | 坐下 ↔ 站立 |
| **RT / LT** | 嘴（任一触发器）——RT 也嘎嘎叫；LT 在按住时骑在"wheee"上 |
| **DPad-Up**，按住 3 秒 | 切换驾驶模式，walk ⇄ roller |
| **Select**，按住 2 秒 | 坐下，然后关机 |

没有停止按钮：松开摇杆，机器人站立，如果 `padd` 死亡，`robotd` 的 deadman 停止它。在 roller 机器人上（`robotd.toml` 中的 `mode = "roller"`），摇杆自动采用 roller 整形——不对称推/刹，没有横移——A 触发蹲下。其他技能随之而来：坐下、踢和翻滚在轮子上也工作，正如原型所拥有的。

**按住 DPad-Up 在两者之间切换**，用于当你刚刚给鸭子装上轮子或取下它们时：机器人嘎嘎叫一次表示 walking 或两次表示 roller，回到其起始姿势，在那里加载该模式的策略并再次驾驶——几秒钟，全程有扭矩，没有重启。`robotd.toml` 不被触及，因此重启回到配置的模式；用 `robotctl configure`（或 `[policy] mode`）使其持久。它是按住而不是按下，因为 D-pad up 在驾驶时很容易靠到。

`pad status` 分别回答两个问题，因为连接的手柄和死掉的驱动程序从外面看起来完全一样：

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

要用非默认限制驾驶，先停止服务，否则两个进程会争夺摇杆：

```
sudo systemctl stop padd
```

```
sudo -u padd /opt/robot/daemon/current/bin/padd --max-linear 0.25
```

当链接本身是嫌疑对象时，实时观看它——`robotctl monitor`，然后 `p`。那在没有机器人的情况下也工作：在舵机未通电或 `robotd` 停止的板上，monitor 打开手柄块而不是拒绝。要在窗口上获得判定而不是实时画面，从这个仓库的克隆复制测量：

```
scp scripts/pad-link-test.sh radxa@<board>:/tmp/
```

已经在 `padd` 日志中的掉线——不需要手柄，它立即回答：

```
sudo sh /tmp/pad-link-test.sh --history
```

或者现在测量它，在整个两分钟内保持摇杆移动：

```
sudo sh /tmp/pad-link-test.sh
```

它针对内核自己对每个掉线的原因计数，并计时手柄输入报告之间的间隔——`padd` 看不到的故障，那里链接保持向上，机器人在陈旧的命令上行走。[`pair-a-gamepad.md`](pair-a-gamepad.md#when-it-drops-while-you-are-driving) 读取数字。

当两个板用同一个手柄表现不同时，差异在它下面的栈中：

```
scp scripts/pad-stack-report.sh radxa@<board>:/tmp/
```

```
sudo sh /tmp/pad-stack-report.sh
```

内核、BlueZ、控制器固件、LE 或 BR/EDR，以及手柄自己的固件修订——被打印并保存到 `/tmp/pad-stack-<host>-<when>.log`。`--fingerprint` 只打印两个板之间必须匹配的值，用于 `diff`。[`pair-a-gamepad.md`](pair-a-gamepad.md#is-this-board-running-the-same-stack-as-that-one) 有比较。

### 声音

```
robotctl quack
```

区分鸭子的最响亮方式：每个机器人的语音库从其 SoC 序列号生成（`sounds ensure-bank`，由每个 release 安装运行），因此回答的机器人——用只属于它自己的声音——是你 SSH 进去的那个。没有声音的机器人——音频关闭，或没有库——说明这一点而不是打印 🦆，因此沉默总是意味着错误的鸭子。机器人在 `robotd` 起来时也打招呼，在关机前啄别，并且——如果你要求它——当麦克风听到它的头被抓挠时咕咕叫。那个默认在两种模式下都关闭（`audio.pet_detect = true` 打开它；分类器随附在 release 中）：常开版本在每次偶然触碰时咕咕叫，变得烦人。启动问候有自己的开关，对于整天重启守护进程的任何人：

```
sudo robotctl configure
```

设置 `audio.greet = false` 并接受它在保存时提供的重启。那使那一次嘎嘎叫静音，而单独留下触发器和麦克风，这是 `audio.enabled = false` 不做的。音频硬件 bring-up——codec 驱动、overlay、混音器——是 `setup-board.sh` 的音频部分，每个板一次。

要试听声音或手动重新生成库，release 携带生成器：

```
/opt/robot/daemon/current/bin/sounds show
sudo /opt/robot/daemon/current/bin/sounds ensure-bank --force
```

`sounds theremin` 试听*实时*合成器——特雷门琴演奏的声音，由脚本化的手在 ToF 自己的帧率下扫过驱动。`--out sweep.wav` 写入它而不是播放它，这是你在面前没有机器人的情况下听到声音变化的方式。

### 鸭子合唱

```
robotctl chorale
```

一个房间里的两只鸭子一起唱四声部作品；更多的加入它们发现已经在进行的东西。运行直到 Ctrl-C。`--off` 停止一个。

**默认关闭**——`robotd.toml` 中的 `[chorale] accept`，并且它必须在应该参与的每只鸭子上设置。合唱移动嘴和头，因此因为另一只鸭子走进来而开始动画的鸭子会在做没有人要求的动作。关闭也意味着*不可见*：没有选择加入的鸭子不会在空气中放任何东西，而不是礼貌地拒绝。

它如何工作，按问题出现的顺序：

- **没有人负责。** 两只鸭子看到相同的信标，较低的 id 指挥，因此没有选举可以输，也没有必须到达的消息。
- **没有共享时钟。** 板没有 NTP 也没有 RTC 协议，因此指挥的节拍计数器*就是*时基：它每拍在 BLE 广告中 bump 一个字节，新值的到达就是强拍。跟随者在大约 25 拍上平均相位，这将无线电的抖动带入合奏需要的 ±20 ms 内。
- **声部是算出来的，不是分配的。** 最低的鸭子唱低音。指挥广播名册，每个人在它上面重放相同的座位——这就是阻止两只鸭子在它们各自能看到房间的不同子集时唱同一声部的东西。
- **加入不改变任何人的声部。** 到达的鸭子拿走空闲的声部。离开的鸭子保持它的*座位*——它的声部只是不被唱，就像在合唱团中有人走出去一样——因为在作品中途重新安排幸存者是唯一值得避免的事情。

读数在声部一确定就命名它，因此鸭子最终唱什么在回滚中幸存：

```
listening for other ducks — Ctrl-C to stop
  singing tenor    with 3 voices
  tenor    bar   12  beat  45.2  3 voices
```

指挥每次表演挑选作品，而表演*结束*——在最后一个音符之后，每个人回到聆听，重新安顿，在呼吸之后唱别的东西。`robotctl chorale --piece 2` 固定这个机器人挑选什么**如果它指挥**（跟随者唱信标命名的东西，因此在每只鸭子上设置它以保证歌曲）；未知 id 被拒绝，带着机器人的目录。Id：1 wistful，2 duck-strut，3 outer-wilds（测试资产，不用于 release）。`robotd` 环境中的 `DUCK_CHORALE_PIECE=<id>` 是常设后备——注意它必须在 **robotd** 上，而不是在 `robotctl` 命令行上。

要在没有任何鸭子的情况下听到编曲，一台机器可以渲染整个合奏：

```
sounds chorale --voices 4                 # 或 --seeds 100,7,42 用于特定的鸭子
sounds chorale --score my-piece.mid       # 任何符号编辑器导出的东西
sounds chorale --rolloff 0                # 用于全范围扬声器，而不是鸭子的
```

乐谱来自 `sounds/scores/*.duckscore`——一种面向行的文本格式，在 `wistful.duckscore` 中有文档，那也是随附的作品——或者 MIDI 文件，这是值得使用的路径：**MuseScore 是乐谱编辑器。** 每个声部一个乐器而不是一个钢琴谱，命名声部，导出 MIDI。声部通过平均音高匹配，因此以顶部谱表优先编写的乐谱仍然将低音放在低音上；名为"Soprano"的音轨被相信超过其音高。

### 演奏鸭子（ToF 特雷门琴）

```
robotctl theremin
```

头部的深度传感器变成乐器：喙前面的手是音高——越近越高——嘴随着音符张开，在范围的顶部张开。运行直到 Ctrl-C，并在退出时放下乐器。`--off` 放下一个客户端留下向上的。

一个里面没有聪明东西的显式模式：当它向上时，可演奏带内最近的返回就是手。将鸭子指向开放空间，它是沉默的；将它指向 40 厘米外的墙，它演奏一个稳定的音符。它坐着、站着或走着演奏——嘴不是任何策略的一部分。

读数的最后一列是**传感器对那一帧说了什么**，它是每个"为什么它停止演奏"的答案：

```
  0.34 m    438.1 Hz   60% ██████    14 usable · 255:38 4*:9 5*:5 1:12
```

有多少个区域携带机器人相信的状态，然后是每个 ST 状态代码的计数，在相信的那些上有 `*`。音符前面的 `~` 意味着它是一个*保持*的音符，桥接传感器丢失，而不是现在测量的东西。

那一列存在是因为它会在一分钟内发现的 bug：ST 将 5 和 9 记录为"range valid"，而只相信那些的构建**在大约 30 厘米处停止看到手**——超过那个，移动的手回来为 4 或 13（*consistency failed*，sigma 太高），携带对于音高完全好的距离。如果可达范围短，将代码添加到 `robotd.toml` 中的 `[theremin] statuses`；如果它在空空气中演奏幻影音符，删除一些。`hold_ms` 是防斩：它骑在闪烁的区域上。

注意 `robotctl monitor` 的 ToF 网格比特雷门琴更严格——它将 5/9 之外的任何东西标记为 `x`，*could not measure*。满是 `x` 的网格不意味着传感器坏了；它意味着它对它拥有的数字持悲观态度。

### ToF 传感器 (`tofd`)

来自头部传感器的 8×8 深度矩阵。`robotctl monitor`，然后 **`t`**：

```
┌ tof VL53L8CX · 15 Hz · 8×8 · 48/64 ranged · 0.12–3.54 m ─────────────┐
│ 0.12 0.15    x 1.44 1.86    · 2.70 3.12                              │
└ · nothing in range · x could not measure · near→far ── seq 412 · 6 ms ┘
```

距离以米为单位，从近暖到远冷着色。两个标记很重要：`·` 是*已测量，范围内没有东西*——自由空间，这是信息——而 `x` 是*无法测量*，它对外面有什么什么都没说。将两者都显示为空白的网格会隐藏差异。

这是传感器自己的帧，不是机器人的：在运动学存在之前没有重投影，这也是使该块成为检查安装角度的正确地方的原因。

`tofd` 拥有传感器，没有其他东西读取总线。它是一个普通服务——`sudo systemctl stop tofd` 是安全的，没有东西依赖它，`monitor` 说"no depth stream"并继续。它区分三件事，因为它们需要不同的修复：

| 块说什么 | 它意味着什么 |
| --- | --- |
| `connecting to tofd…` / `no depth stream` | 守护进程没有运行 |
| `no sensor: …` | `tofd` 起来了；总线上没有任何东西回答（大多数鸭子） |
| `waiting for the first frame…` | 传感器正在测距；它的第一次扫描在大约 66 ms 后 |

要手动查看总线上有什么，或在没有终端 UI 的情况下观看帧：

```
sudo i2cdetect -y -r 3
journalctl -u tofd -b
```

传感器共享 codec 的 I²C 总线，因此 `setup-board.sh` 的音频部分已经 provision 总线本身；ToF 步骤只添加稳定的 `/dev/i2c-pihat` 名称。两代传感器都受支持——VL53L5CX 和 VL53L8CX 在板上可互换，守护进程从读取的 ID 中挑选驱动程序。

### Wifi (`configd`)

```
robotctl net status
```

```
robotctl net scan
```

```
sudo robotctl net connect <ssid> --psk <passphrase>
```

```
sudo robotctl net connect <ssid> --psk-stdin
```

```
sudo robotctl net forget <ssid>
```

`--psk-stdin` 使密码短语不出现在 `ps` 中，`ps` 在命令的生命周期内向盒子上的每个用户显示 `--psk` 参数。在任何共享的东西上更喜欢它。

加入网络**将机器人从它所在的网络断开**，因此通过 wifi 的 ssh 会话会掉线。那是操作在工作。扫描需要几秒钟——它等待无线电扫描，而不是返回上一次扫描的结果。

### 身份和电源 (`configd`)

```
robotctl system info
```

```
robotctl system pin
```

```
sudo robotctl system set-name <name>
```

```
sudo robotctl system set-pin <six-digits>
```

```
sudo robotctl system reboot
```

开箱即用，机器人称自己为 `duck-` 加上从其自己的序列号派生的四个字符，因此从同一个镜像刷新的两个板在手机的蓝牙列表中仍然看起来不同。重命名在几秒钟内通过蓝牙生效——不需要重启——但手机必须再次扫描才能看到它。

PIN 是手机通过蓝牙认证用的。出厂默认是 `000000`，它认证任何读过这个仓库的人。

### 更新 (`updaterd`)

```
robotctl update status
```

```
robotctl update check daemon
```

```
sudo robotctl update apply daemon
```

```
sudo robotctl update rollback daemon
```

```
robotctl update log
```

```
robotctl update show
```

```
robotctl update watch
```

`log` 列出尝试，每行一个，最新的在前；第一列是运行编号。`show` 接受那些数字之一——或什么都不给，用于最近的——并打印那次运行做的一切，然后是同一窗口的日志：

```
run 42 · daemon · 2025-08-27 13:06:40 UTC
  applied 0.1.3 → 0.1.4
  asked for latest, from github.com/pollen-robotics/microduck, onto 0.1.3
  requested by uid=1000 gid=1000 pid=2317

  13:06:41      +1s  manifest     0.1.4 · 184.2 MB · sha256 3f9a1c2b… · signed by release.pub · rev 88efc03
  13:06:41           downloading
  13:07:58   +1m17s  note         downloaded 184.2 MB to /opt/robot/daemon/staging/0.1.4/dl/…
  13:08:02      +4s  note         hash matches; signature verifies against release.pub
  13:08:20     +18s  pre-hook
  13:10:12   +1m52s  hook         hooks/preinstall
                                 │ onnxruntime 1.20.1 already present
                                 │ gstreamer: h264 encode ok
  13:10:12           swapping     0.1.3 → 0.1.4
  13:10:14      +1s  unit         robotd: restart
  13:10:23      +8s  health       the robot reported healthy
  13:10:24           ended        applied 0.1.3 → 0.1.4

  ── journal · 2025-08-27 13:06:40 to 2025-08-27 13:11:24 UTC ──
```

时间是 UTC，下面的日志也是。`+` 列是自上一行以来的间隔，这是你找到那两分钟的方式。

读取日志需要 `robot` 组不携带的权限，因此除非你是 root，否则后半部分回来为空。它在发生时打印 `journalctl` 行；在它前面加 `sudo` 是修复。`--no-journal` 打印那行而不尝试，`--json` 单独给出记录。

组件是 `daemon`——一个覆盖每个二进制文件的组件。`apply daemon` 安装 stable 通道提供的东西；分支构建和发布候选需要 [`cheatsheet-dev.md`](cheatsheet-dev.md)。

### 不下载切换

到板已经解压的东西。不涉及网络：

```
sudo robotctl update select daemon 0.1.4
```

```
sudo robotctl update rollback daemon
```

```
sudo robotctl update reset-to-golden daemon
```

`select` 激活已安装的 release，`rollback` 去到先前安装的那个，`reset-to-golden` 去到永不修剪的已知良好的那个。

以及根本拒绝移动：

```
sudo robotctl update pin daemon 0.1.4
```

```
sudo robotctl update pin daemon
```

第二种形式取消固定。

### 当 `updaterd` 本身无法启动时

上面的一切都通过 `updaterd`，因此当 `updaterd` 是宕机的守护进程时，它们中没有一个工作。检查是哪一个：

```
systemctl status updaterd robotd btd configd
```

然后在没有它的情况下回到 golden：

```
sudo robot-rescue --dry-run
```

```
sudo robot-rescue --reboot
```

`--dry-run` 说明它会做什么并且什么都不改变。没有 `--reboot` 它交换 release 并打印重启命令而不是运行它：每个守护进程通过 `current` exec，因此在它重启之前没有任何东西拾取交换，而站立的机器人应该首先被抓住。

当没有配置 golden 或当 `current` 已经是 golden 时，它拒绝并说明原因——如果守护进程在 golden 本身上失败，回滚不是答案，日志才是：

```
journalctl -b -u robotd -u updaterd -u btd -u configd
```

### 机器人可能已经这样做了

每次启动后三分钟，一个计时器询问 release 是否带来了它的守护进程，如果没有则回退到 golden。因此一个自己重启并且运行比你安装的更旧 release 的机器人可能已经自救了。它做了什么：

```
robotctl update log
```

条目读起来像回滚，在其原因中命名失败的守护进程。要看到正在做出的决定而不是其结果：

```
journalctl -b -u robot-boot-check
```

```
sudo robot-boot-check --dry-run
```

它行动一次。当第一次仍在记录中时，第二次救援被拒绝——`updaterd` 在下次启动时清除那个，因此被拒绝意味着守护进程在 golden 上也没有起来，答案是日志而不是另一次重启。超过它，如果你已经阅读日志并决定：

```
sudo robot-rescue --force --reboot
```

### 三件容易搞错的事情

**`rollback` 需要前驱，但更新创建一个。** 新 provision 的板恰好有一个 release，因此那时的 `rollback` 没有更旧的东西可去并说明。自动回滚*不*受影响：应用 release 将其解压在当前版本旁边，然后才移动 `current`，因此到健康门运行时有两个，你来自的 release 是目标。`rollback_target` 挑选 `current` 之下日志尚未记录为坏的最高已安装版本——因此有一个 release 的板从它进行第一次更新的那一刻起就受到完全保护。

唯一真正不受保护的安装是引导本身，根据定义它之前没有任何东西。`golden` 会覆盖那个，它被故意取消设置直到 1.0.0 存在——因此 `reset-to-golden` 诚实地报告没有配置，而不是做令人惊讶的事情。

**`version` 显示每个组件的实时 release，而不是 release 存储。** 它永远不会列出两个版本，无论解压了多少个。直接询问存储：

```
ls -l /opt/robot/daemon/releases/ /opt/robot/daemon/current
```

**`apply --version` 需要 release 仍然存在于上游；`select` 不需要。** 携带已知坏构建的 release 从 GitHub 删除，因此 `apply --version 0.1.3` 故意失败，而 `select 0.1.3` 在已经解压它的板上仍然工作。不对称是故意的：没有新板可以获取坏 release，而有一个的板保留其逃生舱口。

### 没有网络安装

侧载、工厂安装，或救援一个 `updaterd` 太旧而无法接受修复太旧的 release 的板。参见 [`install-dev.md`](install-dev.md)——它是 `updaterd install --from`，而 `--force` 变体有在你使用它之前值得阅读的条件。

### 日志

```
journalctl -u configd -b --no-pager | tail -40
```

```
journalctl -u btd -f
```

换入 `robotd` 或 `updaterd`。`-f` 跟随；`-b` 仅是这次启动。

启动行携带版本、git 修订和进程启动自的 release 目录，在 `warn`，因此它在任何日志级别幸存。

更新历史故意与日志分开——每个条目在 `/var/lib/robot/updater/` 下被 `fsync`——因此它在日志易失的机器人上幸存：

```
robotctl update log
```

最后二十次运行也在那里保留完整记录，在 `runs/` 下，按发生时写入：

```
robotctl update show 42
```

两者都比交换、回滚和断电活得久，这是这个板上的日志做不到的——`/var/log` 是 zram。如果 `robotctl` 本身是坏的东西，文件是换行分隔的 JSON，用 `cat` 读取正常：

```
sudo cat /var/lib/robot/updater/runs/000042.jsonl
```

### 制表符补全

`install.sh` 在 `/etc/bash_completion.d/` 中设置这个，作为一个向二进制文件询问其自己的补全的加载器——因此它们跟随已安装的 release，而不是在更新添加命令时变得陈旧。对于它没有覆盖的 shell，或者对于你直接从 `target/` 运行的构建：

```
eval "$(robotctl completions bash)"
```

`zsh`、`fish`、`elvish` 和 `powershell` 代替 `bash` 工作。
#（注：内容由AI生成）
