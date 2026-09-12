# faults.rs 文件解析

## 文件位置

`d:\microduck\updater\src\faults.rs`

## 定位

故意故障注入。回滚是最可能被悄悄破坏的功能（仅在其他出错时运行）。使故障可注入把"回滚大概能工作"变成 CI 断言。

**无条件编译入**（非 `#[cfg(test)]`），使发布的同一二进制可在 bench 机器人上演练。启用任何故障需显式 flag，生产配置从不设置。

## `Faults` 结构体

| 字段 | 效果 |
|---|---|
| `corrupt_artifact` | 下载后破坏工件（哈希不匹配，不安装） |
| `fail_post_hook` | post 钩子失败（回滚） |
| `fail_health` | apply 后报告不健康（回滚） |
| `hang_health` | 健康探针挂起（证明超时工作） |
| `abort_after_swap` | symlink 交换后立即 abort（证明 kill -9 留一致状态） |
| `simulate_disk_full` | staging 时假装磁盘满 |
| `fail_rollback` | 回滚本身失败（最糟路径） |
| `fail_rollback_apply` | 回滚时 apply action 失败（机器人回好版本但一 unit 未重启） |

## `Faults::from_names(names, allowed)`

- 空列表→无故障
- `allowed=false` 且有列表→错误（"必须永不客户端机器人启用"）
- 未知名称→错误（静默忽略 typo 会使测试看似通过但故障未注入）

## 测试

- `no_faults_needs_no_permission`
- `injection_is_refused_without_config_permission` — 编译入发布二进制可接受的门
- `unknown_fault_is_rejected` — typo 大声失败

## 关键摘要

faults.rs 提供编译入发布二进制的故障注入点（8 个），使回滚等仅在出错时运行的路径可被 CI 断言；启用需配置 `allow_fault_injection`，未知名称拒绝以防 typo 静默通过。
