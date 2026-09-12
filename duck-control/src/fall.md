# fall.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\fall.rs`

## 核心设计决策

`fall.rs` 实现**跌倒预测**——检测跌倒*开始*，而非确认已完成的跌倒。

`Safety` 已有跌倒判定（投影重力超过 `fall_gravity_z` 约 60° 并保持 `fall_debounce` 200ms），但那在机器人已倒地或即将倒地时才触发。它适合"拒绝启用侧躺的机器人"，但对"落地前降低增益"毫无用处——瘫软的全部价值在于这个预测器要找到的时间窗口。

因此这是**第二个、刻意独立的检测器**，回答不同的问题：*机器人此刻是否正在倒下*——并且足够早可以采取行动，意味着它基于**速率**而非位置。

## 检测原理

躯干坐标系中投影重力随躯干旋转，其导数为 `ġ = −ω × g`（`ω` 为体坐标系角速度）。只需 z 分量（跌倒阈值读取的量）：

```
ġz = −(ωx·gy − ωy·gx)
```

不做滤波、不对四元数求导：陀螺是直接测量，在同一 12 字节块中到达；对 SFLP 四元数求导则会把滤波器自身的滞后加到这个以"早"为全部意义的数字上。

线性外推到 `FallPredictor` 的前瞻时间，测试为"四分之一秒后重力会在哪"——当*那个*越过不归点时触发。三个条件全部需要：
1. **已倾斜**超过 `tilt_z`（正立机器人上的陀螺尖峰是脚步、被推或被抱起，不是跌倒）
2. **仍在倾倒**（`ġz > 0`，向下而非从倾斜中恢复）
3. **预测超过 `predicted_z`** 在前瞻地平线

然后去抖动几个 tick（单个噪声样本不应让机器人在步态中途失去增益）。

## 调参即功能本身

太早：机器人会从本可走出来的倾斜中瘫软——每个误报都是机器人*造成*的跌倒，比被缓冲的跌倒更糟。太晚：僵硬着地，模式没买到任何东西。默认值刻意偏晚：`tilt_z = -0.90` 约 26°（正常行走达不到），60ms 去抖 = 50Hz 下 3 tick（长于任何脚步冲击，短到留给大部分跌倒可瘫软）。

## 类型分析

### `FallPredictorConfig`
- `tilt_z: f64` — 预测生效前机器人必须已越过的投影重力 z（正立 -1.0，-0.90 ≈ 26°）
- `predicted_z: f64` — 外推必须到达才算跌倒进行中（与 `SafetyConfig::fall_gravity_z` 同义同值）
- `lookahead: Duration` — 前瞻时长（默认 300ms）
- `debounce: Duration` — 判定需持续时长（默认 60ms）

### `FallPredictor`
- `falling_for: Duration` — 三条件同时成立的时长（任一 tick 不满足则重置）
- `fired: bool` — 本次跌倒是否已发过触发（边沿触发，调用方每 tick 轮询只得到一次边沿）

### 关键方法
- `gravity_z_rate(imu) -> f64` — 投影重力 z 的变化率（公开，便于调参时查看）
- `predicted_z(imu) -> f64` — `gravity.z + rate × lookahead`
- `observe(imu, dt) -> bool` — 一次 tick，去抖完成时返回 true（每次跌倒恰好一次）
- `reset()` — 清除累积判定（调用方已接管：瘫软中或策略停止驱动）

## 单元测试描述

- `the_rate_is_the_analytic_derivative`：30° 前倾、1 rad/s 俯仰时，重力 z 变化率应为 sin(30°)=0.5。
- `recovering_from_a_lean_never_fires`：从倾斜中恢复（负速率）永不触发。
- `a_footfall_on_an_upright_robot_never_fires`：正立机器人的脚步冲击（8° 倾斜、3 rad/s）不触发（tilt 门控拒绝）。
- `a_fall_fires_once_after_the_debounce`：真实跌倒在去抖完成（第 3 tick）触发一次，后续不重复。
- `a_static_tilt_is_not_a_fall`：静态倾斜（速率 0）不是跌倒。
- `a_slow_lean_waits`：缓慢倾斜（预测 z 未到 -0.5）不触发，等待站立策略有时间接住。
- `it_rearms_when_the_fall_stops`：跌倒停止后重新武装，第二次跌倒再次触发。
- `reset_drops_a_debounce_in_progress`：`reset` 清除进行中的去抖。

## 关键摘要

`fall.rs` 提供与 `Safety` 跌倒判定独立的**早期跌倒预测器**。基于陀螺直接测量的重力 z 变化率 `ġz = −(ωx·gy − ωy·gx)`，线性外推 300ms 后判断是否将越过不归点。三条件（已倾斜、仍在倒、预测过线）+ 60ms 去抖，边沿触发。调参刻意偏晚以避免误报造成跌倒。
