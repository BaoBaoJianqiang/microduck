# 速查表

`robotctl`，在机器人上运行。这里的每个命令都从交付它的分支上的 `--help` 取得，非凭记忆。

只读命令无需特权。任何**更改**机器人的东西需要 `sudo`（或对 `configd` 一个在 `--allow-user`/`--allow-group` 中的用户，对 `updaterd` 在 `updater.toml` 中的 `allow_uids`/`allow_gids`）。

分支构建、发布候选与更新后的重启陷阱在 [`cheatsheet-dev.md`](cheatsheet-dev.md) —— 它们需要一块 dev 板。同一个机器人经蓝牙从一台笔记本、无网络且无 ssh，是 [`duckctl.md`](duckctl.md)。

## 在机器人上 —— `robotctl`

### 第一件该运行的事

```
robotctl version
```

每个守护进程*在运行*什么对照*安装了*什么，加上它们不一致时的警告。在相信任何其他诊断之前运行这个 —— 一个在更新后服务旧代码的守护进程看起来与你刚交付的修复中的一个 bug 完全一样。见下面的"更新之后"。

```
robotctl health
```

一个报告中的硬件与软件。当机器人不健康或不可达时以非零退出，因此它可以门控一个脚本 —— 一个热电机或一个被锁定的组件被报告，不被评判，且不影响退出码。`--json` 用于一个支持包。

### 观察循环

```
robotctl monitor
```

一个客户端要求的东西旁边是实际应用的，当它们不同时命名原因 —— 安全不断钳制东西，而"摇杆向前且机器人不动"没有那个是不可读的。一个限制被拼写出来而非命名：`deadman — no intent arrived recently, velocity zeroed`。

也在画面上：每个关节对照它被命令的测量、IMU 的投影重力与从中得出的跌倒判定，以及作为轨迹的达到循环率，因此一个已经恢复的卡顿仍可见。投影重力是这个流上唯一的 IMU 量 —— 直立约为 `[0, 0, -1]`，且 `fallen` 正是从中决定的。过期读取计数器与它们相对的比率住在 `robotctl health`。

头部的最后一行是机器人的状况而非其行为：电池组的电压与作为分数的电量、最热的舵机与板子自己的温度。它来自 `robot.health`，每两秒轮询，因为它没有任何东西在状态流上 —— 且它是任何出错的东西被命名的地方，无论那是 `unhealthy: control loop at 43.9 Hz`、`degraded: no robot on the motor bus after 3 attempts` 还是 `orientation frozen — 25 stale reads`。最后那个在这一行且画面上别处没有：一块停止融合的板子继续应答总线，因此没有东西出错且上面的重力向量无限期保持一个看似合理的姿态。

0% 是 `BATTERY_EMPTY_V`，那是 `robotd` 让机器人坐下并切断电源的地方，因此该数字是一个倒计时而非仪表 —— 30% 黄，15% 红。一个还没被取的读数说 `batt not read yet` 而非 `0.00 V`，那是运行时间的第一秒与一个无法应答的总线两者看起来的样子。该行即使在完全没有状态时也被绘制，且那是它最重要的情况：一块舵机电源关的板子永不完成一个控制滴答，因此没有东西到达流上且原因只在健康应答上。

底部边框命名加载的策略 —— `.onnx` 文件，以及是否配置了一个站立网络 —— 因为 `walk` 是两个带不同步态的版本都报告的一个模式。一个没有策略的机器人说明，且一个策略无法加载的机器人说明那个，流的 `held` 无法区分。

右侧，**机器人按它站立的样子绘制** —— 与策略训练所对照的同一个视觉模型，由测量的关节角度摆姿势并由 IMU 的重力向量倾斜。一条腿折错方向、一个头俯冲到地板与一只侧躺的鸭子在关节表中都只是数字；它们每个在这里都很明显。它默认开启且在终端足够宽（约 110 列 —— 表格优先，机器人拿剩余的）时出现。**`d`** 关掉它；**`[`** 与 **`]`**，或 `←`/`→`，环绕它。

每当 ToF 在交付帧，它看到的东西被绘制进同一场景 —— 黄色是命中，绿色是地板 —— 对照机器人自己的身体做深度测试，因此喙后面的一个点被它隐藏。这正是让"它看到我的手，还是看到它自己"可回答的原因。它不需要键：点在帧到达时出现，在它们停止时消失。

在它下面，当列足够高时，**机器人去过哪里的地图**：来自脚接触与 IMU 的里程计轨迹，用盲文。面板永不增长 —— 世界随轨迹缩小，因此整条路径保持在画面内。`+` 是它开始的地方，`●` 是带一个短射线表示朝向的机器人，屏幕向上是它启动时的朝向。没有磁力计，因此这是相对运动且会漂移；它回答"它走了一个圈吗"而非"它在哪里"。

`q` 退出；`↑`/`↓` 在一个太短装不下全部的窗口上滚动关节列表；`u` 在角度与弧度间切换；`t` 打开 [ToF 矩阵](#tof-传感器tofd)；`d` 切换机器人视图且 `[` / `]` 环绕它；`p` 打开手柄的原始输入流 —— 来自游戏手柄的每个 evdev 报告，带它们之间的间隙，那是唯一一个卡顿无线电可见的地方（[配对手柄](pair-a-gamepad.md#当你驾驶时它断开)）。屏幕上角度是度 —— 关节、头与偏航率。
被重定向或管道传输时它改为每个滴答打印一行，因此 `> run.log` 与 `| grep FALLEN` 行为正常，且那些数字无论屏幕设为什么都保持弧度。关节向量在 `--json` 中，它携带整个状态，每行一个对象：

```
robotctl monitor --json --hz 50 > run.jsonl
```

### 配置机器人

```
sudo robotctl configure
```

一个在 `/etc/robot/robotd.toml` 上的交互式编辑器：守护进程认识的每个键，特性开关在前（策略开/关、走/轮、跌倒松弛、音频、宠物检测、电池关机、摄像头与视频质量…），当前值对照默认，一行文档。SPACE 切换，ENTER 键入一个值，`u` 把一个键还原为其默认。黄色（标记 `•`）的值是这台机器人偏离默认的键；其他一切是内置默认，且 `unset` 的可选值显示它们解析为 `(auto)` 的东西。

三个值得信任的属性：

- **它无法与守护进程不一致。** 模式、默认与验证来自 `robotd` 解析文件所用的同一个 crate，且键列表被一个测试完整固定 —— 守护进程中一个新的 `[section]` 会在这里出现，否则构建失败。
- **它无法吃掉你的文件。** 注释、排序与来自其他版本的键原封不动存活；只有你更改的键被写。还原一个键移除它（以及附着在它上的注释）而非固定默认，因此文件保持一个*决定*的列表，而非默认的副本。
- **它无法写一个 robotd 拒绝启动的文件。** 每次保存都先通过守护进程自己的加载器验证，原子地（临时文件 + 重命名），并以原因拒绝。

守护进程在启动时读一次文件，因此保存提供一次重启 —— 那些读了你更改的东西的：`[media]` 是 `mediad`，其他一切是 `robotd`。需要 `sudo`，因为文件是 root 拥有的 —— 没有它编辑器以只读打开并在第一次写入时说明。`--file` 把它指向别处用于一个工作台副本。随附的 `deploy/robotd.toml` 保持为*为什么*每个旋钮存在的参考；这是用来翻转它们的。

#### 视频质量

```
sudo robotctl configure
```

设 `media.quality` —— `1080p30`、`720p30`、`720p15` 或 `360p30` —— 并接受它提供的重启。`media.camera` 关改为流式传输一个测试图案，那是一块没摄像头的板子想要的：WebRTC 的*控制*通道搭在视频轨道上，因此一个无法启动的流水线两者都损失。`media.bitrate` 跟随质量，除非你设置它；单位是比特每秒。

`media.congestion_control` 是那一节中的另一个旋钮，且它是移动 CPU 的那个：`disabled` 丢掉带宽估计器，那是 `mediad` 中最大的单一消费者（对照捕获的 0.3%，占一个核的 7.6%），且让 `media.bitrate` 成为速率而非一个起点。它代价是自适应性 —— 在一个降级的链路上，画面停滞而非速率下降。

720p30 是流水线被测量的梯级；一个撑不住的梯级运行更慢而非失败。`robotctl monitor` 在底部边框报告达到的速率，当它低于要求的 90% 时以黄色带 `of <target>` 在旁边。应用了什么：

```
journalctl -u mediad -b | grep streaming
```

#### 你自己的策略

你不需要切一个版本来试一个网络。在板子上把 `robotd` 指向你自己的 `.onnx`，在 `/etc/robot/robotd.toml` 中：

```toml
[policy]
walk = "/home/radxa/my_walking.onnx"
stand = "/home/radxa/my_stand.onnx"
```

```
sudo systemctl restart robotd
```

你的路径在更新中存活 —— 一个版本替换二进制与它交付的策略，不替换指向别处的文件。删除那些行回到版本携带的那些。

一个无法加载的策略报告**不健康**，且 `robotctl health` 与 `monitor` 的底部边框都命名原因。一个策略必须具有的形态，以及加载时还检查什么，在 [`../design/robotd-design.md`](../design/robotd-design.md) §2.3。

### 给关节通电（`robotd`）

```
sudo robotctl robot init
```

```
sudo robotctl robot relax --yes
```

`init` 给关节通电并在约两秒内斜升到初始姿态 —— **它移动每个关节**，因此让机器人在架子上。它不需要策略，且它是手柄的 Start 在去驾驶的路上做的事，因此手动它是一个工作台的事。

`relax` 切断电源且**机器人塌陷**如果没有东西撑它，这就是它想要 `--yes` 的原因。这是除拔插头外回到松弛的唯一方式：再按一次 Stop 停止策略并让机器人保持站立，而 `robot.stop` 在仍站立时把速度清零。

两者都经过 `robotd`，它拥有电机总线。`robotd init` —— 子命令 —— 仍为一个守护进程没运行的机器人存在，且它需要守护进程停止，因为一个 UART 上的两个写者会破坏彼此的应答：

```
sudo systemctl stop robotd && sudo /opt/robot/daemon/current/bin/robotd init && sudo systemctl start robotd
```

`init` 无论机器人是否跌倒都工作 —— 默认一次跌倒是一个*报告*（在 `robotctl monitor` 中可见），非一个门，与原型一致。一块在 `robotd.toml` 中设了 `[safety] fall_limp` 或 `fall_recover` 的板子武装门：那里一个跌倒的机器人变松弛并拒绝 `init`/`enable`/技能，直到它被扶起来。

### 手柄（`configd`）

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

配对是每个手柄一次且有自己的一页 —— [`pair-a-gamepad.md`](pair-a-gamepad.md)：哪个按钮把手柄放进配对模式、添加第二个手柄而不忘记第一个，以及当它无法绑定时该做什么（`/etc/bluetooth/main.conf` 中的 `Privacy` 设置比其他任何东西更常是答案）。

`padd.service` 从启动起运行并驱动任何连接的手柄，因此配对是唯一的步骤。映射是原型的，因此肌肉记忆可以延续：

| 控制 | 做什么 |
| --- | --- |
| 左摇杆 | 驾驶：前进/后退与横移 · 头：头偏航与俯仰 · 身体姿态：向上与蹲下 |
| 右摇杆 | 驾驶：转向 · 头：颈俯仰与头翻滚 · 身体姿态：俯仰与翻滚 |
| **Start** | 切换策略 —— 在它开之前什么都不动 |
| **Y** / triangle | 头部模式：摇杆摆头（身体保持不动） |
| **B** / circle | 身体姿态模式：摇杆倾斜与蹲下站立的机器人 |
| **A** / cross | 地面啄 |
| **X** / square | 滚奏 —— 一次前滚；按住链式翻滚 |
| **LB / RB** | 左 / 右踢 |
| **DPad-Down** | 坐 ↔ 站 |
| **RT / LT** | 嘴（任一扳机）—— RT 也嘎嘎叫；LT 按住时乘坐"wheee" |
| **DPad-Up**，按住 3 秒 | 切换驾驶模式，走 ⇄ 轮 |
| **Select**，按住 2 秒 | 坐下，然后关机 |

没有停止按钮：松开摇杆机器人站立，且如果 `padd` 死亡 `robotd` 的死人开关停止它。在一个轮式机器人上（`robotd.toml` 中 `mode = "roller"`）摇杆自动采取轮式塑形 —— 不对称推/刹，无横移 —— 且 A 触发蹲下。其他技能随行：坐、踢与滚奏在轮子上也工作，如原型所有。

**按住 DPad-Up 在两者间切换**，当你刚给鸭子装上轮子或卸下它们时：机器人嘎嘎叫一次表示走或两次表示轮，回到其初始姿态，在那里加载那个模式的策略并再次驾驶 —— 几秒，全程有扭矩，无重启。`robotd.toml` 不被触碰，因此重启回到配置的模式；用 `robotctl configure`（或 `[policy] mode`）让它持久。它是一个按住而非按压，因为 D-pad 上在驾驶时容易靠到。

`pad status` 分开回答两个问题，因为一个连接的手柄与一个死掉的驱动从外面看完全一样：

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

要用非默认限制驾驶，先停止服务否则两个进程争抢摇杆：

```
sudo systemctl stop padd
```

```
sudo -u padd /opt/robot/daemon/current/bin/padd --max-linear 0.25
```

当链路本身是嫌疑时，实时观察它 —— `robotctl monitor`，然后 `p`。那也在没有机器人时工作：在一块舵机未通电或 `robotd` 停止的板子上，监控器在手柄块打开而非拒绝。要一个窗口的定论而非实时画面，把测量从这个仓库的一个克隆复制过来：

```
scp scripts/pad-link-test.sh radxa@<board>:/tmp/
```

已经在 `padd` 日志中的掉线 —— 无需手柄，且立即应答：

```
sudo sh /tmp/pad-link-test.sh --history
```

或现在测量它，整两分钟保持摇杆移动：

```
sudo sh /tmp/pad-link-test.sh
```

它对照内核自己的每个原因计数掉线，并计时手柄输入报告之间的间隙 —— `padd` 看不到的失败，那里链路保持向上且机器人按一个陈旧命令走路。[`pair-a-gamepad.md`](pair-a-gamepad.md#当你驾驶时它断开) 读这些数字。

当两块板子用同一个手柄表现不同，差异在它下面的栈：

```
scp scripts/pad-stack-report.sh radxa@<board>:/tmp/
```

```
sudo sh /tmp/pad-stack-report.sh
```

内核、BlueZ、控制器固件、LE 或 BR/EDR，以及手柄自己的固件修订 —— 打印并保存到 `/tmp/pad-stack-<host>-<when>.log`。`--fingerprint` 只打印两块板子之间必须匹配的值，用于 `diff`。
[`pair-a-gamepad.md`](pair-a-gamepad.md#这块板子与那块板子运行的是同一个栈吗) 有比较。

### 语音

```
robotctl quack
```

区分鸭子最响亮的方式：每个机器人的音库从它的 SoC 序列号生成（`sounds ensure-bank`，由每个版本安装运行），因此应答的机器人 —— 用一个只属于它的声音 —— 正是你 SSH 进去的那个。一个没有声音的机器人 —— 音频关，或无音库 —— 说明而非打印 🦆，因此静默总是意味着错的鸭子。机器人在 `robotd` 起来时也问候，在关机前啄别，且 —— 如果你要求它 —— 当麦克风听到它的头被挠时咕咕叫。那个在两种模式下默认关闭（`audio.pet_detect = true` 打开它；分类器随版本交付）：常开版本对每个偶然触碰都咕咕叫且变得烦人。启动问候有它自己的开关，给任何整天重启守护进程的人：

```
sudo robotctl configure
```

设 `audio.greet = false` 并接受保存时提供的重启。那静默那一声嘎嘎叫且不碰触发器与麦克风，而 `audio.enabled = false` 会。音频硬件启动 —— 编解码器驱动、overlays、混音器 —— 是 `setup-board.sh` 的音频节，每块板子一次。

要试听一个声音或手动重新生成音库，版本携带生成器：

```
/opt/robot/daemon/current/bin/sounds show
sudo /opt/robot/daemon/current/bin/sounds ensure-bank --force
```

`sounds theremin` 试听*实时*合成器 —— 泰勒明琴演奏的声音，由一个脚本化的手扫以 ToF 自己的帧率驱动。`--out sweep.wav` 写入而非播放，那是你在面前没有机器人时听到一个声音变化的方式。

### 鸭子合唱

```
robotctl chorale
```

一个房间里的两只鸭子一起唱一首四部曲；更多加入它们发现已经在进行的。运行直到 Ctrl-C。`--off` 停止一个。

**默认关闭** —— `robotd.toml` 中的 `[chorale] accept`，且它必须在每个应该参与的鸭子上设置。一个 chorale 移动嘴与头，因此一个因为另一只鸭子走进来而开始动画的鸭子会做没人要求的动作。关闭也意味着*不可见*：一个没选择加入的鸭子不在空中放任何东西，而非礼貌地拒绝。

它如何工作，按问题出现的顺序：

- **没人负责。** 两只鸭子看到相同的信标且较低 id 的指挥，因此没有选举可输且没有必须到达的消息。
- **没有共享时钟。** 板子没有 NTP 且没有 RTC 协议，因此指挥的节拍计数器*就是*时基：它每次节拍在一个 BLE 广播中加一个字节，且一个新值的到达就是重拍。跟随者在约 25 拍上平均相位，这把无线电的抖动带进合奏需要的 ±20 ms 内。
- **声部是算出的，非分配的。** 最低的鸭子唱贝斯。指挥广播花名册且每个人在它上面重放同样的座位安排 —— 这正是阻止两只鸭子在它们各自能看到房间不同子集时唱同一行的东西。
- **加入不改变任何人的声部。** 一只到达的鸭子拿空闲的声部。一只离开的鸭子保留它的*座位* —— 它的行只是没人唱，正如在一个合唱团里有人走出去 —— 因为重新安排幸存者是一曲中间唯一值得避免的事。

读数一声部确定就命名它，因此一只鸭子最终唱什么存活在回滚中：

```
listening for other ducks — Ctrl-C to stop
  singing tenor    with 3 voices
  tenor    bar   12  beat  45.2  3 voices
```

指挥每次表演选曲，且一次表演*结束* —— 在最后一个音符后每个人回去听、重新安定，并在一次呼吸后唱别的东西。`robotctl chorale --piece 2` 固定这个机器人**如果它指挥**选的（一个跟随者唱信标命名的，因此在每只鸭子上设置它以保证那首歌）；未知 id 以机器人的目录被拒绝。Id：1 wistful、2 duck-strut、3 outer-wilds（测试资产，非发布用）。`robotd` 环境中的 `DUCK_CHORALE_PIECE=<id>` 是常备回退 —— 注意它必须在 **robotd** 上，不在 `robotctl` 命令行上。

要在没有任何鸭子的情况下听到编曲，一台机器可以渲染整个合奏：

```
sounds chorale --voices 4                 # 或 --seeds 100,7,42 用于特定鸭子
sounds chorale --score my-piece.mid       # 任何记谱编辑器导出的
sounds chorale --rolloff 0                # 给全频扬声器，非鸭子的
```

乐谱来自 `sounds/scores/*.duckscore` —— 一种面向行的文本格式，在 `wistful.duckscore` 中有文档，那也是随附的曲子 —— 或一个 MIDI 文件，那是值得用的路径：**MuseScore 是乐谱编辑器。** 每个声部一个乐器而非一个钢琴谱表，命名声部，导出 MIDI。声部按平均音高匹配，因此一个顶谱优先写的乐谱仍把贝斯放在贝斯上；一个*命名*为 "Soprano" 的轨道比其音高更可信。

### 演奏鸭子（ToF 泰勒明琴）

```
robotctl theremin
```

头部的深度传感器变成一件乐器：喙前面的一只手是音高 —— 更近更高 —— 且嘴随音符张开，在音域顶部宽。运行直到 Ctrl-C 且在出去时放下乐器。`--off` 放下一个客户留下举着的。

一个里面没有什么聪明的显式模式：当它举着时，可演奏波段内最近的回波就是手。把鸭子指向开阔空间它是静默的；把它指向 40 cm 外的墙它演奏一个稳定的音符。它坐着、站着或走路都演奏 —— 嘴不是任何策略的一部分。

读数的最后一列是**传感器对那一帧说了什么**，且它是每个"为什么它停止演奏"的答案：

```
  0.34 m    438.1 Hz   60% ██████    14 usable · 255:38 4*:9 5*:5 1:12
```

有多少区域携带一个机器人相信的状态，然后每个 ST 状态码的计数，在被相信的那些上带一个 `*`。音符前的一个 `~` 意味着它是一个*保持*的音符，桥接一个传感器掉线而非现在测量的东西。

那一列存在因为它本会在一分钟内发现的 bug：ST 把 5 与 9 记录为"range valid"，且一个只相信那些的构建**在约 30 cm 处停止看到一只手** —— 超过那一只移动的手回来为 4 或 13（*consistency failed*，sigma 太高），携带一个对音高完全好的距离。如果射程短，给 `robotd.toml` 中的 `[theremin] statuses` 添加代码；如果它对空气演奏幽灵音符，移除一些。`hold_ms` 是防抖：它乘坐一个闪烁的区域。

注意 `robotctl monitor` 的 ToF 网格比泰勒明琴严格 —— 它把 5/9 之外的任何东西标记为 `x`，*无法测量*。一个满是 `x` 的网格不意味着传感器坏了；它意味着它对它有的数字悲观。

### ToF 传感器（`tofd`）

来自头部传感器的一个 8×8 深度矩阵。`robotctl monitor`，然后 **`t`**：

```
┌ tof VL53L8CX · 15 Hz · 8×8 · 48/64 ranged · 0.12–3.54 m ─────────────┐
│ 0.12 0.15    x 1.44 1.86    · 2.70 3.12                              │
└ · nothing in range · x could not measure · near→far ── seq 412 · 6 ms ┘
```

距离以米为单位，从近暖色到远冷色着色。两个标记重要：`·` 是*已测量，范围内无物* —— 自由空间，那是信息 —— 而 `x` 是*无法测量*，对外面有什么完全不说明。一个把两者都显示为空白的网格会隐藏差异。

这是传感器自己的帧，非机器人的：在运动学存在之前没有重投影，这也是让这个块成为检查安装角度的正确地方的原因。

`tofd` 拥有传感器且没有其他东西读总线。它是一个普通服务 —— `sudo systemctl stop tofd` 安全，没有东西依赖它，且 `monitor` 说"no depth stream"并继续。它区分三样东西，因为它们需要不同的修复：

| 块说 | 它意味着什么 |
| --- | --- |
| `connecting to tofd…` / `no depth stream` | 守护进程没运行 |
| `no sensor: …` | `tofd` 起来了；总线上没有东西应答（大多数鸭子） |
| `waiting for the first frame…` | 一个传感器在测距；它的第一次扫描约 66 ms 远 |

要手动看总线上有什么，或在无终端 UI 下观察帧：

```
sudo i2cdetect -y -r 3
journalctl -u tofd -b
```

传感器共享编解码器的 I²C 总线，因此 `setup-board.sh` 的音频节已经初始化了总线本身；ToF 步骤只添加稳定的 `/dev/i2c-pihat` 名。两代传感器都支持 —— 一个 VL53L5CX 与一个 VL53L8CX 在板子上可互换，且守护进程从一次 ID 读取选驱动。

### Wifi（`configd`）

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

`--psk-stdin` 让密码短语不进 `ps`，它会在命令生命周期内向盒子上每个用户显示一个 `--psk` 参数。在任何共享的东西上偏好它。

加入一个网络**把机器人从它所在的网络断开**，因此经 wifi 的 ssh 会话会断开。那是操作在工作。一次扫描花几秒 —— 它等待无线电扫频而非返回上一次扫描的结果。

### 身份与电源（`configd`）

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

开箱即用一个机器人叫自己 `duck-` 加上从它自己的序列号派生的四个字符，因此从同一镜像烧的两块板子在手机的蓝牙列表中仍看起来不同。重命名在几秒内经蓝牙生效 —— 无需重启 —— 但手机得重新扫描才能看到。

PIN 是手机经蓝牙认证用的。出厂默认是 `000000`，那认证任何读过这个仓库的人。

### 更新（`updaterd`）

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

`log` 列出尝试，每行一个，最新在前；第一列是运行号。`show` 取那些数字之一 —— 或什么都不给，要最近的 —— 并打印那次运行做的一切，然后同一窗口的日志：

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

时间是 UTC，下面的日志也是。`+` 列是距上面行的间隙，那是你找到那两分钟的方式。

读日志需要 `robot` 组不携带的特权，因此除非你是 root 否则后半部分空着回来。发生时它打印 `journalctl` 行；在它前面加 `sudo` 是修复。`--no-journal` 不尝试就打印那行，且 `--json` 单独给出记录。

组件是 `daemon` —— 一个覆盖每个二进制的组件。`apply daemon` 安装 stable 通道提供的；分支构建与发布候选需要 [`cheatsheet-dev.md`](cheatsheet-dev.md)。

### 无下载切换

到板子已经解包的某个东西。无网络参与：

```
sudo robotctl update select daemon 0.1.4
```

```
sudo robotctl update rollback daemon
```

```
sudo robotctl update reset-to-golden daemon
```

`select` 激活一个已安装的版本，`rollback` 去前一个安装的，`reset-to-golden` 去那个永不修剪的已知良好版本。

以及拒绝完全不动：

```
sudo robotctl update pin daemon 0.1.4
```

```
sudo robotctl update pin daemon
```

第二种形式解锁。

### 当 `updaterd` 自己无法启动

上面的一切都经过 `updaterd`，因此当 `updaterd` 是那个挂掉的守护进程时它们都不工作。检查是哪个：

```
systemctl status updaterd robotd btd configd
```

然后不用它回到 golden：

```
sudo robot-rescue --dry-run
```

```
sudo robot-rescue --reboot
```

`--dry-run` 说明它会做什么且不改任何东西。没有 `--reboot` 它交换版本并打印重启命令而非运行它：每个守护进程通过 `current` 执行，因此没有东西在它重启前拾起交换，且一个站着的机器人应该先被接住。

当没有 golden 被配置或当 `current` 已经是 golden 时，它拒绝并说明原因 —— 如果守护进程在 golden 自身上失败，回滚不是答案，日志才是：

```
journalctl -b -u robotd -u updaterd -u btd -u configd
```

### 机器人可能已经做了这个

每次启动三分钟时，一个计时器问版本是否把它的守护进程带起来了，如果没有就回退到 golden。因此一个自己重启且运行比你安装的版本更旧的机器人可能已经自救了。它做了什么：

```
robotctl update log
```

条目读为一次回滚，失败的守护进程在其原因中命名。要看正在做的决定而非其结果：

```
journalctl -b -u robot-boot-check
```

```
sudo robot-boot-check --dry-run
```

它行动一次。当第一次仍在记录时第二次救援被拒绝 —— `updaterd` 在它下次启动时清除那个，因此被拒绝意味着守护进程在 golden 上也没起来，答案是日志而非另一次重启。过了那，如果你读了日志并决定：

```
sudo robot-rescue --force --reboot
```

### 三件容易弄错的事

**`rollback` 需要一个前任，但一次更新创建一个。** 一块刚初始化的板子恰好有一个版本，因此那时的 `rollback` 没有更旧的可去并说明。自动回滚*不*受影响：应用一个版本把它解包在当前版本旁边，然后才移动 `current`，因此到健康门运行时有两个，且你来自的版本是目标。`rollback_target` 选 `current` 之下日志尚未记录为坏的最高已安装版本 —— 因此一块有一个版本的板子从它接受第一次更新的那一刻起就完全受保护。

唯一一个真正不受保护的安装是引导本身，按定义在它之前没有东西。`golden` 会覆盖那个，且它被故意不设直到 1.0.0 存在 —— 因此 `reset-to-golden` 诚实报告没有配置而非做令人惊讶的事。

**`version` 按组件显示活动版本，非版本存储。** 无论解包了多少个，它永不会列出两个版本。直接问存储：

```
ls -l /opt/robot/daemon/releases/ /opt/robot/daemon/current
```

**`apply --version` 需要版本仍存在于上游；`select` 不需要。** 携带已知坏构建的版本从 GitHub 被删除，因此 `apply --version 0.1.3` 故意失败，而 `select 0.1.3` 在一块已经解包它的板子上仍工作。不对称是故意的：没有新板子能获得一个坏版本，且有一个的板子保留其逃生舱。

### 无网络安装

侧载、工厂安装，或救援一块其 `updaterd` 太旧无法接受修复太旧的那个版本的板子。见 [`install-dev.md`](install-dev.md) —— 它是 `updaterd install --from`，且 `--force` 变体有在使用前值得一读的条件。

### 日志

```
journalctl -u configd -b --no-pager | tail -40
```

```
journalctl -u btd -f
```

换入 `robotd` 或 `updaterd`。`-f` 跟随；`-b` 只是这次启动。

启动行携带版本、git 修订与进程被启动自的版本目录，在 `warn` 级别，因此它在任何日志级别存活。

更新历史故意与日志分开 —— 每条目在 `/var/lib/robot/updater/` 下 `fsync` —— 因此它在一个日志易失的机器人上存活：

```
robotctl update log
```

最近二十次运行也在那里保留一个完整记录，在 `runs/` 下，按发生时写入：

```
robotctl update show 42
```

两者都比交换、回滚与断电活得久，这是这块板子上的日志做不到的 —— `/var/log` 是 zram。如果 `robotctl` 本身是坏的，文件是换行分隔的 JSON 且用 `cat` 读得很好：

```
sudo cat /var/lib/robot/updater/runs/000042.jsonl
```

### Tab 补全

`install.sh` 在 `/etc/bash_completion.d/` 设置这个，作为一个向二进制询问它自己补全的加载器 —— 因此它们跟随已安装的版本而非在一次更新添加命令时过时。对一个它没覆盖的 shell，或对你直接从 `target/` 运行的一个构建：

```
eval "$(robotctl completions bash)"
```

`zsh`、`fish`、`elvish` 与 `powershell` 替代 `bash` 工作。
