# main.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\main.rs`

## 定位

`robotctl` 主入口。`updaterd` unix socket 上的**瘦客户端**：解析 argv、发一个 JSON-RPC 请求、打印流式通知与结果、映射退出码。不含任何更新逻辑——那在 `updaterd` 引擎内。

```
phone ──▶ btd ──────┐
                    ├──▶ /run/updaterd.sock ──▶ updaterd
you / CI ─▶ robotctl┘
```

## 设计规则（面向两类用户）

1. **现场恢复**（app/BLE 不可用时）
2. **CI/bench 测试**（每个操作必须可脚本化）

因此：
- **永不提示**（no prompts）
- **幂等**：重跑已成立的命令=成功
- **退出码有意义**：CI 断言退出码而非解析文本
- **通知到 stderr，结果到 stdout**：`--json` 可管道化
- **robotd 死时仍工作**：它跟 updaterd 对话，不跟 robotd

## 退出码（稳定，CI 断言）

| 码 | 名 | 含义 |
|---|---|---|
| 0 | OK | 成功 |
| 1 | FAILED | 通用失败 |
| 2 | USAGE | 用法错误（与 clap 一致） |
| 3 | UNREACHABLE | updaterd 不可达（区别于拒绝） |
| 4 | BUSY | 另一更新在进行（脚本应重试） |
| 5 | REFUSED | 拒绝（不兼容/预检失败，"正确拒绝"） |
| 6 | DENIED | 无变更权限（修法是 sudo） |

## 命名空间

`Net`(wifi, configd) · `System`(名字/身份/电源, configd) · `Robot`(关节电源/技能/凝视, robotd) · `Quack`(chirp) · `Chorale`(合唱) · `Theremin`(ToF 特雷门琴) · `Configure`(robotd.toml 编辑器) · `Pad`(手柄配对) · `Update`(更新管理) · `Monitor`(控制循环实况) · `Health`(软硬一体健康) · `Version`(运行+安装版本) · `Completions`(shell 补全)

## `Client` — 阻塞 JSON-RPC

故意用 `std::os::unix::net`（非 tokio）：短命 CLI 发一个请求。
- `connect_to(service, path)` — 连接失败按服务名给不同建议（`unreachable_hint` 区分 EACCES=组权限/ENOENT=无守护/ConnectionRefused=死进程）
- `call()` — 发请求，过滤进度通知到 stderr，匹配 id 取响应
- `hello()` — 存活检查（非门控，曾是门控导致无法修复 skew）

## 关键函数

- `unreachable_hint()` — EACCES 说"组权限"而非"服务挂了"（新装机用户常见）
- `Failure::from_rpc()` — 映射 daemon 错误码到退出码（BUSY→重试，INCOMPATIBLE/PREFLIGHT→REFUSED）
- `run_health()` — 同时问 robotd 硬件 + updaterd 软件，unhealthy→REFUSED，不可达→UNREACHABLE
- `run_version()` — 问三个 daemon 运行版本 + installed 组件 + systemd units
- `is_behind()` — **用 revision 判断运行/安装差异**（dev 通道版本号相同但 SHA 不同），版本号仅作 fallback
- `run_pad()` + `BtdPaused` — 配对时临时停 btd 并重启蓝牙适配器（特定板子 workaround，在 CLI 而非 daemon）
- `report_progress()` — 进度到 stderr，tty 单行覆写，重定向时按十分位节流
- `restore_sigpipe()` — 修复 `robotctl | head` 时 println panic
- `apply_target()` — `--version`/`--ref`/`--staging`/`--from` 组合映射到 Target
- `resolve_from_dir()` — 客户端侧把 `--from` 绝对化（updaterd 工作目录是 `/`）

## 关键摘要

main.rs 是 robotctl 瘦客户端核心：7 个有意义的退出码支撑 CI；阻塞 unix socket 客户端；`unreachable_hint` 区分权限/缺席/死进程；health 一次答硬件+软件；is_behind 用 revision 而非版本号判断 dev 通道运行/安装差异；配对时 BtdPaused 临时停 btd；进度到 stderr，结果到 stdout。
