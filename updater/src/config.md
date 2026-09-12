# config.rs 文件解析

## 文件位置

`d:\microduck\updater\src\config.rs`

## 定位

每机器人配置。引擎通用；所有机器人特定逻辑在此。适配新机器人应只需新配置文件。

## `Config` 根结构

| 字段 | 说明 |
|---|---|
| `trusted_keys_dir` | minisign 公钥目录（集合而非单密钥，可应对密钥丢失/泄露） |
| `hw_rev` | 单一前向兼容守卫（非能力矩阵） |
| `state_dir` | 引擎状态（锁/日志/boot counter），**必须在 install_dir 之外** |
| `robot_socket` | robotd 监听处（机器人级单一事实） |
| `allow_dev_keys` | 接受 dev 密钥（生产关闭） |
| `allow_fault_injection` | 允许 `--inject-fault`（生产关闭） |
| `max_uncompressed_bytes` / `max_archive_entries` | 归档限制 |
| `check_interval` | 轮询间隔（None 禁用，也使 min_supported 失效） |
| `auto_apply` | 无人值守应用策略 |
| `allow_uids/gids` / `allow_users/groups` | 变更操作授权（按名优先，因 sysusers 动态分配 uid） |
| `components` | BTreeMap 组件配置 |

## `ComponentConfig`

`source` / `install_dir` / `on_apply` / `health` / `keep_previous`（默认 1）/ `golden` / `pinned`。

## `AutoApply` 枚举

有序设置而非每紧急度布尔：
- `Off` — 从不
- `Mandatory`（默认）— 仅 `min_supported` 地板以下的版本
- `All` — 所有版本（canary/bench）

防止"自动应用普通但不应用强制"这种无意义组合。

## `SourceConfig`

- `GithubReleases { repo, tag_prefix, manifest_asset, ref_tag_prefix, staging_tag_prefix }` — 三个前缀分隔稳定/分支/候选流，防混淆
- `HfHub { repo, revision, manifest_file }` — 模型通道
- `LocalDir { path }` — 非生产源（测试/旁加载）

## `ApplyAction`

- `None` — 无操作
- `Restart { units }` — 全重启（daemon 通道）
- `Reload { unit, signal }` — 原位信号（模型，不中断电机控制）

## `HealthCheck`

- `None` — 无门
- `Socket { timeout }` — 问 robotd
- `Command { program, args, timeout }` — 运行命令

## `Config::validate()`

加载时拒绝自相矛盾配置：
- 无组件
- install_dir 非绝对
- state_dir 在 install_dir 内（交换/回滚会销毁更新日志）
- 两组件共享 install_dir（互相 prune）
- keep_previous=0 且无 golden（无回滚目标）
- state_dir 非绝对

## 测试（钉死安全值）

- `example_config_parses` — 示例配置可解析，daemon 必须 restart robotd 且不重启 updaterd/btd，有真实 health gate
- `every_unit_on_apply_restarts_is_actually_shipped` — on_apply 重启的每个 unit 必须在工作区存在、有 service 文件、被两个构建 workflow 复制
- `shipped_config_is_safe_for_a_client_robot` — deploy/updater.toml 的安全值断言：dev_keys=false、fault_injection=false、check_interval 存在、按名授权、robot 组无变更权、btd 可中继、auto_apply≠All、稳定通道
- `installer_guard_and_shipped_config_agree_about_the_placeholder` — install.sh 的 ORG/ 占位符守卫与配置一致

## 关键摘要

config.rs 是机器人特定配置的唯一真相源：三前缀分隔发布流、AutoApply 有序策略防止无意义组合、validate 在加载时拒绝会导致数据丢失的配置；测试钉死 shipped 配置的安全值，防止"一个词"级别的回归。
