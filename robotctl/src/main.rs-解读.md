# main.rs（robotctl CLI 入口）解读与架构梳理

> 分析对象：`main.rs`（约 4276 行），microduck 机器人的本地 CLI 工具 `robotctl` 的入口和核心。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `robotctl`——机器人的本地命令行客户端。它是一个**薄客户端**：解析命令行参数，通过 Unix socket 向 `updaterd` 发送 JSON-RPC 请求，打印流式通知和结果，将退出码映射为有意义的值。它本身不包含任何更新逻辑——那在 `updaterd` 守护进程内部。

**当前范围**：只实现了 `update` 命名空间。`robotctl` 这个名字保留给未来的通用机器人 CLI，命令从一开始就用命名空间组织。

**设计原则**（两类用户：现场恢复 + CI/bench）：
- **永不交互提示**：不问任何问题。
- **幂等**：重跑已有状态的命令就是成功。
- **退出码有意义**：CI 直接断言退出码，不解析文本。
- **通知走 stderr，结果走 stdout**：`--json` 可管道化，进度不干扰。
- **robotd 死了也能工作**：它跟 updaterd 通信，不跟 robotd。

**子模块**：`configure`（配置编辑器）、`duck`（3D 视图）、`monitor`（实时监控）、`path_map`（路径地图）、`show`（更新记录展示）。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 | main.rs（约 4276 行） |
| crate | robotctl |
| 角色 | 薄 CLI 客户端 over updaterd.sock |
| 通信 | UnixStream → /run/updaterd.sock |
| 协议 | JSON-RPC 2.0 |
| 当前命名空间 | 仅 update |
| 退出码 | 0=OK, 1=FAILED, 2=USAGE, 3=UNREACHABLE, 4=BUSY, 5=REFUSED |

---

## 三、架构

```
robotctl (你 / CI)
  │
  ├─ main.rs: clap 解析 argv
  │    ├─ update namespace (实现)
  │    │    ├─ update check   → JSON-RPC check
  │    │    ├─ update apply   → JSON-RPC apply (流式通知)
  │    │    ├─ update log      → JSON-RPC log
  │    │    └─ update show N  → show::render (本 crate)
  │    ├─ monitor              → monitor::run (ratatui TUI)
  │    └─ configure            → configure::run (ratatui TUI)
  │
  └─ UnixStream → /run/updaterd.sock → updaterd
```

**与 btd 的关系**：
```
phone ──▶ btd ──────┐
                    ├──▶ /run/updaterd.sock ──▶ updaterd
you / CI ─▶ robotctl┘
```

btd 通过 BLE 连接，robotctl 通过 Unix socket 连接——同一个 daemon，不同传输。

---

## 四、关键设计决策

### 4.1 为什么是薄客户端

更新逻辑在 updaterd 内部，robotctl 只负责：
1. 解析 argv
2. 发一个 JSON-RPC 请求
3. 打印流式通知和结果
4. 映射退出码

**好处**：现场恢复时 robotctl 必须能工作——如果 robotd 死了，robotctl 不依赖 robotd。更新逻辑在 updaterd 中，updaterd 在 recovery 模式下仍然运行。

### 4.2 退出码语义

| 码 | 含义 | CI 用途 |
|----|------|---------|
| 0 | 成功 | — |
| 1 | 失败 | 通用失败 |
| 2 | 用法错误 | clap 约定 |
| 3 | updaterd 不可达 | 区分"拒绝"和"没连上" |
| 4 | 另一个更新进行中 | 脚本重试而非报错 |
| 5 | 被拒绝 | 不兼容/preflight 失败 |

### 4.3 通知 stderr / 结果 stdout

- 进度通知 → stderr（人看）
- 最终结果 → stdout（管道）
- `--json` 时 stdout 是机器可解析的 JSON

### 4.4 子模块关系

| 模块 | 行数 | 功能 |
|------|------|------|
| configure | 1333 | robotd.toml TUI 编辑器 |
| duck | 952 | 终端 3D 机器人渲染器 |
| monitor | ~4220 | 实时控制循环监控 TUI |
| path_map | 354 | 盲文路径地图 |
| show | 741 | 更新记录时间线渲染 |

---

## 五、结论

### confirmed
- C1：robotctl 是 updaterd 的薄 CLI 客户端。
- C2：只实现 update 命名空间，其他保留。
- C3：Unix socket JSON-RPC，退出码语义化。
- C4：永不交互、幂等、通知 stderr/结果 stdout。
- C5：5 个子模块：configure/duck/monitor/path_map/show。

### inferred
- I1：monitor 模式订阅 robot.state 流（2Hz）。
- I2：configure 和 monitor 用 ratatui TUI。
- I3：update apply 是流式的（通知逐步打印）。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
