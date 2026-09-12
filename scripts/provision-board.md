# provision-board.sh

## 文件位置

`d:\microduck\scripts\provision-board.sh`

## 核心设计决策

从操作员自己的机器上用一条命令配置板端。本目录中唯一在**操作员机器**而非机器人上运行的脚本，所有操作通过 ssh 完成，本机不安装任何东西。

- **解决中间断层**：`provision.sh` 重启板端并自行完成，从外部看就是 ssh 会话中断+未知间隔+猜测何时重连。本脚本等待板端恢复、流式输出无人值守阶段写的日志，最后落到 `robotctl health`，让配置成为一条命令而非中间有缺口的三条。
- **DHCP 租约迁移问题**：配置中间的 cutover 会让板端从 netplan 的租约切到 NetworkManager 的租约，从外部看就像板端没启动。等待是一场竞赛：ssh 轮询原地址 + BLE probe 问机器人自身地址（`net.status`），谁先回答用谁。
- **BLE probe 信任条件**：只有当**恰好一个机器人应答本板名称**时才采用其报告的地址。
- **本地模式**：`--local` 发送本 clone 的 `provision.sh`，使未推送的分支可测试（`--local --ref that-branch` 完整测试分支）。

## 命令行参数

| 参数 | 说明 |
|---|---|
| `[user@]host` | 板端地址（必填，推荐用地址而非名称，mDNS 不可靠） |
| `--ref BRANCH` | 从分支配置（分支脚本 + 分支构建装在 stable 之上，失败则配置失败） |
| `--name NAME` | 命名机器人（可选，默认 `duck-<4 hex>`） |
| `--forget-host-key` | 先从 known_hosts 删除该主机密钥（重刷卡后会变化） |
| `--local` | 发送本 clone 的 provision.sh |
| `--no-dev-key` | 不安装 team dev key |
| `--dev-key PATH` | 指定 dev key 路径 |
| `--no-ble` | 不用 Bluetooth 重新定位板端 |
| `--no-gstreamer` | 跳过 GStreamer 栈 |
| `--no-rkaiq` | 跳过相机 3A 引擎 |
| `--pause-btd-on-pair` | 配对时暂停 btd（仅停止 btd 并开关适配器，不改 Privacy） |
| `--weird-ble` | 蓝牙无法绑定手柄的板子（隐含 --pause-btd-on-pair + 设置 Privacy=device） |

## 核心函数分析

### ssh 封装

- `rsh`：非交互 ssh，`BatchMode=yes`、`ConnectTimeout=5`、`ServerAliveInterval=3`/`CountMax=2`（半启动网络栈握手后不说话时避免永久等待）、`ControlPath=none`（避免多路复用残留 master socket 指向已死连接）、`StrictHostKeyChecking=accept-new`。
- `alive N`：`timeout(1)` 不在 macOS 上，手写看门狗。`rsh true` 后台跑，每秒检查，超 N 秒 kill。子 shell 关闭 stderr 避免 `Terminated: 15` 作业控制通知。
- `scp_target`：处理 IPv6 字面量（scp 需方括号，ssh 不需），按冒号检测而非解析地址。

### 蓝牙 fallback

- `learn_ble_name`：重启前通过 ssh 获知机器人广播名称。两个来源：
  - `robotctl system info --json` 的 name 字段（已运行 daemon 的板端，包含 `system.setName` 的改名）。
  - 从 `/proc/device-tree/serial-number` 派生（精确镜像 `configd/src/identity.rs`：`duck-` + SHA-256 前两字节 hex）。
  - 无 readable serial 时留空（所有板端 hostname 相同，名称会撞车，探测不能猜）。
- `ble_probe_start`：后台 `cargo run -p duckctl --name NAME --pin PIN wifi status`，verdict 写临时文件再 rename（避免读到半行）。
- `ble_verdict`：读 verdict 并删除（单次消费）。
- `ble_stop`：杀 probe、清理 verdict（防止下次等待读到重启前的旧地址）。
- `wait_for_board`：墙钟计时（非 sleep 累加），与 BLE probe 竞赛。verdict 为 `ip <addr>` 时 adopt_address；`no-address`（机器人报告无 IPv4）；`failed`（失败原因）。verdict 后重臂 probe（最常见原因是 btd 未启动完）。

### still_provisioning

三值而非二值：
- `0`：仍在配置（STATE 文件存在，unit 未 failed）
- `1`：完成（STATE 文件不存在，板端可达）
- `2`：无法判断（板端不可达）
- `3`：STATE 存在但 `systemctl is-failed`（Phase 2 失败）

关键：STATE 文件在 Phase 2 失败时不删除（只在正常退出时删除），所以光看文件存在无法区分"还在跑"和"已失败"，必须问 systemd。

### adopt_address

切换 HOST/HOST_ONLY 到新地址，保留 `[user@]` 前缀。检测已知主机密钥冲突（回收的租约可能指向其他板端的密钥）并给明确提示。

### 日志流式输出

- `choose_log_reader`：决定日志读取方式（直接读或 `sudo -n`，不能用裸 sudo 因为 BatchMode ssh 无终端提示）。
- `drain_log`：读 `wc -c` 算大小，只输出新字节（`tail -c +N`），像流式。数字用 `tr -dc '0-9'` 过滤防止 ssh/sudo 杂行污染算术。
- 轮询而非 `tail -f`：连接要承受服务还在启动，每次重连的轮询不会持有死通道。

## 关键要点总结

1. **需要密钥访问**：配置后需自行重连，密码提示无法跨越重启，所以密钥访问必须。
2. **BLE fallback 限制**：需要 `cargo` + 本 clone（duckctl 是 example 而非安装的二进制）；首次配置时 btd 未安装，蓝牙前几分钟无应答；`net.status` 报告 wifi 接口，以太网不在覆盖范围。
3. **phase 1 命令 exit status 故意忽略**：命令以重启机器结束，ssh 报连接断开是预期结果。是否成功由后续等待板端判断。
4. **等待板端先等其下线**：重启中板端可能还应答一会儿，先等它下线再等上线。
5. **日志读取决定一次**：首次读时决定用不用 sudo -n，每次轮询不猜。
6. **2 分钟无新日志警告**：仍在等但提醒查看。
7. **Phase 2 失败**：不自行诊断，让用户看日志最后内容，给两条排查命令（`systemctl status` + `cat LOG`）。
