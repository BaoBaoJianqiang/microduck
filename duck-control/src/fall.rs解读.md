# `fall.rs` 解读

## 概述

`fall.rs`（273 行）实现了一个**第二跌倒检测器**——与 [`crate::safety::Safety`](../safety/) 中已有的跌倒判定**故意分开**。`Safety` 的跌倒判定回答"是否已经倒了"（投影重力越过 `fall_gravity_z`，约 60° 倾斜，保持 200ms 后锁存），而本模块回答一个更早的问题：**"机器人此刻是否正在往下倒？"**

两者服务于不同目的：
- `Safety` 的跌倒判定用于"拒绝启用一个侧躺在地上的机器人"——到它锁存时机器人已经在地上了，这对该用途足够。
- 本预测器要在**落地之前**触发，让控制环路有时间降低电机增益（limp），从而以柔性而非刚性着陆。整个"变软"特性的价值就花在本预测器所要找到的这个时间窗口里。

核心设计意图：**在跌倒开始的瞬间看到它，而不是在跌倒完成后确认它**。因此它基于**角速度（rate）**判定，而非基于位置（position）。

## 关键结构体 / 函数

### `FallPredictorConfig`（第 55–68 行）

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct FallPredictorConfig {
    /// Projected-gravity z the robot must *already* be past before a prediction counts.
    /// Upright is -1.0; -0.90 is about 26° of tilt.
    /// —— 预测生效前机器人必须*已经*越过的投影重力 z 值。
    ///    直立为 -1.0；-0.90 约对应 26° 倾斜。
    pub tilt_z: f64,
    /// Where the extrapolation has to reach to count as a fall in progress.
    /// The same sense as `SafetyConfig::fall_gravity_z`, and by default the same number.
    /// —— 外推必须到达的位置，才算"跌倒进行中"。
    ///    语义与 `SafetyConfig::fall_gravity_z` 相同，默认值也相同。
    pub predicted_z: f64,
    /// How far ahead to extrapolate. —— 向前外推多远。
    pub lookahead: Duration,
    /// How long the verdict must hold before it fires. —— 判定须持续多久才触发。
    pub debounce: Duration,
}
```

四个阈值构成了检测器的全部行为，全部可配置。默认值实现于 `Default`（第 70–79 行）：

```rust
impl Default for FallPredictorConfig {
    fn default() -> Self {
        Self {
            tilt_z: -0.90,
            predicted_z: -0.5,
            lookahead: Duration::from_millis(300),
            debounce: Duration::from_millis(60),
        }
    }
}
```

### `FallPredictor`（第 86–94 行）

```rust
#[derive(Debug, Clone)]
pub struct FallPredictor {
    config: FallPredictorConfig,
    /// How long the three conditions have held together. Any tick that fails one resets it.
    /// —— 三个条件同时满足了多久。任何一个 tick 有一个条件不满足就归零。
    falling_for: Duration,
    /// Whether the trigger has already been handed out for this fall,
    /// so a caller polling every tick gets one edge rather than a stream of them.
    /// —— 本次跌倒是否已经发出过触发，
    ///    使得每 tick 轮询的调用方只得到一个边沿，而非一串信号。
    fired: bool,
}
```

—— 这是一个**边沿触发（edge-triggered）**的检测器：`observe()` 在去抖动完成的那个 tick 返回 `true`，之后保持 `false`，直到机器人看起来不再在跌倒为止。调用方自行决定触发后做什么、做多久——本模块只负责报告**那个瞬间**。

### `gravity_z_rate`（第 110–112 行）

```rust
pub fn gravity_z_rate(imu: &ImuData) -> f64 {
    -(imu.gyro[0] * imu.gravity[1] - imu.gyro[1] * imu.gravity[0])
}
```

—— 这是整个预测器的数学核心。躯干坐标系中，投影重力随躯干旋转，其导数恰好为 `ġ = −ω × g`（`ω` 为机身坐标系角速度）。只关心 z 分量（跌倒阈值读的就是它），只需一次乘减：

```text
ġz = −(ωx·gy − ωy·gx)
```

**设计理由与踩坑点**：
- **不做滤波**：陀螺仪是直接测量，与 IMU 数据块在同一个 12 字节包中到达。
- **不对四元数做微分**：对 SFLP 四元数做微分反而会把滤波器自身的延迟加到这个"以早为贵"的量上。
- **公开为 `pub`**：因为调阈值时需要对着录制数据直接观察这个数，且外部无法从其他途径恢复它。

### `predicted_z`（第 115–117 行）

```rust
pub fn predicted_z(&self, imu: &ImuData) -> f64 {
    imu.gravity[2] + Self::gravity_z_rate(imu) * self.config.lookahead.as_secs_f64()
}
```

—— 以当前速率线性外推 `lookahead` 时长后，重力 z 将到达哪里。测试就是"四分之一秒后重力在哪里"——当那个位置越过不归点时触发。

### `observe`（第 120–141 行）

```rust
pub fn observe(&mut self, imu: &ImuData, dt: Duration) -> bool {
    let rate = Self::gravity_z_rate(imu);
    let going_down = imu.gravity[2] > self.config.tilt_z
        && rate > 0.0
        && self.predicted_z(imu) > self.config.predicted_z;

    if !going_down {
        self.falling_for = Duration::ZERO;
        self.fired = false;  // 重新武装：机器人不再像在跌倒
        return false;
    }

    self.falling_for = self.falling_for.saturating_add(dt);
    if self.falling_for >= self.config.debounce && !self.fired {
        self.fired = true;
        return true;
    }
    false
}
```

三个条件**全部**必须满足：

1. **已经倾斜**越过 `tilt_z`。直立机器人上的陀螺尖峰是落脚、被推一把（它会吸收住）、或有人把它抱起来——这些都不是跌倒。没有位置门限的预测器会把三者全判为跌倒。
2. **仍在倾倒**（`ġz > 0`）——正在*往下*，而不是从倾斜中恢复。
3. **在 `lookahead` 视野内预测越过 `predicted_z`**。

然后去抖动几个 tick——因为一个噪声采样不能让机器人在迈步中途丢掉增益。

**重新武装逻辑**：只有当机器人不再看起来在跌倒时，`fired` 才重置为 `false`。这样正在 limping 的调用方在第一次跌倒的下落途中不会收到第二次触发。

### `reset`（第 145–148 行）

```rust
pub fn reset(&mut self) {
    self.falling_for = Duration::ZERO;
    self.fired = false;
}
```

—— 丢弃累积的判定——调用方已经接管了（正在 limp，或策略已停止驱动），下一次跌倒必须从头检测。

## 重要常量

| 常量 | 默认值 | 含义 |
|------|--------|------|
| `tilt_z` | `-0.90` | 已倾斜门限，约 26°，正常行走不会达到 |
| `predicted_z` | `-0.5` | 预测落点门限，语义同 `SafetyConfig::fall_gravity_z` |
| `lookahead` | `300 ms` | 向前外推视野 |
| `debounce` | `60 ms` | 去抖时长，50 Hz 下三个 tick |

**调参是整个特性**：
- **太早** → 机器人从本可自行走回的倾斜中 limp 出来——每个假阳性都是机器人*自己造成*的跌倒，比被阻尼的跌倒更糟。
- **太晚** → 刚性着陆，模式毫无价值。

默认值**故意偏晚**：`tilt_z = -0.90`（约 26°）是正常行走不会达到的；60ms 去抖（50Hz 下三个 tick）比任何落脚脉冲长，又短到能让大部分跌倒过程完成 limp。

## 测试要点（第 151–272 行）

测试辅助函数 `tipping(tilt, pitch_rate)` 构造一个躯干前倾 `tilt` 弧度、绕 y 轴俯仰速率 `pitch_rate` 的 IMU 数据。

### `the_rate_is_the_analytic_derivative`

验证导数就是解析值而非近似。从 30° 以 1 rad/s 前倾，重力 z 应以 `sin(30°) = 0.5` / 秒的速率上升——容差 `1e-12`。

### `recovering_from_a_lean_never_fires`

从倾斜中恢复（`rate < 0`）绝不能触发——无论速率多大。已经越过倾斜门限但往回转，不应被误判为跌倒。

### `a_footfall_on_an_upright_robot_never_fires`

直立机器人上的硬性落脚：大陀螺瞬态但背后没有倾斜。这是**最重要的假阳性**——每一步都会发生。倾斜门限就是用来拒绝它的。

### `a_fall_fires_once_after_the_debounce`

真正的跌倒：越过门限且倾倒速率使它落地。必须触发，且**只触发一次**。去抖 60ms = 三个 tick，前两个必须安静，第三个触发，之后 20 个 tick 都不再触发。

### `a_static_tilt_is_not_a_fall`

被手保持在某角度的静止倾斜——无论倾斜多远都不是跌倒。**位置 alone 不能触发**；那个判定属于 `Safety`，用它自己的阈值。

### `a_slow_lean_waits`

缓慢倾倒——已倾斜、在倾倒，但慢到站立策略还有四分之一秒来纠正。等待是正确答案：在这里 limp 反而会*造成*它正在预测的跌倒。

### `it_rearms_when_the_fall_stops`

触发后只有机器人恢复直立才重新武装，使第二次跌倒能被捕捉，但第一次的尾部不会被重复报告。

### `reset_drops_a_debounce_in_progress`

`reset` 丢弃进行中的去抖——调用方接管后，下一次跌倒从头计数。

## 与其他模块的关系

- **依赖 `imu::ImuData`**：所有输入来自 IMU 数据结构（陀螺 + 投影重力）。
- **与 `safety::Safety` 并列**：两者都是跌倒检测器，但回答不同问题、使用不同阈值。`Safety` 做位置判定（已经倒了），本模块做速率判定（正在倒）。
- **被 `policy` 或 `robotd` 调用**：控制循环每个 tick 调用 `observe()`，返回 `true` 时调用 `RobotIo::set_gain()` 降低增益。
- **不依赖 `model` 或 `obs`**：纯粹基于 IMU 数据，不关心关节状态。
#（注：内容由AI生成）
