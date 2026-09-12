# install.sh

## 文件位置

`d:\microduck\scripts\install.sh`

## 核心设计决策

该脚本在全新板端上从零安装机器人守护进程。

- **打破循环**："更新需要 updater，而 updater 随更新到达"的循环通过下载一个裸 `updaterd` 二进制并运行其 `install` 子命令打破。该命令运行普通引擎：签名验证、提取、原子交换、日志记录。无 bootstrap 特定安装逻辑，不会与后续更新行为漂移。
- **永不解析 manifest**：交给 updaterd 配置，让配置源解析 `latest`，因为 shell 脚本从签名 JSON 中挑版本是第二个更弱的读取者。
- **只从仓库取两个文件**：配置和公钥（验证前必需）。unit 文件和 journald drop-in 从已安装的 release 中取——与签名检查过的相同字节。

### 信任链（4 步）

1. TLS 到 raw.githubusercontent.com 获取脚本、配置和公钥。
2. TLS 到 github.com 获取 bootstrap updaterd（**尚未验证**）。
3. 该二进制验证 manifest 和制品对步骤 1 密钥的签名，拒绝未签名的内容。
4. 比较 bootstrap 二进制 sha256 与 `current/bin/updaterd`（来自已验证制品）。相等则 bootstrap 也是真的。CI 断言两者字节相同，不匹配是真实发现。

## 常量/参数分析

### 环境变量

| 变量 | 说明 |
|---|---|
| `DUCK_REPO` | 仓库，默认 `pollen-robotics/microduck` |
| `DUCK_REF` | 密钥和脚本来源的分支（默认 main）。**不**影响配置来源 |
| `DUCK_CONFIG_REF` | 配置来源，默认与安装的 release tag 一致 |
| `DUCK_TOKEN` | GitHub token（私有仓库） |
| `DUCK_FORCE_REINSTALL` | 强制重装（停止 daemon，无健康门控） |
| `DUCK_DEV_KEY` | team.dev.pub 公钥路径，信任 dev 签名 |
| `DUCK_NO_START` | 安装但不启动任何东西 |

### 关键常量

| 常量 | 值 | 说明 |
|---|---|---|
| `KEYS` | `release-1.pub release-2.pub release-3.pub` | 三个公钥（密钥轮换路径），release-1 必须存在 |
| `BOOTSTRAP_ASSET` | `updaterd-bootstrap-aarch64` | bootstrap 二进制资产名 |
| `CONFIG_DIR` | `/etc/robot` | 配置目录 |
| `KEYS_DIR` | `/etc/robot/trusted_keys` | 信任密钥目录 |
| `INSTALL_DIR` | `/opt/robot/daemon` | 安装目录 |
| `UNIT_DIR` | `/etc/systemd/system` | systemd unit 目录 |

## 核心函数

### resolve_bootstrap_asset

通过 GitHub API（非浏览器 URL，私有仓库浏览器 URL 返回 404）查找最新稳定 release 的 bootstrap 资产下载 URL。用 grep 解析 JSON（板端无 jq），提取资产 id 和 tag_name。

### install_config

- 密钥从 `REF`（默认 main）取（信任集只增长，最新最安全）。
- 配置从 `RELEASE_TAG`（正在安装的 release tag）取（配置字段只有同版本二进制才理解，配 main 配置给旧二进制会 `unknown field` 错误）。
- `updater.toml` 和 `robotd.toml` 永不覆盖（操作员编辑）。

### bootstrap_first_release

- 已有 release 且无 FORCE_REINSTALL：跳过，提示用 `robotctl update apply`。
- 已有 release 且 FORCE_REINSTALL：停止 daemon，强制安装（无健康门控，不能自动回滚）。
- 无 release：下载 bootstrap updaterd，运行 `updaterd install --config`，验证 sha256 与已安装 release 一致。

### create_group

- 从 release 的 `systemd/sysusers.d/*.conf` 安装服务账户。
- 手动创建 btd、padd 用户（systemd-sysusers 不可用时）和 robot 组。
- 将操作员（SUDO_USER）加入 robot 组（只读访问，非特权授予）。

### install_dev_key

信任 team.dev.pub 并启用 `allow_dev_keys = true`。文件名必须以 `.dev.pub` 结尾（否则被归为 release 密钥）。警告：此板将安装团队任何人推送的分支构建，不应用于发货机器人。

### install_units

- 从 release 目录读取所有 `.service` 和 `.timer`（不硬编码列表，因为脚本来自分支而 release 是最新稳定版）。
- 复制（非符号链接）到 `/etc/systemd/system`。
- 安装 journald drop-in、robotctl 符号链接、救援脚本（robot-rescue、robot-boot-check 复制到 /usr/local/sbin）、setup-login.sh。
- 按顺序 enable：updaterd、robotd、configd、btd、padd、mediad。btd/padd/mediad 允许失败。
- `robot-boot-check.timer` 用 `enable`（非 `--now`，避免在 provisioning 中途触发）。
- `robot-boot-check.service` 不 enable（无 [Install] 段，由 timer 启动）。

### install_token_dropin

写 `/etc/systemd/system/updaterd.service.d/token.conf`（mode 600，umask 077），设 `GITHUB_TOKEN`。`try-restart updaterd` 使其生效。仅在有 token 时（开发板），不用于客户机器人。

## 关键要点总结

1. 必须 root 且 aarch64 运行，需要 curl、systemctl、sha256sum、install。
2. 配置从 release tag 取而非 main，避免旧二进制不认识新字段。
3. bootstrap 二进制 sha256 与已验证 release 中的 updaterd 比对，关闭唯一未验证下载的信任缺口。
4. unit 文件复制而非符号链接，避免更新时 systemd 视图随 current 变化。
5. 操作员加入 robot 组后需 `newgrp robot`（进程组在 exec 时固定）。
6. `DUCK_NO_START` 用于隔离板端故障与 daemon，需重启才能得到干净测量。
7. `main "$@"` 在最后一行，防止 `curl | sh` 截断。
