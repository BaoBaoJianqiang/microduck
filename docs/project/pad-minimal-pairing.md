# 手柄绑定的最小设置

记录于 2026-08-18，在 Radxa Zero 3W `50:37:CD:16:2A:39` 上，使用一个 Xbox Wireless Controller `78:86:2E:92:47:67`。在第二块同卡的 Zero 3W 上确认。

## 有效的序列

刷入 Radxa Zero 3 的 Armbian，Minimal。在 imager 中填写 wifi 与用户名。不安装其他任何东西 —— 无守护进程，无初始化。

```bash
sudo sed -i -E 's|^[[:space:]]*#?[[:space:]]*Privacy[[:space:]]*=.*|Privacy = device|' /etc/bluetooth/main.conf
```

```bash
grep -n "^Privacy" /etc/bluetooth/main.conf
```

```bash
sudo reboot
```

用重启而非 `systemctl restart bluetooth`：重启有时会让内核以 `No default controller available` 持有 hci0，且只有重启能清除它。

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

`pair` 从不键入。

## 成功时看起来什么样

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

输入实际流动 —— 全程移动左摇杆，并越过前 184 字节寻找 `type 0x02` 且时间戳递增的事件：

```bash
sudo timeout 5 cat /dev/input/js0 | od -Ad -tx1 | head -20
```

绑定在手柄电源循环后存活（按住 Xbox 按钮约 6 秒，然后重新打开）：

```bash
ls /dev/input/js*; bluetoothctl info 78:86:2E:92:47:67 | grep -E "Connected|Bonded|Trusted"
```

## 它工作的那块板子

不是任何人预期的配置，这就是为什么每个都被写下来：

| | |
|---|---|
| `cat /sys/module/aic8800_bsp/srcversion` | `738316A2E9D9825966BDB6B` (86016) |
| `conn_min_interval` / `conn_max_interval` | 24 / 40 —— 内核默认，30–50 ms |
| `/etc/bluetooth/main.conf` | `Privacy = device` |
| 运行的守护进程 | 无 |

驱动是 `design/pad-bond-failure.md` 称为坏掉的那个构建。连接间隔未被触碰。两者都不决定手柄是否绑定。

## 失败过什么

| 设置 | 结果 |
|---|---|
| 裸 Armbian，`Privacy = off`（或未设 —— BlueZ 默认为 `off`） | `connect` 返回 `le-connection-abort-by-local`；`Paired: no`；完全没有 SMP 交换 |
| 裸 Armbian，无 `Privacy` 行，以 `pair` 而非 `connect` 开头 | 绑定，然后每次重连失败 `Encryption Change: PIN or Key Missing (0x06)`，约每秒抖动一次 |
| 已初始化的板子（`Privacy = device` 在第 99 行确认），`padd` 与 `btd` 已停止，手动 `connect` | `Request authorization` 被接受，然后 `ServicesResolved: no` —— 手柄立即断开，无 `js0` |

第三行是开放的那个：同一张卡镜像与同一个 `Privacy` 值，一旦板子被初始化就失败。因此是初始化改变的某些东西，而非一个恰好运行的进程 —— 停止 `padd` 与 `btd` 没有让配对回来。

## 建立绑定与保持绑定是两回事

在一个一切运行的完全初始化的板子上，一个**已经绑定**的手柄可以连接并驱动。一个**全新的** `robotctl pad pair`，在 `pad forget` 后且手柄处于配对模式时，失败。

因此这里没有任何东西破坏一个已有的绑定。已安装系统中的某些东西阻止建立新绑定。`configd` 与 `btd` 日志对此没有任何有用的说法。

## 明天：一次加一样东西

从一块按本页顶部序列重新刷过并确认工作的板子开始，加一层，重启，在加下一层之前尝试一次全新配对。

把脚本复制到 `~`，**不是** `/tmp` —— 这里每一步都重启，而 `/tmp` 不存活重启。任何值得保留的 `btmon` 捕获也是。

```bash
scp scripts/setup-board.sh scripts/migrate-network.sh pierre@BOARD:~/
```

| 步骤 | 它添加什么 | 手柄配对？ | 备注 |
|---|---|---|---|
| 0 | 无 —— 上面的最小序列 | 是 | 对照 |
| 1 | `sudo sh ~/setup-board.sh` —— overlays、`console=display`、getty 掩码、onnxruntime | **是** | |
| 2 | `sudo sh ~/migrate-network.sh` —— netplan → NetworkManager | **是** | |
| 3 | `sudo -E sh ~/install.sh` | **否** | 第一次尝试；守护进程在运行 |
| 3b | `DUCK_NO_START=1`，然后重启 | **是** | 因此 install.sh 写入磁盘的东西都无过错 |
| 4 | `systemctl enable --now updaterd` | | |
| 5 | `... robotd` | | |
| 6 | `... configd` | | |
| 7 | `... btd` | | |
| 8 | `... padd` | | |

运行于 2026-08-19，在每一步用 `bluetoothctl` 手动配对并随后移除手柄。步骤 1 与 2 绑定正常；`install.sh` 是它停止的地方。因此板子启动、设备树 overlays、控制台移动与 NetworkManager 切换都被开脱。

2026-08-19 在 `install.sh` 内部分拆：用 `DUCK_NO_START=1` 加一次重启，手柄绑定。因此 `install.sh` 写入磁盘的东西都无过错 —— 不是 units、用户、组、版本树或 token drop-in。是五个守护进程中的某一个，在运行。

在它工作之前，三次那个测量的尝试被浪费了，全因同一个原因：版本内部的 `hooks/postinstall` 从 `updaterd install` 内部对它交付的每个 unit 做 `systemctl enable --now`，这发生在 `install_units` *之前* —— 因此守护进程在每次测试期间都起来了，而之后的 `systemctl disable --now` 不会撤销它们推给适配器的东西。`btd` 留下 `Pairable` 设置、一个广播实例，以及其默认配对代理给适配器的 IO 能力。**测量前重启。**

## 是 `btd`

2026-08-19，一个变量，在 `2A:39` 上双向可复现：

| | |
|---|---|
| `btd` 启用，重启，手柄重置，全新配对 | **失败** |
| `btd` 禁用，重启，手柄重置，全新配对 | **工作** |

工作那次运行**没有**擦除 `/var/lib/bluetooth`，因此 `btd` 导致 BlueZ 写入的两个文件 —— `attributes`，持久化的本地 GATT 数据库，与 `identity`，适配器的本地 IRK —— 都被开脱。其他任何持久化的东西也是。

早先一轮禁用 `btd` *没有*恢复配对，是手柄自己的绑定槽：一个 Xbox 手柄持有一个主机绑定，而一个半途而废的尝试让它持有一个板子不再有的密钥。**在尝试之间在笔记本上重置手柄**，否则故障与一个被污染的手柄无法区分 —— 这花掉了差不多两天。

没有任何运行的东西阻止一个已有绑定工作：一个已绑定的手柄在整个栈起来时连接并驱动。只有*建立*绑定失败。

### 机制，尚未测量

`btd` 作为外设持续广播。在 `Privacy = device` 下，该广播使用一个可解析私有地址，而同一个适配器作为中心来绑定手柄。SMP DHKey 检查是在两个设备的地址上计算的，因此这是产生 `DHKey check failed (0x0b)` 的形态 —— 正是当初在 `btd` 运行时让本树放弃 `Privacy = device` 的失败。一次失败配对的 `btmon` 捕获，读取自身地址类型与 SMP 失败原因，可以定论。

### 两个故障，且它们被读成一个

用一个重置过的手柄重新测试，因此两个结果都不依赖一个被污染的绑定槽：

1. **板子分裂。** 在一个什么都没安装的全新 Armbian 上，一些 Zero 3W 单元在 BlueZ 默认的 `Privacy = off` 下绑定手柄。另一些在 `off` 下完全不绑定，只有 `device` 工作。十块中大约每组一半，没有可测量的东西把它们分开。
2. **`device` 对 `btd`。** 在 `device` 下，当 `btd` 广播时手柄无法形成新绑定。

两者合起来解释了当初让本树设置 `Privacy = off` 的 `DHKey check failed (0x0b)`：那是故障 2，在一块需要 `device` 来应对故障 1 的板子上测量，且它被读为 `device` 破坏配对的证据。结果是一个在一半板子上无法绑定手柄的设置，持续了两周。

单独停止 `btd` 不够：它推给控制器的东西比进程活得久，因此每个成功的手动配对都在停止后有一次**重启**。一次适配器电源循环（`bluetoothctl power off && power on`）替代那次重启 —— 2026-08-19 测量，`pad pair` 在一块完全安装的板子上第一次尝试就成功 —— 这正是让配对保持为一个命令的原因。

两个变通方案都在 `provision-board.sh --weird-ble` 后面，因此不需要它们的板子不携带它们：该标志设置 `Privacy = device` 并留下 `/var/lib/robot/weird-ble`，而 `robotctl pad pair` 只在有该标记的板子上暂停 `btd`。先在没有标志的板子上试。两者在 aic8800 消失时一起消失。

每步后重启，并在每次尝试前清除绑定的**两半**：板子上的 `pad forget` 或 `bluetoothctl remove`，以及处于配对模式的手柄。一个 Xbox 手柄持有一个主机绑定，而一个半途而废的尝试让它持有一个板子不再有的密钥 —— 这看起来与故障完全一样。

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

步骤 1 与 2 既不需要 token 也不需要网络。

## 与 `microduck_runtime` 的未测试差异

`microduck_runtime` 的安装程序在活动的 NetworkManager 连接上禁用 wifi 省电（`install.sh:244`、`:383`）：

```
sudo nmcli con mod "$WIFI_CON" wifi.powersave 2
```

`scripts/` 中没有等价物。aic8800 是一个通过 SDIO 共享一个无线电的组合 wifi 与蓝牙部件，因此这是上面第三行的一个候选 —— 但工作的裸板子上的值从未被读取，因此它是一个候选而非发现。

```bash
iw dev wlan0 get power_save
```
