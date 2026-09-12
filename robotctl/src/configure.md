# configure.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\configure.rs`

## 定位

`robotctl configure` — 编辑 `robotd.toml` 而无需读 400 行注释参考。daemon 已知的每个键，功能开关在前，当前值对默认值，一行文档。

## 真相来源

**不定义任何键**。schema、默认值、校验、一行文档全来自 `robotd-params`（`robotd` 自身解析用的 crate）。新增 section 时编辑器编译期获知或构建失败——不可能错。

## 编辑如何应用

`toml_edit` 解析保留所有未触碰内容（注释、顺序、未知键）。只设置或移除改动的键。写之前通过 `Params::load`（daemon 自身门控，含范围检查）重新解析——本工具不可能写出 robotd 拒绝启动的文件。

## 关键类型

- `Row` — 一个键的状态：file set 值 / default / resolved（可选键实际解析到的值）
- `Edit::{Set, Clear}` — 待保存编辑
- `Model` — 可编辑状态：doc + pending + written（跨保存累积，供重启判断）

## 关键方法

- `rows()` — registry 顺序所有键，pending 编辑如已应用显示
- `edit(entry, input)` — 输入默认值或 `unset` 时 Clear（不钉默认值）；按 kind 校验
- `toggled(row)` — SPACE 的下一个值（Bool 翻转、TriBool auto→on→off、Choice 循环）
- `rendered()` — 应用所有 pending 的文档文本
- `save()` — 写 .toml.new → `Params::load` 校验 → rename 原子写入；记录 written

## 重启

daemon 启动时读一次配置，所以每次改动需重启。**重启哪个 daemon 从改动的键推导**：`[media]`/`[detect]` 是 `mediad` 读同一文件，其余是 `robotd`。`robotd` 先于 `mediad`（`mediad.service After=robotd.service`）。

## UI（ratatui）

单屏：功能开关分区在前，然后各 section；底部 footer 显示选中键的文档。SPACE 切换、ENTER 编辑、u 恢复默认、q 退出（有 pending 时确认保存）。

## 测试要点

- `comments_and_unknown_content_survive_an_edit` — 注释/顺序/未知键存活
- `a_config_robotd_would_reject_is_never_written` — hz=0 被 `Params::load` 拒绝，磁盘不动
- `the_restart_offer_names_the_daemon_that_reads_the_change` — media→mediad, control→robotd, 两者→[robotd, mediad]
- `an_unset_bitrate_shows_what_the_quality_resolves_to` — resolved hint 跟随 quality
- `the_shipped_example_sets_nothing_away_from_default` — 发货示例无偏离默认

## 关键摘要

configure.rs 是 robotd.toml 的无损编辑器：schema 来自 robotd-params（不可能错）；toml_edit 保留注释/未知键；写前用 Params::load 校验（范围检查）；原子 tmp+rename；重启 daemon 从改动键推导（media→mediad，其余→robotd，robotd 先）；输入默认值=清除而非钉住。
