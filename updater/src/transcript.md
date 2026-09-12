# transcript.rs 文件解析

## 文件位置

`d:\microduck\updater\src\transcript.rs`

## 定位

每次运行的更新 transcript——updaterd 实际做了什么，保存在能存活的地方。

- 更新日志（journal）记录尝试发生了及如何结束
- transcript 记录**做了什么**：每个 phase 边界、验证的 manifest、钩子输出、重启的 unit、门的裁决

每个运行一个文件，在 `state_dir/runs/` 下，与日志同规则——在每个 install_dir 之外，使交换/回滚不销毁交换/回滚的记录。

## 为何不用 journal

`/var/log` 是 zram 设备，`Storage=persistent` 只买干净重启存活而非断电存活，而需要 transcript 的更新恰恰是断电结束的那些。

## 关键常量

- `MAX_EVENTS = 4000` — 单运行最多事件（防重试循环钩子）
- `MAX_BYTES = 2 MB` — 单文件最大字节
- `MAX_TEXT = 64 KB` — 单事件最大文本（钩子输出截断）

## `Transcript`

- `begin(state_dir, max_runs)` — 取下一个 id（磁盘最高+1，在更新锁下无竞态），修剪旧 transcript，创建空文件
- `record(event)` — 追加，永不失败（写不进去是 journal 警告）。ending 事件总是记录（超限时其余丢弃）
- id 短、可排序、可输入（`robotctl update show 42`）

## 记录永不失败更新

每次写入都是 best-effort，失败报 journal——完成但丢失日记的更新严格优于为保日记诚实而放弃的更新。

## 关键摘要

transcript.rs 记录每次更新的详细操作（phase/manifest/钩子/重启/门裁决）到 state_dir 下的独立文件，在断电中存活；有事件/字节/文本上限防爆炸；写入永不失败更新（best-effort）；id 简短可输入。
