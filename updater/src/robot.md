# robot.rs 文件解析

## 文件位置

`d:\microduck\updater\src\robot.rs`

## 核心设计

引擎对 robotd 的视图。用 trait 而非具体客户端：
1. 引擎在 robotd 存在之前构建
2. 降级路径（robotd 死/崩溃循环/挂起）必须可测试而无需真实崩溃

**每个方法都允许失败且必须超时有界。** 死的或沉默的 robotd 是正常预期答案——updaterd 是恢复路径，不能依赖它要恢复的东西。

## `SafeToRestart` 枚举

| 变体 | 含义 |
|---|---|
| `Yes` | 可重启 |
| `No(String)` | 正在移动/任务中，携带可显示原因 |
| `Incompatible(String)` | 有回复但无法解析——**不允许重启**（回复到达=控制循环在运行） |
| `Unreachable` | robotd 未应答——**视为安全**（控制循环不在运行=没东西在动，更新正是修复方式） |

`Incompatible` 与 `Unreachable` 的区分至关重要：曾因字段重命名把"不，我在任务中"读成"可以"而重启行走机器人。

## `Health` 枚举

| 变体 | 含义 |
|---|---|
| `Healthy` | 健康 |
| `Degraded(String)` | 有问题但属于板子（无舵机电源、无电机总线）——**通过门** |
| `Unhealthy(String)` | 报告问题 |
| `Incompatible(String)` | 有回复但无法解析——**失败门**（不可读裁决不是健康） |
| `Unreachable` | 超时未应答——失败门 |

`Degraded` 通过门的原因：降级是发布不可能造成的，回滚也无法修复。

## `RobotClient` trait

所有方法超时有界：
- `safe_to_restart(timeout)` — 不在运动中才重启
- `health(timeout)` — 健康门
- `model_api(timeout) -> Option<u32>` — 模型兼容性
- `remote_session_active(timeout) -> bool` — 远程会话中（礼貌检查，默认 false，绝不阻挡恢复更新）

## `SocketRobotClient`

通过 unix socket 与 robotd 通信。`ask()` 在 timeout 内完成一次请求/响应；连接拒绝、无回复、畸形回复都映射到 `None`（调用方转 Unreachable）。挂起对端（socket 开但不回复）必须与死端不可区分，否则引擎会挂在要修的机器人上。

## `AbsentRobot`

不返回任何东西的 robotd。不仅是测试替身——也是 `health=None` 组件的正确客户端，记录了预期降级行为。

## 测试

- `unreachable_robot_permits_restart` — 死 robotd 允许重启（恢复场景）
- `unreachable_robot_fails_health_gate` — 缺席从不被误判为成功
- `an_older_robotd_that_omits_a_health_field_is_still_healthy` — 真实 socket 上旧 robotd 回复仍可解析（此形状曾回滚好发布）
- `an_unreadable_answer_is_incompatible_not_unreachable` — 不可读回复不声称 absent
- `incompatible_still_fails_the_gate` — Incompatible 仍失败门

## 关键摘要

robot.rs 定义 robotd 客户端接口与四态安全/健康判定：核心是 `Incompatible`（有回复但不可解析）与 `Unreachable`（无回复）的严格区分——前者是活机器人，后者是可恢复的缺席；降级（板子问题）通过健康门，不可读回复失败健康门。
