# 手柄配对的最小设置

记录于 2026-08-18，在 Radxa Zero 3W `50:37:CD:16:2A:39` 上，使用 Xbox 无线控制器 `78:86:2E:92:47:67`。在第二块使用相同卡的 Zero 3W 上确认。

## 能工作的序列

刷入 Radxa Zero 3 的 Armbian，Minimal 版本。在镜像工具中填写 wifi 和用户名。不安装其他任何东西——没有守护进程，没有配置。

```bash
sudo sed -i -E 's|^[[:space:]]*#?[[:space:]]*Privacy[[:space:]]*=.*|Privacy = device|' /etc/bluetooth/main.conf
```

```bash
grep -n "^Privacy" /etc/bluetooth/main.conf
```

```bash
sudo reboot
```

用重启而不是 `systemctl restart bluetooth`：重启有时会让内核保持持有 hci0 并报 `No default controller available`，只有重启能清除。

按住手柄的配对按钮直到它快速闪烁，然后：

```bash
bluetoothctl
```

```
scan on
```

等待 `[NEW] Device 78:86:2E:92:47:67 Xbox Wireless Controller`，然后：

```
scan off
```

```
connect 78:86:2E:92:47:67
```

对 `Request authorization` 回答 `yes`，然后：

```
trust 78:86:2E:92:47:67
```

`pair` 永远不会被输入。

## 成功时是什么样子

```bash
ls /dev/input/js*
```

```bash
dmesg | tail -3
```

```
input: Xbox Wireless Controller as /devices/virtual/misc/uhid/0005:045E:0B13.0001/input/input5
microsoft 0005:045E:0B13.0001: input,hidraw0: BLUETOOTH HID v5.09 Gamepad [Xbox Wireless Controller]
```

输入实际在流动——全程移动左摇杆，越过前 184 字节寻找 `type 0x02` 和时间戳递增的事件：

```bash
sudo timeout 5 cat /dev/input/js0 | od -Ad -tx1 | head -20
```

绑定在手柄电源循环后仍然存在（按住 Xbox 按钮约 6 秒，然后重新打开）：

```bash
ls /dev/input/js*; bluetoothctl info 78:86:2E:92:47:67 | grep -E "Connected|Bonded|Trusted"
```

## 它能工作的板

不是任何人预期的配置，这就是为什么每个都被写下来：

| | |
|---|---|
| `cat /sys/module/aic8800_bsp/srcversion` | `738316A2E9D9825966BDB6B` (86016) |
| `conn_min_interval` / `conn_max_interval` | 24 / 40 — 内核默认，30–50 ms |
| `/etc/bluetooth/main.conf` | `Privacy = device` |
| 运行的守护进程 | 无 |

驱动是 `design/pad-bond-failure.md` 称为损坏的那个构建。连接间隔未被改动。两者都不是决定手柄是否配对的因素。

## 失败了什么

| 设置 | 结果 |
|---|---|
| 裸 Armbian，`Privacy = off`（或未设置——BlueZ 默认为 `off`） | `connect` 返回 `le-connection-abort-by-local`；`Paired: no`；完全没有 SMP 交换 |
| 裸 Armbian，没有 `Privacy` 行，先用 `pair` 而不是 `connect` | 配对，然后每次重连失败 `Encryption Change: PIN or Key Missing (0x06)`，约 1 秒抖动一次 |
| 已配置的板（`Privacy = device` 在第 99 行确认），`padd` 和 `btd` 已停止，手动 `connect` | `Request authorization` 被接受，然后 `ServicesResolved: no`——手柄立即断开，没有 `js0` |

第三行是开放的那个：相同的卡镜像和相同的 `Privacy` 值，一旦板被配置就失败。因此是配置改变了某些东西，而不是一个恰好在运行的进程——停止 `padd` 和 `btd` 没有恢复配对。

## 建立绑定和保持绑定是不同的

在一切运行的完全配置的板上，一个**已经配对**的手柄可以连接并驱动。一个**全新的** `robotctl pad pair`，在 `pad forget` 之后且手柄保持在配对模式下，失败。

因此这里没有任何东西破坏现有绑定。安装的系统中有某些东西阻止建立新的绑定。`configd` 和 `btd` 日志对此没有任何有用的说明。

## 明天：一次加一样东西

从一块重新刷入并通过本页顶部序列确认能工作的板开始，添加一层，重启，在添加下一层之前尝试一次全新配对。

把脚本复制到 `~`，**不是** `/tmp`——这里的每一步都重启，而 `/tmp` 一次重启都不能幸存。任何值得保留的 `btmon` 抓包也是如此。

```bash
scp scripts/setup-board.sh scripts/migrate-network.sh pierre@BOARD:~/
```

| 步骤 | 添加什么 | 手柄配对？ | 备注 |
|---|---|---|---|
| 0 | 什么都没有——上面的最小序列 | 是 | 对照组 |
| 1 | `sudo sh ~/setup-board.sh`——overlay、`console=display`、getty 掩码、onnxruntime | **是** | |
| 2 | `sudo sh ~/migrate-network.sh`——netplan → NetworkManager | **是** | |
| 3 | `sudo -E sh ~/install.sh` | **否** | 第一次尝试；守护进程在运行 |
| 3b | `DUCK_NO_START=1`，然后重启 | **是** | 因此 install.sh 写入的任何东西都不是过错 |
| 4 | `systemctl enable --now updaterd` | | |
| 5 | `... robotd` | | |
| 6 | `... configd` | | |
| 7 | `... btd` | | |
| 8 | `... padd` | | |

于 2026-08-19 运行，在每一步用 `bluetoothctl` 手动配对，之后再次移除手柄。步骤 1 和 2 配对正常；`install.sh` 是它停止的地方。因此板启动、设备树 overlay、控制台移动和 NetworkManager 切换都被排除了。

于 2026-08-19 在 `install.sh` 内部分割：用 `DUCK_NO_START=1` 并重启，手柄配对。因此 `install.sh` 写入磁盘的任何东西都不是过错——不是单元、用户、组、发布树或 token drop-in。是五个守护进程中的一个在运行。

在它工作之前，三次测量尝试都浪费了，全部因为同一个原因：release 内的 `hooks/postinstall` 对它交付的每个单元执行 `systemctl enable --now`，从 `updaterd install` 内部，这发生在 `install_units` *之前*——因此守护进程在每次测试期间都在运行，而之后的 `systemctl disable --now` 不会撤销它们推送到适配器的东西。`btd` 留下 `Pairable` 设置、一个广播实例，以及其默认配对代理给适配器的 IO 能力。**测量前重启。**

## 是 `btd`

2026-08-19，一个变量，在 `2A:39` 上两个方向都可复现：

| | |
|---|---|
| `btd` 启用，重启，手柄重置，全新配对 | **失败** |
| `btd` 禁用，重启，手柄重置，全新配对 | **成功** |

工作运行中**没有**清除 `/var/lib/bluetooth`，因此 `btd` 导致 BlueZ 写入的两个文件——`attributes`，持久化的本地 GATT 数据库，和 `identity`，适配器的本地 IRK——都被排除了。其他任何持久化的东西也是。

更早一轮禁用 `btd` *没有*恢复配对，是手柄自己的绑定槽：Xbox 手柄持有一个主机绑定，半完成的尝试让它持有一个板不再拥有的密钥。**在尝试之间在笔记本上重置手柄**，否则故障与中毒的手柄无法区分——这花费了将近两天。

没有任何运行的东西阻止现有绑定工作：配对的手柄在整个栈启动时连接并驱动。只有*建立*绑定失败。

### 机制，尚未测量

`btd` 作为外设持续广播。在 `Privacy = device` 下，该广播使用可解析私有地址，而同一个适配器作为中心来配对手柄。SMP DHKey 检查是在两个设备的地址上计算的，因此这就是产生 `DHKey check failed (0x0b)` 的形状——这个失败最初让这个树放弃了 `Privacy = device`，当时 `btd` 在运行。对失败配对的 `btmon` 抓包，读取自身地址类型和 SMP 失败原因，就能解决它。

### 两个故障，它们被读成了一个

用重置后的手柄重新测试，因此两个结果都不依赖中毒的绑定槽：

1. **板分裂。** 在什么都没安装的全新 Armbian 上，一些 Zero 3W 单元在 BlueZ 默认的 `Privacy = off` 下配对手柄。另一些在 `off` 下完全不配对，只有 `device` 能工作。每组大约十个中的一半，没有可测量的东西区分它们。
2. **`device` 对比 `btd`。** 在 `device` 下，当 `btd` 广播时手柄无法形成新绑定。

它们一起解释了最初让这个树设置 `Privacy = off` 的 `DHKey check failed (0x0b)`：那是故障 2，在一块因故障 1 而需要 `device` 的板上测量，它被读为 `device` 破坏了配对的证据。结果是一个在一半板上都无法配对手柄的设置，持续了两周。

单独停止 `btd` 不够：它推送到控制器的东西比进程存活更久，因此每次成功的手动配对都在停止后有一次**重启**。适配器电源循环（`bluetoothctl power off && power on`）可以替代那次重启——2026-08-19 测量，`pad pair` 在完全安装的板上第一次尝试就成功——这就是让配对保持一条命令的原因。

两个变通方案都在 `provision-board.sh --weird-ble` 后面，因此不需要它们的板不携带它们：该标志设置 `Privacy = device` 并留下 `/var/lib/robot/weird-ble`，而 `robotctl pad pair` 只在有该标记的板上暂停 `btd`。先尝试没有标志的板。当 aic8800 消失时两者都消失。

每步后重启，并在每次尝试前清除绑定的**两半**：板上的 `pad forget` 或 `bluetoothctl remove`，以及保持在配对模式的手柄。Xbox 手柄保持一个主机绑定，半完成的尝试让它持有一个板不再拥有的密钥——这看起来和故障一模一样。

步骤 3 需要 `install.sh` 读取的环境：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
export DUCK_REF=pad-privacy-device-not-off
```

```bash
export DUCK_DEV_KEY=$HOME/team.dev.pub
```

步骤 1 和 2 既不需要 token 也不需要网络。

## 与 `microduck_runtime` 的未测试差异

`microduck_runtime` 的安装器在活动的 NetworkManager 连接上禁用 wifi 省电（`install.sh:244`，`:383`）：

```
sudo nmcli con mod "$WIFI_CON" wifi.powersave 2
```

`scripts/` 没有等效项。aic8800 是一个组合 wifi 和蓝牙部件，通过 SDIO 共享一个无线电，因此这是上面第三行的候选——但能工作的裸板上的值从未被读取，因此它是候选而不是发现。

```bash
iw dev wlan0 get power_save
```
#（注：内容由AI生成）
