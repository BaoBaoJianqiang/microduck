# 配对手柄

每个手柄一次。之后，`padd.service` 从启动起驱动任何连接的手柄 —— 无需启动任何东西，也没有东西随你的 ssh 会话死亡。

## 把手柄放进配对模式

在一个 **Xbox** 控制器上这是两次按压，且第二次是出错的那个：

1. 用 Xbox 按钮的一次**短**按打开它。不要按住那个按钮 —— 按住会关掉控制器。
2. 按顶部边缘、USB-C 端口旁边的小 **Sync** 按钮，直到 Xbox 灯**快速闪烁**。慢闪表示它开着但没在配对。

在一个 **DualSense** 上：同时按住 Create 与 PS 直到灯条闪烁。

## 配对它

```bash
sudo robotctl pad pair
```

```
looking for a gamepad in pairing mode — on an Xbox pad, press the small Sync button on the
top edge (not the Xbox button, which switches it off)
paired  Xbox Wireless Controller 78:86:2E:BB:13:28
padd is driving from it now.
```

不需要 MAC 地址：机器人寻找一个处于配对模式的手柄并拿走它找到的那个。手柄被*信任*且被配对，这正是让它在无人登录的重启后自己重连的原因。

如果有两个处于配对模式，它拒绝而非猜测，并打印它们的地址。命名一个也是配对机器人不识别为手柄的硬件的方式：

```bash
sudo robotctl pad pair 78:86:2E:BB:13:28
```

**第二个手柄无需忘记。** 一个已绑定的手柄在范围内且在每次扫描中，因此机器人偏好一个处于配对模式的；之后两者都保持配对，`padd` 驱动 whichever connects。代价是，在没有新东西处于配对模式时重新运行会等完整个搜索窗口才报告你已经有的手柄 —— 如果你只是修复信任，用 `--timeout 5`。

## 检查

```bash
robotctl pad status
```

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

两行，因为它们分别失败：一个连接的手柄带一个死掉的驱动看起来与一个工作的机器人不理你完全一样。

`paired but NOT trusted` 是值得知道的状态。它现在工作且重启后不重连，因为批准一次重连需要一个代理，而启动时没有。重新运行 `pad pair` 来修复它。

## 忘记一个

```bash
sudo robotctl pad forget 78:86:2E:BB:13:28
```

这移除**机器人那半**的绑定，那是一个机器人能移除的全部。手柄保留它自己那半，因此再次配对需要它回到配对模式 —— 否则它带着一个这个机器人不再有的密钥到达，绑定被拒绝。

一个 Xbox 手柄持有**一个**主机绑定，而一个半途而废的尝试让它持有一个这块板子不再有的密钥 —— 这以与一块坏掉的板子完全相同的方式失败。如果配对一直失败，把手柄配对到一台笔记本一次并在那里移除它；那会消耗并释放它的绑定槽，而仅把它放进配对模式不可靠地做到。

## 当手柄完全无法绑定时

在 aic8800 无线电上，当 `btd` 广播时手柄无法形成**新**绑定。用 `--pause-btd-on-pair` 重新初始化这样一块板子：

```bash
./scripts/provision-board.sh --pause-btd-on-pair pierre@192.168.1.42
```

那在 `/var/lib/robot/weird-ble` 留下一个标记且不改其他任何东西。在有那个标记的板子上，`sudo robotctl pad pair` 自己处理其余 —— 它停止 `btd`、对适配器下电再上电、配对、然后再次启动 `btd`。它边做边说。已有绑定不受这些影响，因此一个已配对的手柄在一切运行时连接并驱动。

一些单元另外在 BlueZ 默认的 `Privacy = off` 下完全无法绑定，即使 `btd` 暂停。那些想要 `--weird-ble`，它隐含暂停且也设置 `Privacy = device`。

**先试暂停。** 在一块只需要暂停的板子上 `Privacy = device` 产生一个比完全无标志更糟的失败：手柄绑定然后以 `Encryption Change: PIN or Key Missing (0x06)` 抖动，永不创建输入设备。如果你看到那个，去掉 `--weird-ble` 并保留暂停 —— [`install-dev.md`](install-dev.md#三种配置以及如何分辨你有哪种) 有表与在两者间移动一块板子的命令。

在这样一块板子上手动配对需要同样的两步：

```bash
sudo systemctl stop btd
```

```bash
sudo bluetoothctl power off && sudo bluetoothctl power on
```

配对，然后：

```bash
sudo systemctl start btd
```

下电再上电不是可选的。停止 `btd` 留下它的广播以及其配对代理给控制器的 IO 能力，且手柄仍拒绝绑定 —— 一次重启有同样效果，这正是这被发现的方式。绝不要改用 `systemctl restart bluetooth`：在这块板子上那会直到重启前都没有适配器。

`--weird-ble` 在 [`install-dev.md`](install-dev.md) 中是默认，因为大约一半这些板子需要它且没有可测量的东西说哪块。但一块不需要它的板子不该带它 —— 该标志在每次配对时花费一次 `btd` 停止与一次适配器下电再上电 —— 因此那一页也说了如何检查与如何去掉它。

两者都是对 aic8800 无线电的变通方案，非设计属性。它们随无线电一起消失。

## 当配对每次都失败

检查 `/etc/bluetooth/main.conf` 的 `Privacy` 设置。它应读为 `Privacy = device`。一块携带 `Privacy = off` 的板子可能完全拒绝绑定手柄 —— connect 以 `le-connection-abort-by-local` 放弃，且手柄永远不到 `Paired: yes`。

如果相反一个捕获显示配对到达 `DHKey check failed (0x0b)`，那是相反的故障，且 `Privacy = off` 是那块板子上该试的值。在更改值之前拍一个 `btmon` 捕获，因为两者都见过且它们需要不同的答案。

```bash
sudo sh scripts/setup-board.sh
```

```bash
sudo reboot
```

`setup-board.sh` 纠正该值，且它直到重启才生效。

否则通常原因是手柄在交换开始前已经离开配对模式：再按一次 Sync 并在灯仍快速闪烁时重新运行。要查看交换本身：

```bash
sudo btmon -t > /tmp/btmon.log 2>&1 &
```

配对，然后 `sudo pkill btmon` 并寻找 `SMP: Pairing Failed` 及其旁边的原因。那是唯一一个区分板子设置与一个没在听的手柄的工具。

## 当你驾驶时它断开

查看手柄自己的输入流，它已经在板子上：

```bash
robotctl monitor
```

按 `p`。打开的块是 `padd` 正在驱动的手柄的原始 evdev 流 —— 每个报告，由内核打时间戳：

```
┌ pad Xbox Wireless Controller · /dev/input/event5 · 78:86:2e:bb:13:28 ─────────────┐
│ cadence  124/s while driving · last 8 ms ago · worst 84 ms · over 100 ms 0 · …    │
│ gap ms                                       ▁▁▁▁▂▁▁▁▁▃▁▁▁▁▁▁▂▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁ │
│ X     ····│██··   -8734 Y     ····│····       0 Z     ·········       0 …         │
│ held     BTN_START                                                                │
└ 4213 reports · gap ≤100 ms ───────────────────────────────── reports intact ──────┘
```

移动摇杆，条形跟随它们。轨迹的用途是上面那行：每个报告一个条，因此一次卡顿是一个尖峰，且一个已经恢复的仍在屏幕上。全高是 100 ms —— 驱动开始感觉到的点 —— 且超过 500 ms `robotd` 已经把速度清零。

**机器人上没有任何其他东西能显示这个。** `padd` 以 50 Hz 重发最后的摇杆值，因此一个已经停止交付的无线电在下游各处看起来仍像一个活的驱动：`robot.state` 携带新鲜的意图，死人开关从不触发，且机器人继续按一个没人给的命令走路。上面块中的 `asked` 列在这个扁平线时看起来完美。

一个静止的手柄什么都不发，因此静默只在你驾驶时是证据。该块在五秒后说 `the sticks are still` 而非指控链路，并分别计数那些时段。

然后，为一个窗口的定论而非实时画面，把测量从这个仓库的一个克隆复制到板子上：

```bash
scp scripts/pad-link-test.sh radxa@<board>:/tmp/
```

已经发生的，来自 `padd` 的日志 —— 无需手柄，且立即应答：

```bash
sudo sh /tmp/pad-link-test.sh --history
```

要现在测量链路，手柄开着且 `padd` 运行。**整两分钟保持摇杆移动**：一个静止的手柄什么都不发，且静默读起来完全像一个卡顿的链路。

```bash
sudo sh /tmp/pad-link-test.sh
```

它计数掉线，以及连接时手柄输入报告之间的间隙。超过 500 ms 的间隙是机器人停止 —— `robotd` 在那里把速度清零。每次掉线后跟内核的原因：`0x08` 是监督超时，意味着距离或干扰，`0x13` 意味着有人关了手柄。

放下手柄不是卡顿，且不算作一个 —— 但它是测量什么都没学到的时间，因此报告说你实际驾驶了窗口的多少，并拒绝对一个它几乎没看到的链路下判断。

在机器人看着时走开是你找到距离的方式。

## 这块板子与那块板子运行的是同一个栈吗

一个在一个机器人上卡顿而在它的孪生兄弟上不卡顿的手柄通常不是手柄的问题。相隔数周构建的两块板子运行不同的内核、不同的 BlueZ、不同的控制器固件，且手柄在不同的手柄固件上 —— 且那些在 `pad status` 中都不可见。

把报告复制到每块板子上，从这个仓库的一个克隆：

```bash
scp scripts/pad-stack-report.sh radxa@<board>:/tmp/
```

```bash
sudo sh /tmp/pad-stack-report.sh
```

它打印整个栈并把同样的文本保存到 `/tmp/pad-stack-<host>-<when>.log`：内核、BlueZ、适配器的 HCI 版本、它是哪个无线电以及内核在启动时为它加载的固件、什么承载 HID、绑定持有哪些密钥、当前使用的传输，以及手柄自己的固件修订。它无需 root 运行，并对需要它的三样东西说 `unreadable`。

要比较两块板子，只问每块必须匹配的值：

```bash
ssh radxa@<board-a> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/a.fp
```

```bash
ssh radxa@<board-b> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/b.fp
```

```bash
diff /tmp/a.fp /tmp/b.fp
```

无输出意味着同一个栈。指纹不带时间戳与地址，因此 `diff` 打印的任何东西都是一个真实差异。

在其余之前读两行。`transport` 在目前试过的每个手柄上都是 `LE`，且一块说 `BR/EDR` 的板子是把手柄放进内核的经典 HID 路径而非 BlueZ 的 —— 一个不同的驱动、不同的按钮编号。`input` 是 `Bus`/`Vendor`/`Product`/`Version` 四元组，SDL 与 `gilrs` 把它哈希成一个映射 GUID：在那里不同的两块板子有不同的轴与按钮映射，无论其他什么匹配。

---

驾驶 —— 控制与速度限制 —— 在[速查表](cheatsheet.md#手柄configd)中；从笔记本经一个转发的 socket 运行 `padd` 在 [dev 速查表](cheatsheet-dev.md#从笔记本--用手中的手柄驾驶)中。
