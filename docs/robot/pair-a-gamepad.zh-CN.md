# 配对游戏手柄

每个手柄一次。在此之后，`padd.service` 从启动开始驱动任何连接的手柄——没有什么要启动的，也没有什么会随你的 ssh 会话死亡。

## 将手柄置于配对模式

在 **Xbox** 控制器上这是两次按下，而第二次是出错的那个：

1. 用**短**按 Xbox 按钮开机。不要按住那个按钮——按住它会关闭控制器。
2. 按下顶部边缘 USB-C 端口旁边的小 **Sync** 按钮，直到 Xbox 灯**快速闪烁**。慢闪烁意味着它已开机但没有在配对。

在 **DualSense** 上：同时按住 Create 和 PS，直到灯条闪烁。

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

不需要 MAC 地址：机器人寻找处于配对模式的游戏手柄并拿走它找到的那个。手柄被*信任*以及配对，这就是使它在重启后无人登录时自行重新连接的原因。

如果有两个处于配对模式，它拒绝而不是猜测，并打印它们的地址。命名一个也是配对机器人不识别为游戏手柄的硬件的方式：

```bash
sudo robotctl pad pair 78:86:2E:BB:13:28
```

**第二个手柄不需要忘记。** 已经绑定的手柄在范围内并在每次扫描中，因此机器人更喜欢处于配对模式的那个；之后两者保持配对，`padd` 驱动任何连接的那个。代价是在没有新的处于配对模式的东西的情况下重新运行会等待整个搜索窗口，然后报告你已经有的手柄——如果你只是在修复信任，用 `--timeout 5`。

## 检查它

```bash
robotctl pad status
```

```
pad     Xbox Wireless Controller 78:86:2E:BB:13:28  connected
padd    active — driving whatever pad connects
```

两行，因为它们分别失败：一个连接的手柄加上一个死掉的驱动程序看起来和一个工作的机器人无视你完全一样。

`paired but NOT trusted` 是值得知道的状态。它现在工作，但在重启后不会重新连接，因为批准重新连接需要一个代理，而在启动时没有。重新运行 `pad pair` 来修复它。

## 忘记一个

```bash
sudo robotctl pad forget 78:86:2E:BB:13:28
```

这移除了绑定的**机器人的一半**，这是机器人能移除的全部。手柄保留它自己的一半，因此再次配对它需要它回到配对模式——否则它带着这个机器人不再有的密钥到达，绑定被拒绝。

Xbox 手柄持有**一个**主机绑定，而一次半完成的尝试让它持有这个板不再有的密钥——这以完全像坏板一样的方式失败。如果配对一直失败，将手柄配对到笔记本电脑一次并在那里移除它；这会消耗并释放它的绑定槽，而仅将其置于配对模式并不可靠地做到这一点。

## 当手柄根本无法绑定时

在 aic8800 无线电上，当 `btd` 正在广告时，手柄无法形成**新的**绑定。用 `--pause-btd-on-pair` 重新 provision 这样的板：

```bash
./scripts/provision-board.sh --pause-btd-on-pair pierre@192.168.1.42
```

这在 `/var/lib/robot/weird-ble` 留下一个标记，其他什么都不改变。在有那个标记的板上，`sudo robotctl pad pair` 自己处理其余部分——它停止 `btd`，对适配器进行电源循环，配对，然后再次启动 `btd`。它一边做一边说。现有的绑定不受任何这些影响，因此配对的手柄在一切运行时连接并驱动。

有些单元另外在 BlueZ 的默认 `Privacy = off` 下根本无法绑定，即使 `btd` 已暂停。那些想要 `--weird-ble`，它暗示暂停并且还设置 `Privacy = device`。

**先尝试暂停。** 在只需要暂停的板上设置 `Privacy = device` 会产生比根本没有标志更糟的失败：手柄绑定然后以 `Encryption Change: PIN or Key Missing (0x06)` 抖动，永远不会创建输入设备。如果你看到那个，去掉 `--weird-ble` 并保留暂停——[`install-dev.md`](install-dev.md#the-three-configurations-and-how-to-tell-which-you-have) 有表格和在两者之间移动板的命令。

在这样的板上手动配对需要相同的两个步骤：

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

电源循环不是可选的。停止 `btd` 留下它的广告和其配对代理给控制器的 IO 能力，手柄仍然拒绝绑定——重启有相同的效果，这就是这是如何被发现的。永远不要用 `systemctl restart bluetooth` 代替：在这个板上那会在重启前留下根本没有适配器。

`--weird-ble` 是 [`install-dev.md`](install-dev.md) 中的默认值，因为大约一半的这些板需要它，并且没有什么可测量的说明哪个。但不需要它的板不应该携带它——该标志在每次配对时花费一次 `btd` 停止和一次适配器电源循环——因此该页面也说明了如何检查以及如何去掉它。

两者都是 aic8800 无线电的变通方法，不是设计的属性。它们随无线电一起消失。

## 当配对每次都失败时

检查 `/etc/bluetooth/main.conf` 中的 `Privacy` 设置。它应该显示 `Privacy = device`。携带 `Privacy = off` 的板可能根本拒绝绑定手柄——连接以 `le-connection-abort-by-local` 放弃，手柄永远不会到达 `Paired: yes`。

如果相反，捕获显示配对到达 `DHKey check failed (0x0b)`，那是相反的故障，`Privacy = off` 是在那个板上要尝试的值。在更改值之前进行 `btmon` 捕获，因为两者都见过，它们需要不同的答案。

```bash
sudo sh scripts/setup-board.sh
```

```bash
sudo reboot
```

`setup-board.sh` 纠正该值，并且它在重启后才生效。

否则通常的原因是手柄在交换开始前离开了配对模式：再次按 Sync 并在灯仍然快速闪烁时重新运行。要查看交换本身：

```bash
sudo btmon -t > /tmp/btmon.log 2>&1 &
```

配对，然后 `sudo pkill btmon` 并寻找 `SMP: Pairing Failed` 及其旁边的原因。那是区分板设置和没有在听的手柄的唯一仪器。

## 当你驾驶时它掉线

观看手柄自己的输入流，它已经在板上：

```bash
robotctl monitor
```

按 `p`。打开的块是来自 `padd` 正在驱动的手柄的原始 evdev 流——每个报告，由内核加时间戳：

```
┌ pad Xbox Wireless Controller · /dev/input/event5 · 78:86:2e:bb:13:28 ─────────────┐
│ cadence  124/s while driving · last 8 ms ago · worst 84 ms · over 100 ms 0 · …    │
│ gap ms                                       ▁▁▁▁▂▁▁▁▁▃▁▁▁▁▁▁▂▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁ │
│ X     ····│██··   -8734 Y     ····│····       0 Z     ·········       0 …         │
│ held     BTN_START                                                                │
└ 4213 reports · gap ≤100 ms ───────────────────────────────── reports intact ──────┘
```

移动摇杆，条形跟随它们。轨迹的用途是上面的行：每个报告一个条形，因此停滞是一个尖峰，而已经恢复的仍然在屏幕上。全高是 100 ms——驱动程序开始感觉到它的点——超过 500 ms `robotd` 已经将速度归零。

**机器人上没有其他东西能显示这个。** `padd` 以 50 Hz 重发最后的摇杆值，因此已经停止交付的无线电在下游的任何地方看起来仍然像一个活的驱动程序：`robot.state` 携带新鲜的意图，deadman 永远不会触发，机器人继续在没有人给出的命令上行走。当这个块平线时，上面块中的 `asked` 列看起来会完美。

静止的手柄什么都不发送，因此沉默只有在你驾驶时才是证据。该块在五秒后说 `the sticks are still` 而不是指责链接，并分别计算那些时段。

然后，对于窗口上的判定而不是实时画面，从这个仓库的克隆将测量复制到板上：

```bash
scp scripts/pad-link-test.sh radxa@<board>:/tmp/
```

已经发生的事情，来自 `padd` 的日志——不需要手柄，它立即回答：

```bash
sudo sh /tmp/pad-link-test.sh --history
```

要现在测量链接，手柄开机且 `padd` 运行。**在整个两分钟内保持摇杆移动**：静止的手柄什么都不发送，沉默读起来完全像停滞的链接。

```bash
sudo sh /tmp/pad-link-test.sh
```

它计数掉线，以及连接时手柄输入报告之间的间隔。超过 500 ms 的间隔是机器人停止——`robotd` 在那里将速度归零。每次掉线后面跟着内核的原因：`0x08` 是监控超时，意味着范围或干扰，`0x13` 意味着有人关掉了手柄。

放下手柄不是停滞，也不计为停滞——但那是测量什么都学不到的时间，因此报告说明你实际驾驶了窗口的多少，并拒绝判断它几乎没看到的链接。

在机器人观看时走开是你找到范围的方式。

## 这个板和那个板运行的是同一个栈吗

在一个机器人上停滞而在其双胞胎上不停滞的手柄通常不是手柄。相隔数周构建的两个板运行不同的内核、不同的 BlueZ、不同的控制器固件，以及手柄上不同的手柄固件——而这些在 `pad status` 中都不可见。

将报告复制到每个板上，从这个仓库的克隆：

```bash
scp scripts/pad-stack-report.sh radxa@<board>:/tmp/
```

```bash
sudo sh /tmp/pad-stack-report.sh
```

它打印整个栈并将相同的文本保存到 `/tmp/pad-stack-<host>-<when>.log`：内核、BlueZ、适配器的 HCI 版本、它是哪个无线电以及内核在启动时为它加载的固件、什么在承载 HID、绑定持有哪些密钥、当前使用的传输，以及手柄自己的固件修订。它无需 root 运行，并对需要它的三件事说 `unreadable`。

要比较两个板，只向每个询问必须匹配的值：

```bash
ssh radxa@<board-a> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/a.fp
```

```bash
ssh radxa@<board-b> sudo sh /tmp/pad-stack-report.sh --fingerprint > /tmp/b.fp
```

```bash
diff /tmp/a.fp /tmp/b.fp
```

没有输出意味着相同的栈。指纹不携带时间戳也不携带地址，因此 `diff` 打印的任何东西都是真正的差异。

在其余之前要读的两行。`transport` 到目前为止在每个试过的手柄上都是 `LE`，而说 `BR/EDR` 的板正在将手柄通过内核的经典 HID 路径而不是 BlueZ 的——不同的驱动程序，不同的按钮编号。`input` 是 `Bus`/`Vendor`/`Product`/`Version` 四元组，SDL 和 `gilrs` 将其哈希成映射 GUID：在那里不同的两个板有不同的轴和按钮映射，无论其他什么匹配。

---

驾驶——控制和速度限制——在[速查表](cheatsheet.md#gamepad-configd)中；从笔记本电脑通过转发的 socket 运行 `padd` 在[开发速查表](cheatsheet-dev.md#from-a-laptop--drive-with-a-pad-in-your-hands)中。
#（注：内容由AI生成）
