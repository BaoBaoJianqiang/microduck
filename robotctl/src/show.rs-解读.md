# show.rs（更新记录时间线渲染器）解读与架构梳理

> 分析对象：`show.rs`（741 行），`robotctl update show`——一次更新运行的完整时间线。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `robotctl update show`——把一次更新运行完整地渲染成时间线。`update log` 告诉你"发生了一次更新、结果如何"；这个告诉你"它*做了什么*"：每个阶段耗时、验证过的 manifest、收集的 hook 输出、重启的 unit、gate 判定——然后拼接同一时间窗口的 journal，让记录包含更新重启的 daemon 们，不只是 updaterd 的一面之词。

**关键设计**：
- **全 UTC**：两个时钟在一个屏幕上 = 时间线不可读。`journalctl --utc` 让两半对齐。手写 civil 日期转换（不引入 chrono/tzdb——保持恢复路径依赖树小）。
- **verdict 先行**：读者来是为了看结果，时间线在下面解释它。
- **gap 时间列**：`+2m40s` 之类——"时间去哪了"。
- **gutter 列**：hook 输出和 artifact URL 放 gutter（│ 前缀），不挤进主列。
- **journal 拼接**：--since/--until 时间窗口，--unit=updaterd + 运行触及的所有 unit。跳过 transcript 中已有的 hook 行（去重）。
- **无结束运行**：明确说"被截断了，下次运行才有 verdict"。

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | render(RunTranscript) → 时间线文本 | L47-133 | confirmed |
| F2 | verdict 先行 | L63-68 | confirmed |
| F3 | 全 UTC，手写 civil 日期转换 | L8-11, L326-348 | confirmed |
| F4 | JOURNAL_TAIL=60s（结束后多读 60 秒） | L23 | confirmed |
| F5 | ALWAYS_SPLICED=updaterd | L27 | confirmed |
| F6 | ALREADY_IN_THE_TRANSCRIPT="updater::hooks:"（去重） | L39 | confirmed |
| F7 | gap 格式: +Ns/+NmNs/+NhNm | L351-357 | confirmed |
| F8 | hook 输出 gutter 列 | L194-197 | confirmed |
| F9 | manifest URL gutter 列 | L176-178 | confirmed |
| F10 | 13 个 Phase 名字（human-readable） | L248-264 | confirmed |
| F11 | 无结束 → 提示下次运行看 verdict | L95-105 | confirmed |
| F12 | Health passed ≠ healthy（分开表达） | L210-221 | confirmed |
| F13 | Unrecognised event 也渲染（新 release 的事件） | L230-233 | confirmed |
| F14 | log_line: 一行一次运行 | L386-409 | confirmed |
| F15 | civil() 验证：1970-01-01、2024-02-29、2000-03-01 | L422-430 | confirmed |
| F16 | units() 从 transcript 发现而非硬编码 | L280-290 | confirmed |

---

## 三、渲染流程

```
RunTranscript {
  run: 42,
  component: "daemon",
  events: [RunRecord { at, RunEvent }, ...],
  available: [40,41,42]
}
  ↓
Header:
  run N · component · YYYY-MM-DD HH:MM:SS UTC
  ↓
Verdict（先行）:
  "applied 0.1.3 → 0.1.4"
  ↓
Began（散文头，不重复成行）:
  asked for latest, from github.com/...
  requested by uid=1000...
  ↓
时间线（每个事件一行）:
  HH:MM:SS  +gap    kind         detail
  ├─ manifest: version · bytes · sha256 · signed · rev
  │  └─ gutter: URL
  ├─ hook: name, exit N
  │  └─ gutter: 输出每行
  ├─ unit: robotd: restart
  ├─ health: passed / FAILED — reason
  ├─ phase: preflight → downloading → ... → ROLLING BACK
  ├─ note: free text
  ├─ ended: summary
  ├─ truncated: N events dropped
  └─ ?: newer release event
  ↓
Footer:
  runs 40 to 42 are kept · `robotctl update log`
```

---

## 四、关键设计决策

### 4.1 为什么手写 civil 日期转换

注释说得很清楚："what must never be hand-rolled is a timezone database, and this does not touch one." 这是纯 UTC 算术（Howard Hinnant 的 civil_from_days）。给 robotctl 加 chrono 依赖 = 给恢复路径加 tz 数据库——为了 15 行有测试的算术。

### 4.2 为什么 verdict 先行

读者来是为了看结果。时间线在下面解释它。如果 verdict 在屏幕底部要滚一屏才看到——那不是设计。

### 4.3 为什么 gap 列

"时间去哪了"是时间线的核心问题。`+2m40s` 在下载阶段——那 2m40s 就是下载花的时间。`+0s` 列是噪声，所以只显示 ≥1s 的 gap。

### 4.4 为什么 gutter

hook 输出和 artifact URL 很长——挤进主列会把所有内容推出屏幕。gutter 列（│ 前缀）保持主行可读，细节在下面。

### 4.5 为什么 units() 从 transcript 发现

不是硬编码列表。一个发布新增了一个 daemon——当天就覆盖，不用改这里。transcript 里的 Unit 事件说了哪些 daemon 被重启。

---

## 五、结论

### confirmed
- C1：update show 时间线渲染，verdict 先行 + gap 列 + gutter。
- C2：全 UTC，手写 civil 转换（Hinnant 算法），不引入 chrono。
- C3：journal 拼接：60s tail，--unit 从 transcript 发现，hook 行去重。
- C4：13 个 Phase human-readable 名字。
- C5：无结束运行明确提示，Health passed ≠ healthy。
- C6：Unrecognised 事件也渲染（前向兼容）。

### inferred
- I1：RunTranscript/RunEvent 来自 duck-ipc-proto。
- I2：journalctl 由调用方执行（本模块只生成 argv）。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
