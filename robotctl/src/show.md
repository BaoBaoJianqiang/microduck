# show.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\show.rs`

## 定位

`robotctl update show` — 一次更新运行的完整记录。`update log` 说发生了什么及结果；这说**做了什么**：每个 phase 耗时、验证的 manifest、钩子输出、重启的 unit、门裁决——然后拼上同期 journal，使叙述包含被重启的 daemon 而不仅 updaterd 一侧。

## 时间全用 UTC

两个时钟同一屏=时间线不可读，`journalctl --utc` 免费让两半一致。**手写 civil 日期算术**（Howard Hinnant 的 `civil_from_days`），不用日期 crate——避免把时区数据库放进刻意保持小依赖树的恢复路径 crate。

## 关键常量

- `JOURNAL_TAIL=60s` — journal 读到运行最后事件后 60s（延迟重启 5s 后 fired，daemon 启动身份在其后）
- `ALWAYS_SPLICED="updaterd"` — updaterd 总在拼接内
- `ALREADY_IN_THE_TRANSCRIPT="updater::hooks:"` — 钩子输出已在 transcript，去重

## 渲染结构

1. 头部：run · component · 起始 UTC 时间
2. **裁决在前**（读者来此目的），然后时间线
3. Began：asked for target, from source, requested by
4. 无 Ended 的 run 明确提示（updaterd 自重启或断电，裁决在后续 run）
5. 每个事件一行：时间 · 间隔(+Xs) · kind · detail
6. 多线事件（manifest URL、钩子输出）走 `│` 缩进 gutter，不挤主行

## 事件类型

`Began`(不重复成行) · `Phase` · `Manifest`(version/sha256/signed_by/rev + URL gutter) · `Hook`(exit code + 输出逐行) · `Unit` · `Health`(区分 passed 与 healthy——degraded 但通过门不渲染为 healthy) · `Note` · `Ended` · `Truncated` · `Unrecognised`(新版本事件，渲染而非跳过)

## journal 拼接

- `window()` — `(first, last+60)` unix 秒
- `units()` — updaterd + run 触碰的所有 unit（从 run 自身推导，非硬编码）
- `journal_command()` — `journalctl --utc --no-pager --since=@X --until=@Y --unit=...`
- 由 robotctl 本地运行（非 updaterd 服务——系统 journal 读取权限不应经 socket 借出）
- `worth_splicing()` — 去掉已在 transcript 的钩子行

## 测试

- `civil_dates_match_known_timestamps` — 钉死已知时间戳（含闰日、世纪边界）
- `a_degraded_commit_does_not_render_as_healthy` — passed≠healthy
- `a_run_with_no_ending_says_where_the_verdict_went`
- `hook_output_is_kept_line_by_line`
- `a_full_run_renders_as_a_timeline` — 整体形状断言

## 关键摘要

show.rs 渲染单次更新运行的完整 transcript：裁决在前；manifest/hook 多线内容走 gutter 不挤主行；passed≠healthy（degraded 通过门不渲染为 healthy）；无 Ended 的 run 明确提示；journal 拼接用 UTC，单位从 run 推导，去重钩子输出；手写 civil 日期避免时区依赖。
