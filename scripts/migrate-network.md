# migrate-network.sh

## 文件位置

`d:\microduck\scripts\migrate-network.sh`

## 核心设计决策

该脚本将 wifi 从 netplan 迁移到 NetworkManager，仅在到达 Armbian 库存网络的板端上运行一次。

- **为什么从 setup-board.sh 拆出**：
  1. **不同生命周期**：setup-board.sh 中的 overlay 修复和 ONNX 安装是永久的（每块板永远需要）。此脚本只因 Armbian 库存镜像带 netplan + systemd-networkd + wpa_supplicant 而存在。哪天构建自带 NetworkManager 的机器人镜像，整个文件删除，其他不变。
  2. **不同风险**：setup-board.sh 中一切随时可安全运行。此脚本是唯一能使无头板端不可达的步骤，需要重启生效。应在显式决策后而非常规初始化中运行。
- **为什么用 NetworkManager**：netplan 是配置生成器而非运行时网络管理器。它无扫描 API，`netplan apply` 报告"配置已应用"而非关联是否成功。"显示网络"和"密码错误"是手机配置机器人最需要的两个功能，所以 wifi 归 NetworkManager。以太网保持 netplan 和 networkd，不动。
- **运行两次**（重启前后各一次）：第二次运行撤销 backstop。

## 常量/参数分析

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `NM_CONF_DIR` | `/etc/NetworkManager/conf.d` | NetworkManager 配置目录 |
| `NM_PROFILE` | `robot-wifi` | NM 配置文件名，重跑修改而非累积 |
| `NET_CHECK` | `/usr/local/sbin/robot-net-check` | 启动时回退脚本 |
| `NET_CHECK_UNIT` | `/etc/systemd/system/robot-net-check.service` | backstop unit |
| `SELF` | `/usr/local/sbin/robot-migrate-network` | 脚本自拷贝位置 |
| `REPO` | `pollen-robotics/microduck` | 仓库（可被 DUCK_REPO 覆盖） |
| `REF` | `main` | 分支（可被 DUCK_REF 覆盖） |

## 核心函数

### install_networkmanager

安装前先写 `99-robot-wifi-only.conf` 设 `unmanaged-devices=*`，防止 NM 启动后立即与 netplan 的 wpa_supplicant 争抢 wlan0（两个 supplicant 在一个 netdev 上，运行此脚本的链路会断）。

### configure_nm_dns

根据 systemd-resolved 是否运行设置 `dns=systemd-resolved` 或 `dns=none`，`rc-manager=unmanaged`。独立于 `99-robot-wifi-only.conf` 另存为 `98-robot-dns.conf`，因为 backstop 会重写前者。

### credentials_from_wpa_conf / credentials_from_netplan_yaml

从 `/run/netplan/wpa-wlan0.conf`（优先，netplan 自己的翻译，扁平正则可解析，已解析密钥管理）或 netplan YAML 中提取 SSID、PSK、密钥管理方式。

WPA3 SAE 处理：检测 `key_mgmt=SAE` 或 `key-management: sae`，设为 `sae` 而非 `wpa-psk`，否则配置文件创建后静默从不关联。

### migrate_wifi_profile

将凭证迁移到 NM 配置。若配置已存在则跳过。

### arm_net_check / retire_net_check

**backstop 机制**：cutover 前武装，重启后 wifi 证明归 NM 后撤销。
- `robot-net-check` 脚本：启动后 90 秒内若 wlan0 无 IPv4 地址，恢复 netplan 文件（重命名 `.disabled` 回来），设 NM 管理所有设备，`netplan generate`，重启。
- 所有失败路径都回退：中途死亡的 backstop 会使板端变暗，正是它要防止的结果。

### cut_over

1. 读取 netplan wifi 文件，迁移凭证（失败则拒绝，不改变任何东西）。
2. 有 wifi 网络时才武装 backstop（无凭证的板端 wlan0 永无地址，武装会无限重启）。
3. 重命名 netplan 文件为 `.disabled`（而非屏蔽生成的 unit，停止生成器发出 netplan-wpa-wlan0.service 和 10-netplan-wlan0.network）。
4. 写 `99-robot-wifi-only.conf` 设 `unmanaged-devices=*,except:type:wifi`。
5. `netplan generate`（不 apply，不 reload NM，使运行链路保持到重启）。

### mask_networkd_wait_online

Armbian 自带 drop-in 将 `systemd-networkd-wait-online` 变为 `--any`。wifi 归 NM 后 networkd 唯一链路是通常无电缆的以太网，`--any` 永不可满足，每次启动烧完超时。mask 它，`NetworkManager-wait-online` 是诚实门控。

## 关键要点总结

1. 必须 root 运行，需要 systemctl、apt-get、install、mktemp。
2. 安装 NM 前先设"管理空"，防止两个 supplicant 争抢导致链路断开。
3. backstop 90 秒内 wlan0 无地址则回退 netplan 并重启，所有失败路径都回退。
4. 凭证优先从 `/run/netplan/wpa-wlan0.conf` 读取，其次从 YAML 解析。
5. WPA3 SAE 必须正确识别，否则配置静默不关联。
6. netplan 文件重命名为 `.disabled` 而非删除，作为手动撤销。
7. 重启后必须重跑此脚本撤销 backstop，否则以后任何 wifi 慢的启动都会回退。
8. `main "$@"` 在最后一行，防止 `curl | sh` 截断只定义函数不执行。
