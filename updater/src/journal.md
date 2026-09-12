# journal.rs 文件解析

## 文件位置

`d:\microduck\updater\src\journal.rs`

## 定位

更新日志、单飞锁、启动计数器——引擎自身的持久状态。全部在 `state_dir`，**必须在每个组件 install_dir 之外**：symlink 交换或回滚绝不能销毁发生过什么的记录。

## `Journal` — 更新日志

追加式 JSONL 记录（`update-log.jsonl`），记录**拒绝与回滚的尝试**（不仅成功）——支持问的第一个问题。

- `append(entry)` — 写一行 + `sync_data` 持久（应对失败更新本身引发的断电）；撕裂尾行在读取时容忍（丢最后一条可接受，因它拒绝读日志不可接受）
- `trim()` — 超过 max_entries 时写-改名重写（崩溃不留下截断日志）
- `recent(limit)` — 最新条目
- `last_for(component)` — 某组件最后尝试
- `known_bad(component)` — 最近结果为回滚的版本（派生自日志，自愈；用于阻止回滚到已失败版本、阻止无人值守重试坏版本）

## `UpdateLock` — 单飞锁

`state_dir` 中的文件锁（flock）。`try_acquire` 失败返回 `None`→引擎返回 `Busy`。**非互斥锁机制**——注释说明此处不重复。

## `BootCounter` — 启动计数器

`recover_on_start` 使用：对每个 armed 的 trial 推进启动计数，耗尽则回滚。在交换**之前** armed，使交换与健康检查之间的崩溃可恢复。

## `Pins` — 钉版

`robotctl pin` 设置的版本，存储在 state_dir。

## 关键摘要

journal.rs 管理引擎持久状态：JSONL 更新日志（含拒绝/回滚，持久且容忍撕裂尾行）、flock 单飞锁、启动计数器（交换前 armed 保证崩溃可恢复）、钉版；全部在 state_dir 以在交换/回滚中存活。
