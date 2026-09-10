# `safety.rs` 解读

## 概述

`safety.rs` 是 `duck-control` crate 中的 **安全权威层**（约 581 行），也是整个 crate 里**唯一能写电机的地方**。

它的核心设计意图用一句话概括：**把"除了这里没人能动电机"这条不变量，交给借用检查器（borrow checker）强制，而不是靠人在加第八个技能的演示前夜还记着。** 上层——策略、仲裁器、任何客户端——都不持有 `RobotIo`，所以它们**在类型上就不可能**指挥电机。这与 updater 对其恢复路径的论证是同一个思路："只在已经出事之后才运行的代码，最容易悄悄坏掉"，那就让坏状态根本不可表达。

模块文档列出了它的规则体系：

- **两条无条件规则**：
  1. **非有限拒绝（non-finite rejection）**：`NaN` 目标不被钳制，而是被**直接拒绝**。
  2. **范围钳制（range clamp）**：目标被限制在执行器行程内。
- **一个命令上的死区开关（deadman）**：意图（intents）停止到达时，速度归零。**停住不是发软（limp）**——失联让机器人站定，因为站立是双足机器人的安全状态；失去平衡是另一回事，不归这层管。
- **跌倒判定是报告，不是规则**：`Safety::fallen` 每个 tick 跟踪并对外发布，但**不抢占任何东西**——跌倒的机器人继续被驱动，人类继续掌权。

## 关键常量

```rust
/// The XL330's position range: one turn, centred, from the count↔radian conversion.
/// XL330 的位置量程：一整圈、居中，来自 计数值↔弧度 换算。
pub const ACTUATOR_MIN: f64 = -std::f64::consts::PI;
pub const ACTUATOR_MAX: f64 = std::f64::consts::PI;
```

- **`ACTUATOR_MIN = -π`、`ACTUATOR_MAX = π`**：这是**执行器的**行程，不是逐关节的解剖学限位。alpha 机器人真正的关节限位在 MJCF 文件里，而 MJCF 没有 vendored 进本 crate。所以这个钳制能拦下策略吐出的 `NaN`、离谱的动作缩放或垃圾张量，但**拦不住**把关节驱动到机械上不明智的位置。注释特意把这一点写明、不粉饰——一个看起来像逐关节、其实不是的限位，会让人误以为有根本不存在的保护。

## 关键结构体与函数

### `SafetyConfig`：安全参数配置

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct SafetyConfig {
    pub fall_gravity_z: f64,      // 投影重力 z 超过此值即算跌倒。直立约 -1.0，侧翻接近 0。
    pub fall_debounce: Duration,  // 该状态需持续多久才计数，去抖，避免重踏步被误判为跌倒。
    pub deadman: Duration,         // 意图年龄超过此值则速度命令归零。
    pub gain_running: u16,         // 运行时增益。
    pub gain_limp: u16,            // 向地板"屈服"而非对抗的增益。此处不应用它——由 robotd 在 limp-fall 期间命令——但它是个安全数，故与其他安全数放一起。
}
```

`Default` 实现沿用原型数值：

```rust
fall_gravity_z: -0.5,
fall_debounce: Duration::from_millis(200),
deadman: Duration::from_millis(500),
gain_running: 200,
gain_limp: 50,
```

### `Limit` 与 `Applied`：本次 tick 改了什么

```rust
pub enum Limit {
    Deadman,     // 意图过期，速度被归零。
    Range,       // 目标超出执行器行程。
    NotFinite,   // 目标是 NaN 或无穷。
}

pub struct Applied { pub limits: Vec<Limit> }
impl Applied { pub fn limited_by(&self, limit: Limit) -> bool { ... } }
```

—— 被拦下的原因要**暴露给客户端**，而不是让客户端看着机器人不理它却不知为何。

### `Safety<T: RobotIo>`：安全权威本体

```rust
pub struct Safety<T: RobotIo> {
    io: T,
    config: SafetyConfig,
    falling_for: Duration,   // 重力已超阈值多久，被任一直立样本清零。
    fallen: bool,
    gain: Option<u16>,       // 记录上次写入的增益，未变则不重写（否则每 tick 15 次总线写）。
}
```

泛型 `T: RobotIo` 让生产实现是真实总线、测试实现是 `FakeIo`，而"唯一写句柄"这一点靠 `io` 是私有字段来保证。

### 读与透传：`read` / `slow_sensors` / `imu_stale` / `imu_ready`

```rust
pub fn read(&mut self) -> Result<Sensors, IoError> { self.io.read() }
pub fn slow_sensors(&mut self) -> Result<SlowSensors, IoError> { self.io.slow_sensors() }
pub fn imu_stale(&self) -> ImuStale { self.io.imu_stale() }
pub fn imu_ready(&self) -> bool { self.io.imu_ready() }
pub fn fallen(&self) -> bool { self.fallen }
```

慢传感器和 IMU 诊断是**只读**的，对"唯一写句柄"无威胁，但把句柄发出去让它们自己去读**就**有威胁了——所以这些都做成透传方法，而不是把 `&RobotIo` 交出去。

### `set_torque`：上电，且只在人类启用策略时调用一次

```rust
pub fn set_torque(&mut self, on: bool) -> Result<(), IoError> {
    tracing::warn!(on, "torque");
    self.io.set_torque(on)
}
```

注释把"它是什么/不是什么"讲得很直白：

- 它在**人类对一只 limp 机器人启用策略时调用一次**，不是每 tick 调。
- **启动时不调用**。被更新重启的 `robotd` 必须让站着的机器人继续站着。
- 它**不绕过任何东西**。扭矩只决定电机是否保持 `apply` 写下的位置；所有钳制、跌倒门、limp 增益仍然作用于"写什么"。一只上了扭矩的跌倒机器人，仍然被以 `gain_limp` 命令 `hold`。

### `observe`：更新跌倒状态（每 tick 调，先于 `apply`）

```rust
pub fn observe(&mut self, sensors: &Sensors, dt: Duration) {
    if !self.io.imu_ready() { return; }   // 未收敛的姿态滤波器没有投票权。
    let down = sensors.imu.gravity[2] > self.config.fall_gravity_z;
    if down {
        self.falling_for = self.falling_for.saturating_add(dt);
        if self.falling_for >= self.config.fall_debounce { self.fallen = true; }
    } else {
        self.falling_for = Duration::ZERO;
        self.fallen = false;
    }
}
```

**两个去抖方向**：倒下要持续 `fall_debounce` 才计数；任一**直立样本**立刻清零累加器。不去抖的话，一次扎实的落脚冲击就会被读成跌倒。

**IMU 未就绪时直接 return（保持前一判定）是关键踩坑点**。注释记录了真实事故：SFLP 滤波器需要几秒采样后四元数才有意义，此前投影重力不是 `[0,0,-1]`，而是滤波器"半路上"的值——读起来"高于跌倒阈值"，即"侧翻"。持续 200ms 后，台上直立机器人在启动时就锁存 `fallen`：`apply` 写 `gain_limp`、策略以"机器人倒下了，先把它扶起来"为由被拒，几秒后自己又清掉判定，却把增益留在 50、无人能解释。板子上观测到：手动设成 137 的关节，在 `robotd` 跑了五秒后读回 50。保持前一判定是双向的安全默认——启动时是"没跌倒"（台上站着的机器人确实没跌倒），运行中滤波器中途失效则让已跌倒的机器人保持跌倒。

### `gate`：死区开关，只归零 twist

```rust
pub fn gate(&self, command: Command, intent_age: Duration) -> (Command, Option<Limit>) {
    if intent_age <= self.config.deadman { return (command, None); }
    let mut stopped = command;
    stopped.twist = [0.0; 3];
    (stopped, Some(Limit::Deadman))
}
```

—— 只把**速度（twist）**归零。头部目标**故意保留**：过时的头姿无害，而过时的速度会把机器人走进墙里。

### `apply`：唯一到电机的路径

```rust
pub fn apply(&mut self, targets: [f64; NUM_JOINTS], hold: [f64; NUM_JOINTS],
             running_gain: u16) -> Result<Applied, IoError>
{
    let mut applied = Applied::default();
    self.set_gain(running_gain)?;

    // 注意这里**没有**跌倒门。倒下不阻止调用方驱动——判定只是报告（见模块文档）。
    // 跌倒该怎么办由上层决定，以普通目标 + 普通增益的形式到达。
    if targets.iter().any(|v| !v.is_finite()) {
        applied.limits.push(Limit::NotFinite);
        self.io.write(&JointTargets::new(hold))?;
        return Ok(applied);          // 拒绝：写 hold 位姿，不动
    }

    let mut safe = targets;
    for value in safe.iter_mut() {
        let clamped = value.clamp(ACTUATOR_MIN, ACTUATOR_MAX);
        if clamped != *value {
            if !applied.limited_by(Limit::Range) { applied.limits.push(Limit::Range); }
            *value = clamped;
        }
    }
    self.io.write(&JointTargets::new(safe))?;
    Ok(applied)
}
```

要点：

- `hold` 是"策略不该驱动时"要命令的位置——通常是机器人当前已在的姿态；`running_gain` 是调用方想要的增益（站立策略跑软一点、走路硬一点、limp-fall 更软）。这些都是**控制决策，不是安全决策**，由调用方传入，本层不替它做判断。
- **非有限是拒绝，不是钳制**。把 `NaN` 默默钳到边界，会得到一个"看起来合理"的关节角——比拒绝移动糟得多：机器人会猛地扑到限位，而不是原地保持。
- 钳制时只要本轮任一关节越界就记一次 `Range`（去重），并实际改写值。

### `set_gain`：只在变化时写总线

```rust
fn set_gain(&mut self, kp: u16) -> Result<(), IoError> {
    if self.gain == Some(kp) { return Ok(()); }
    self.io.set_gain(kp)?;
    self.gain = Some(kp);
    Ok(())
}
```

—— 增益未变就不重写。朴素版本每 tick 写一次增益，15 个关节在 50Hz 下等于每秒 750 次无谓总线写，还挤着控制循环要用的总线。

`gain()` 返回"当前实际在跑的增益"，注意这**不总是调用方要求的值**。

### 测试专用的 `io()` 访问器

```rust
#[cfg(test)]
fn io(&self) -> &T { &self.io }
```

—— 只在测试里存在、故意不公开。生产环境把它发出去就等于拆掉了"安全层独占写者"这整个设计的意义。

## 测试要点

测试用 `FakeIo::at(DEFAULT_POSITION)` 构造安全层，围绕几条契约逐条钉死：

1. **`a_brief_tilt_is_not_a_fall`**：100ms 侧倾 < 200ms 去抖 → 不算跌倒；且一个直立样本必须把累加器清零（再次侧倾也不累加）。硬落脚把重力短暂打尖，若当跌倒处理就会在步中把机器人放倒——而那本身就是造成跌倒的方式。
2. **`a_sustained_tilt_is_a_fall`**：持续倾斜（11×20ms）即判跌倒。
3. **`an_unconverged_imu_cannot_declare_a_fall`**（回归测试）：`imu_ready=false`、重力 `[0,0,0]`（与"侧翻"完全相同的读数）连跑 50 个 tick，也**不得**判跌倒。这就是板子上花了一下午定位的那个事故的回归。
4. **`a_converged_imu_still_detects_a_fall`**：`imu_ready=true` 时同样的读数必须判跌倒——否则上面那个守卫就等于把跌倒检测整个关掉了。
5. **`by_default_a_fall_reports_but_does_not_preempt` / `a_fall_does_not_preempt_the_caller`**：判了跌倒后照常 `apply`，limits 为空、写入的仍是调用方要的目标、增益仍是调用方要的 `gain_running`。这是让 limp-fall 能完全活在本层之上的契约——它靠**主动请求**（走同一个 `apply`、同一条路径、无豁免）来降增益。这里若放一个跌倒门，摆姿 ramp 想动一只躺在地上的机器人就得绕过它，而"有旁路的安全规则就不是规则"。
6. **`a_caller_that_asks_for_the_limp_gain_gets_it`**：调用方要 `gain_limp` 就直通，且 `gain()` 如实上报当前在跑 limp。
7. **`a_non_finite_target_is_refused_not_clamped`**：把 `[3]` 设成 `NaN` → 记 `NotFinite`、**不**记 `Range`、实际写入的是 `DEFAULT_POSITION`（hold）。
8. **`out_of_range_targets_are_clamped_and_reported`**：`100.0` 钳到 `ACTUATOR_MAX`、`-100.0` 钳到 `ACTUATOR_MIN`，并记 `Range`。上报很重要——被悄悄改动的命令，客户端无从知道机器人为何不照做。
9. **`an_ordinary_target_passes_through_unchanged`**：正常 tick 必须原样穿过，否则钳制器就在悄悄篡改正常操作、其他测试全都失去意义。
10. **`the_deadman_zeroes_the_twist_only`**：100ms 新鲜意图直通；5 秒过期只把 `twist` 归零、`head` 原样保留，并返回 `Deadman`。
11. **`the_gain_is_only_written_when_it_changes`**：三次 `apply` 三次位置写，但增益只在第一次写——钉住"每 tick 不重写增益"的优化。

## 与其他模块的关系

- **依赖 `crate::io`**：持有唯一的 `RobotIo` 写句柄，消费 `IoError`、`JointTargets`、`Sensors`，并把只读的 `SlowSensors`、`ImuStale` 透传出去。`FakeIo` 来自同模块，专供测试。
- **依赖 `crate::model`**：`NUM_JOINTS` 决定目标数组长度；测试里用 `DEFAULT_POSITION`。
- **依赖 `crate::obs`**：`Command`（twist + head）是 `gate` 的输入。
- **对上层（`robotd`，crate 外）**：`robotd` 通过 `observe`→`gate`→`apply` 三步驱动每 tick；limp-fall 模式靠自己读 `crate::fall::FallPredictor`，然后**以普通目标和普通增益**调用 `apply`，在本层没有任何后门或豁免。跌倒预测器（`crate::fall`）是另一个、更上层的事件源，与本层的 `fallen` 判定互不替代。
- **不做的事**：不决定"跌倒后怎么办"（那是控制决策，在上面）、不设逐关节解剖学限位、不在启动时上扭矩、不抢占策略。
#（注：内容由AI生成）
