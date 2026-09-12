# pad-stack-report.sh

## 文件位置

`d:\microduck\scripts\pad-stack-report.sh`

## 核心设计决策

该脚本报告板端驱动游戏手柄的整个蓝牙栈，生成可日志记录且可与另一板端 diff 的报告。

- **回答的问题不是"手柄能用吗"**（`pad-link-test.sh` 测量那个），而是"这两块板一样吗"。相隔数周构建的两个机器人运行不同内核、BlueZ、控制器固件和手柄固件，每一个都会改变手柄行为而不改变任何人输入的代码。
- **为什么有 fingerprint 模式**：完整报告供阅读，包含易变部分（谁连接了、电池电量、运行时间）。这些在 `diff` 中无法保留，满是时间戳的 diff 没人看。`--fingerprint` 只打印两块运行相同栈的板端之间必须匹配的值，固定顺序，无时间戳无地址。
- **两种模式来自同一收集过程**：不会不一致。
- **永不因缺工具失败**：缺失的 `hcitool` 或不可读的 bond 目录打印为一行说明，因为"absent"和一个值之间的 diff 正是此脚本要揭示的发现。root 多获得三样东西：适配器 HCI 版本、bond 的密钥类型、BlueZ 不愿说时的手柄固件字段。

## 常量/参数分析

### 环境变量（可覆盖路径，便于笔记本上用捕获文件测试）

| 变量 | 默认值 |
|---|---|
| `PAD_INPUT_DEVICES` | `/proc/bus/input/devices` |
| `PAD_BT_CONF` | `/etc/bluetooth/main.conf` |
| `PAD_BT_LIB` | `/var/lib/bluetooth` |
| `PAD_SYS_BT` | `/sys/class/bluetooth` |
| `PAD_OS_RELEASE` | `/etc/os-release` |
| `PAD_PROC_MODULES` | `/proc/modules` |

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--mac ADDR` | 多个手柄已绑定时指定哪个（默认：先连接的） |
| `--out FILE` | 日志输出文件（默认 `/tmp/pad-stack-<host>-<when>.log`） |
| `--fingerprint` | 只打印必须匹配的值，供 diff |

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `BUS_BLUETOOTH` | `0005` | /proc/bus/input/devices 中的蓝牙总线号 |
| `BUS_USB` | `0003` | USB 总线号 |
| `WATCHED_MODULES` | `bluetooth hci_uart btusb btrtl btbcm hidp uhid joydev` | 决定手柄如何到达用户态的模块 |

## 核心函数

### 辅助函数

| 函数 | 功能 |
|---|---|
| `field` | 打印对齐的标签值对，自动 trim 尾部空白（diff 中尾部空白不可见，是必须杜绝的） |
| `dump` | 缩进打印多行输出，空白输入不打印 |
| `trim` | 去除尾部空白 |
| `have` | 检查命令是否存在 |
| `bt` | 包装 `bluetoothctl`，加 5 秒超时（在 bluetoothd 坏掉的板上不会挂起），非零状态不退出 |

### 报告章节函数

| 函数 | 内容 |
|---|---|
| `section_report` | 脚本版本、时间、主机、OS、内核、uptime |
| `section_stack` | BlueZ 版本、btmon 版本、bluetoothd 状态、Privacy 设置、已加载模块 |
| `section_adapter` | 控制器、HCI 版本（btmgmt 或 hciconfig）、settings、芯片标识（sysfs）、控制器固件（journalctl/dmesg） |
| `section_pads` | 每个手柄：BlueZ info、bond 密钥类型、实时链路、地址类型、modalias、传输方式判定、input 设备 |
| `section_daemons` | padd/btd/configd/robotd 状态、robotctl 版本、pad status |
| `section_fingerprint` | 固定顺序的 14 个必须匹配的值 |

### 传输方式判定逻辑

bond 目录中的密钥类型是唯一确定性答案：
- LE bond：`LongTermKey` / `PeripheralLongTermKey` / `SlaveLongTermKey`
- BR/EDR bond：`LinkKey`
- 两者都有：dual

实时链路用 `hcitool con` 判断 LE 或 ACL。

## 关键要点总结

1. `--fingerprint` 模式仍收集完整报告（值来自那里），只是不打印易变部分。
2. 报告同时写入日志文件和终端，保存路径输出到 stderr（不干扰 stdout 的 diff）。
3. 手柄识别启发式镜像 `configd/src/pad.rs` 的 `looks_like_a_gamepad`：input-gaming 图标 → 0x03c4 appearance → 名称匹配。
4. bond 信息打印所有 section 名而非预设列表，因为 BlueZ 版本可能以未预料的名称存储密钥。
5. 所有 `bluetoothctl` 调用经 `bt()` 包装加超时，防止 bluetoothd 挂起时报告卡死。
6. `set -eu` 模式下每个可能失败的命令都用 `|| true` 保护，确保报告永不中途退出。
