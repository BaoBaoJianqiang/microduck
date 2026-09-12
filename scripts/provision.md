# provision.sh

## 文件位置

`d:\microduck\scripts\provision.sh`

## 核心设计决策

该脚本端到端配置刚刷好的板端，在重启允许的最少命令内完成。

- **编排而非复制**：调用 `setup-board.sh`、`migrate-network.sh`、`install.sh`，不复制它们。它们保持可单独运行，各有不同生命周期和风险。此脚本提供它们缺失的串联。
- **两阶段（重启分割）**：
  - **Phase 1**：板端和网络（device-tree overlay、wifi 迁移），需重启生效。
  - **Phase 2**：确认板端、安装 daemon（GStreamer、rkaiq、install.sh）。
- **总是要求重启**：不是因为总需要，而是判断是否需要意味着重新推导两个脚本已决定的事或解析其输出，两者都会漂移。重启代价 30 秒，收益：Phase 2 运行在活动启动配置上，重启后的 shell 是新登录会话（robot 组自动生效）。
- **无人值守循环防护**：
  1. resume unit 在做任何工作前先 disable 自己，最多一次自动尝试。
  2. `migrate-network.sh` 只在 NetworkManager 已拥有 wifi 时重跑（即 cutover 成功，只为撤销 backstop）。若 backstop 触发恢复了 netplan，不重跑（避免循环）。

## 常量/参数分析

### 环境变量（与 install.sh 同名，传递给下游）

| 变量 | 默认值 | 说明 |
|---|---|---|
| `DUCK_REPO` | `pollen-robotics/microduck` | 仓库 |
| `DUCK_REF` | `main` | 脚本来源分支 |
| `DUCK_TOKEN` | - | GitHub token |
| `DUCK_DEV_KEY` | - | team.dev.pub 路径 |
| `DUCK_FORCE_REINSTALL` | - | 强制重装 |
| `DUCK_WEIRD_BLE` | - | 蓝牙 workaround |
| `DUCK_GSTREAMER` | `1` | 安装 GStreamer 栈（默认开） |
| `DUCK_RKAIQ` | `1` | 安装 rkaiq 3A 引擎（默认开） |
| `DUCK_NO_REBOOT` | - | 不自动重启 |

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--name NAME` | 机器人名称（可选，默认从 SoC 序列号派生 `duck-xxxx`） |
| `--resumed` | systemd 重启后调用 |

### 关键路径

| 路径 | 说明 |
|---|---|
| `SELF` | `/usr/local/sbin/robot-provision` |
| `STATE` | `/var/lib/robot/provision.env`（0600） | 状态文件，跨重启携带 token 等 |
| `LOG` | `/var/lib/robot/provision.log`（640 root:robot） | Phase 2 日志 |
| `UNIT` | `/etc/systemd/system/robot-provision.service` | 重启后恢复 unit |
| `DEV_KEY_KEPT` | `/var/lib/robot/team.dev.pub` | dev 密钥跨重启存放 |

## 核心逻辑

### save_state / load_state

- `save_state`：将 REPO、REF、TOKEN、DEV_KEY、FORCE、WEIRD_BLE、GSTREAMER、ASKED_REF、NAME、BOOT_ID 写入 STATE 文件（0600，先创建再 chmod 再写密钥）。
- `load_state`：source STATE 文件，但操作员实际输入的环境变量优先于文件（错误的 token 可在 Phase 2 命令行纠正）。

### boot_id / same_boot_as_phase_one

用 `/proc/sys/kernel/random/boot_id` 判断是否真重启（板端无电池 RTC，时钟从 1970 开始，NTP 在配置期间调整，文件时间比较不可靠）。

### create_group（Phase 1）

在 Phase 1 就创建 robot 组并加入操作员，这样重启后新登录会话自动有组（无需 `newgrp`）。install.sh 仍安装 sysusers.d，但发现组已存在。

### install_resume_unit

创建 `robot-provision.service`：
- `ConditionPathExists=${STATE}`（finish 删除 STATE，防止重跑）。
- `Wants/After=network-online.target`。
- `ExecStart=${SELF} --resumed`。
- 输出 append 到 LOG（640 root:robot，操作员可读）。
- `TimeoutStartSec=1800`。

### phase_one

1. keep_dev_key（将 /tmp 中的 dev key 移到 STATE_DIR）。
2. create_group。
3. 运行 setup-board.sh。
4. 运行 migrate-network.sh。
5. save_state。
6. install_resume_unit + reboot_now（10 秒延迟，Ctrl-C 可取消）。

### phase_two

1. 重跑 setup-board.sh（用持久化副本）。
2. 运行 setup-gstreamer.sh（若 GSTREAMER 非 0）。
3. 运行 setup-rkaiq.sh + rkaiq-modinfo-shim.c（若 RKAIQ 非 0）。
4. 条件运行 migrate-network.sh（NM 已拥有 wifi 才跑，撤销 backstop）。
5. 运行 install.sh（传递所有 DUCK_* 变量）。
6. apply_asked_ref（若指定了 --ref，安装分支构建在 stable 之上）。
7. name_the_robot（通过 configd socket 设名）。
8. finish（删除 STATE，报告 token 去向）。

### apply_asked_ref

- install.sh 只装 stable（`/releases/latest` 排除预发布）。
- 分支构建装在 stable 之上：stable 成为 golden（boot recovery 的 fallback），current 是分支。
- 致命（非警告）：要求跑分支却静默跑 stable 是最难调试的失败。
- 检查运行的版本（非 exit status，因为 postinstall 重启 updaterd 会断开连接导致非零退出）。
- 等 updaterd 恢复（90s），再等 5s（健康门控回滚窗口），读 current 符号链接判断是否为 dev 版本。

## 关键要点总结

1. 必须从文件运行（非 pipe），因为 Phase 2 需在重启后有文件可运行。
2. STATE 文件 0600 跨重启携带 token，finish 删除；但 install.sh 写的 systemd drop-in 中的 token 保留（updaterd 需要它获取后续更新）。
3. robot 组在 Phase 1 创建，使重启后操作员自动有组成员身份。
4. GStreamer 和 rkaiq 默认开启，在 Phase 2 运行（不改启动配置，不需重启）。
5. resume unit 带 ConditionPathExists，finish 删除 STATE 防止意外重跑。
6. Phase 2 日志写文件（journald 持久化由 release 的 drop-in 配置，首次启动可能 RAM-only）。
7. `main "$@"` 在最后一行。
