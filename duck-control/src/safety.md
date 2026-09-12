# safety.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\safety.rs`

## 核心设计决策

`Safety` 是**安全权威**，持有**唯一的机器人写句柄**。其上方（策略、仲裁器、客户端）都不持有 `RobotIo`，因此都不能命令电机。不变量由借用检查器强制执行，而非靠"加第八个技能时需记住的规则"。

这与更新器恢复路径的论点相同：只在出错后才运行的代码最可能悄悄坏掉，所以应让坏掉的状态不可表示。

### 两条无条件规则
1. **非有限拒绝**：`NaN` 目标不钳制，直接拒绝。
2. **范围钳制**：目标保持在执行器行程内。

加上命令本身的**死开关**：意图停止到达时速度归零。**停止不是瘫软**——丢通信让机器人*原地站立*，因为站立是双足的安全态；失去平衡是另一回事，不是这一层要回答的。

### 跌倒判定是报告而非规则
`Safety::fallen` 每 tick 跟踪并发布，但不抢占任何东西：跌倒的机器人继续被驱动，人类保持掌控。跌倒后该做什么是控制决策，住在这层之上（`robotd` 的 limp-fall 模式基于 `FallPredictor` 降增益，通过 `apply` 命令，无豁免、无后门）。

## 常量与配置

- `ACTUATOR_MIN = -π`、`ACTUATOR_MAX = π` — XL330 位置范围（一整圈居中）。这是*执行器*行程，非逐关节解剖学极限（真实极限在 MJCF，未纳入此处），仅捕获 NaN、荒谬动作缩放、垃圾张量。

### `SafetyConfig`（默认值为原型数值）
- `fall_gravity_z = -0.5` — 超过此投影重力 z 算跌倒（正立约 -1.0，侧卧近 0）
- `fall_debounce = 200ms` — 需持续时长（硬脚步不是跌倒）
- `deadman = 500ms` — 意图老化阈值，超过则速度归零
- `gain_running = 200` — 运行增益
- `gain_limp = 50` — 瘫软增益（不由此处施加，由 `robotd` 在 limp-fall 命令）

### `Limit` 枚举
`Deadman`（意图老化）、`Range`（超出行程）、`NotFinite`（NaN/无穷）。

### `Applied`
一次 tick 的安全处理结果，含触发的 `Limit` 列表；`limited_by(limit)` 查询是否被某限制约束。

## `Safety<T: RobotIo>` 结构

- `io: T` — 唯一的 `RobotIo` 句柄
- `config: SafetyConfig`
- `falling_for: Duration` — 重力超阈值时长（正立样本重置）
- `fallen: bool`
- `gain: Option<u16>` — 最后写入的增益（避免每 tick 重复写，省 15 次总线写/帧）

### 关键方法

- `read()` / `slow_sensors()` / `imu_stale()` / `imu_ready()` — 透传（读操作不威胁写句柄垄断，但暴露句柄本身会）。
- `fallen() -> bool` — 查询跌倒状态。
- `set_torque(on)` — 上电关节。人类启用策略时调用一次，非每 tick；**启动时不调用**。不绕过任何东西（扭矩决定电机是否保持 `apply` 写入的内容，钳制、跌倒门控、瘫软增益仍作用于写入内容）。
- `gain() -> Option<u16>` — 最后写入的增益。
- `observe(sensors, dt)` — 每 tick（`apply` 前）更新跌倒状态：
  - **IMU 未收敛时不投票**：SFLP 滤波器需几秒样本其四元数才有意义，此前投影重力不是 `[0,0,-1]`，会被误读为"超跌倒阈值"→ 正立机器人在启动 200ms 内锁存 `fallen`→ `apply` 写 `gain_limp`→ 策略被拒"机器人倒下；请扶起"→ 几秒后自清除但增益留在 50 无法解释。保持上一判定是双向安全默认（启动时"未跌倒"，滤波器中途失通则保持跌倒）。
  - `down = gravity[2] > fall_gravity_z`；持续累加达 `fall_debounce` 则 `fallen = true`；正立样本重置。
- `gate(command, intent_age) -> (Command, Option<Limit>)` — 死开关：仅归零 twist（头部目标不动，过时头部姿态无害，过时速度会撞墙）。
- `apply(targets, hold, running_gain) -> Result<Applied>` — 通往电机的唯一路径：
  - 先 `set_gain(running_gain)`（调用方决定站立/行走/瘫软增益）。
  - 非有限目标：拒绝，写入 `hold`，返回 `NotFinite`（钳制 NaN 会得到边界值——看似合理的关节角，机器人会猛冲到极限而非静止）。
  - 范围钳制到 `[ACTUATOR_MIN, ACTUATOR_MAX]`，超界记录 `Range`。
  - 写入安全目标。
  - **无跌倒门控**：跌倒不阻止调用方驱动（判定是报告）。
- `set_gain(kp)` — 仅在增益变化时写总线。

## 单元测试描述

- `a_brief_tilt_is_not_a_fall`：100ms 倾斜（<200ms 去抖）不算跌倒，正立样本重置累加器。
- `a_sustained_tilt_is_a_fall`：持续 11×20ms 触发跌倒。
- `an_unconverged_imu_cannot_declare_a_fall`：IMU 未收敛时不判定跌倒（回归测试：曾让正立机器人启动 200ms 内锁存 fallen 并写 gain_limp）。
- `a_converged_imu_still_detects_a_fall`：收敛后同样样本必须判定跌倒。
- `by_default_a_fall_reports_but_does_not_preempt` / `a_fall_does_not_preempt_the_caller`：跌倒仅报告，策略继续驱动，增益不变。
- `a_caller_that_asks_for_the_limp_gain_gets_it`：调用方请求瘫软增益即直接生效。
- `a_non_finite_target_is_refused_not_clamped`：NaN 拒绝而非钳制，写入 hold。
- `out_of_range_targets_are_clamped_and_reported`：超界钳制并报告 `Range`。
- `an_ordinary_target_passes_through_unchanged`：正常目标直通。
- `the_deadman_zeroes_the_twist_only`：死开关仅归零 twist，头部不动。
- `the_gain_is_only_written_when_it_changes`：增益仅在变化时写总线（3 次 apply 只写 1 次增益）。

## 关键摘要

`Safety` 是唯一持有 `RobotIo` 写句柄的层，由借用检查器强制。两条硬规则（非有限拒绝、范围钳制）+ 死开关（意图老化归零 twist）。跌倒判定是报告不抢占；瘫软由上层通过 `apply` 请求实现。关键回归：IMU 未收敛时不参与跌倒判定。增益仅变化时写总线以节省带宽。
