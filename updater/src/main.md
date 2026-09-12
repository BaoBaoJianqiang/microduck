# main.rs 文件解析

## 文件位置

`d:\microduck\updater\src\main.rs`

## 核心设计

`updaterd` 守护进程刻意**极薄**：解析参数→加载配置→从任何中断运行恢复→服务 socket。所有逻辑在库中，以便无 socket/无机器人也能测试。

**启动顺序至关重要。** `Engine::recover_on_start` 在 socket 服务**之前**运行，因此启动进坏版本的机器人在任何操作请求到达前已开始回滚。

## CLI 参数

| 参数 | 默认 | 说明 |
|---|---|---|
| `--config` | `/etc/robot/updater.toml` | 配置路径（全局） |
| `--socket` | `proto::DEFAULT_SOCKET` | 监听 socket |
| `--robot-socket` | 配置中 | robotd socket（覆盖配置） |
| `--check-only` | false | 执行启动恢复后退出（非只读探针，会推进启动计数） |
| `--self-test` | false | 加载配置+构造引擎后退出（不触状态；更新对新二进制的探针） |
| `--inject-fault` | 无 | 故障注入点（需配置允许，可重复） |
| 子命令 | `install` | 安装首个版本后退出 |

### `install` 子命令

机器人首次安装的引导路径，强制两项设置（仅因尚无版本在运行）：
- `on_apply → none`：unit 还不存在，`systemctl restart` 会失败
- `health → none`：`probe=socket` 会问 robotd，而它还未运行

**拒绝在已有版本运行时使用**（除非 `--force` 且 robotd 已停止），因为这些覆盖会静默禁用工作机器人的自动回滚。`--force` 用于已安装的 updaterd 太旧无法接受新发布的情况。

## 关键函数

### `load(args) -> Option<Loaded>`

加载配置与密钥环，两者必须**大声失败**：
- 空 trusted-keys 目录是致命错误（不是空 allow-list）
- 故障注入需配置允许
- 构造 `SocketRobotClient` 指向 robotd socket

### `install(...)`

引导安装：
1. `--from` 目录规范化（路径错误在此失败而非下游"找不到"）
2. 通过 store 检查是否已有 live 版本：无→允许；有+force+robotd 静默→允许；否则拒绝
3. 强制 `on_apply=none`、`health=none`
4. 设置 source（`--from` 或配置源）
5. 构造 Engine，调用 `apply(Target::Latest)`
6. 进度日志**按十分位**（decile）而非每 chunk——3.6 MB 下载曾产生 250 行日志挤掉关键日志

### `serve(args)`

1. `load` + 解析 allow_uids/gids（名称通过 `getpwnam`/`getgrnam` 解析为数字，因 sysusers 动态分配 uid）
2. `--self-test` 在恢复前退出（不触状态）
3. **`recover_on_start`** 在服务前执行
4. `reconcile_running_units` 检查延迟重启是否发生
5. `--check-only` 在此退出
6. 构造 `ipc::Server`，启动周期性检查（`check_interval`）
7. `tokio::select!`：serve socket / shutdown（SIGTERM/SIGINT）
8. 清理 socket 文件

## 测试要点

- `a_robot_answering_unreadably_still_counts_as_answering` — `Health::Incompatible`（不可读回复）仍算"有机器人在运行"，`--force` 必须拒绝（曾因字段重命名把活机器人读成 absent 而重启行走机器人）。
- `a_full_download_logs_eleven_lines_not_hundreds` — 进度按十分位节流（0-90 + 完成 = 11 行）。
- `completion_is_its_own_bucket` — 100% 独占桶，完成不显示停在 90%。

## 关键摘要

main.rs 是 updaterd 守护进程入口：极薄，所有逻辑在库；启动顺序确保恢复先于服务；`install` 是引导路径，强制无 health/无 on_apply 且拒绝覆盖 live 版本；名称解析与十分位进度节流是两个关键工程决策。
