# tof.rs 文件解析

## 文件位置

`d:\microduck\kinematics\src\tof.rs`

## 核心设计决策

ToF 重投影：8×8 距离网格变为 trunk 帧中的点。

`tofd` 发布传感器看到的东西——沿传感器自身帧中固定光束的斜距。消费者想要的是机器人可行动的几何：每个回波在 trunk 帧中的位置，并按原型运行时确定的方式过滤两个系统性干扰：

1. **地板回波。** 低头看的头在每个距离都看到地板；若光束的斜距乘以其向下分量到达传感器离地高度（乘安全系数以应对姿态误差），则击打的是地板而非障碍
2. **过近回波。** ~10cm 以下传感器读数不再可信——盖玻璃串扰与脉冲堆积产生假短回波——该波段读数作为噪声丢弃

传感器 pose 每帧来自 `HeadFk::tof_in_trunk`，躯干离地高度来自 MJCF 自身（`Model::trunk_height_m`）——无转录常量。

故意放在 `kinematics` 而非 `tof` crate：这是纯几何，而 `tof` crate 携带 vendored 的 ST C 驱动，`robotctl` 等客户端绝不该链接它。

## 常量

- `ROWS = 8`, `COLS = 8`, `N_ZONES = 64` — VL53L5CX/L8CX 8×8 模式网格
- `FOV_DEG = 45.0` — 传感器方形视场（度/轴），ST 数据手册对两代都为 45°×45°
- `FLOOR_SAFETY = 0.85` — 向下光束需覆盖传感器离地高度的比例才算地板命中。低于 1.0 使 FK 与姿态误差让*近*地板的回波读为地板而非障碍
- `MIN_RANGE_M = 0.10` — 水平距离小于此值丢弃（串扰假回波）

## `Zone` 枚举

一个 zone 的回波判定：
- `Empty` — 无可用回波
- `TooClose` — 短程噪声带内
- `Floor { point }` — 光束先到达地板
- `Hit { point, range }` — 有东西。`point` 在 trunk 帧；`range` 是距躯干原点垂直轴的水平距离，障碍规避与之比较

## `Posture` 结构体

躯干自身的状态——头关节无法知道的那一半几何。地板过滤器推理*世界*的向下；前倾的躯干把整个头链带走，忽略它会在机器人斜靠时把地板标为障碍。

- `gravity: [f64; 3]` — trunk 帧中的投影重力，直立约 `[0, 0, -1]`
- `trunk_height_m: Option<f64>` — 测量的躯干离地高度，`None` 回退模型站立静止高度

## `Reprojector` 结构体

- `fk: HeadFk`
- `beams: [[f64; 3]; 64]` — 传感器帧中单位光束方向（row-major，与线帧相同）
- `trunk_height_m: f64`

### 方法

- `new(model)` / `alpha()` — 构造光束：zone 中心均匀分布在 FOV 上，边缘半 zone 内嵌（8 zone 行有 8 个中心而非 9 个栅栏柱）。第 0 行是网格顶部，第 0 列是传感器左侧——与原型流式化器对同一 ULD 缓冲区的排序一致
- `beams()` — 暴露光束方向，供需要推理光束本身的几何（如 `hand` 的平面拟合）
- `sensor_in_trunk(head_joints)` — 传感器在 trunk 帧的 pose
- `project(ranges_m, head_joints, posture) -> [Zone; 64]` — 重投影一帧：
  - 计算 `level`（把测量重力转到正下的旋转，使过滤器 Z 轴是世界的）
  - 对每个有回波的 zone：旋转光束到 trunk 帧，再转水平帧；若 `r * downward >= floor_threshold` 则为 `Floor`；若水平距离 `< MIN_RANGE_M` 则为 `TooClose`；否则 `Hit`

### `level_from_gravity(gravity) -> Quat`

把 trunk 帧重力转到正下的旋转。重力太小（IMU 未收敛）返回 identity——不水平化好过随机水平化。处理正立（identity）、完全倒置（绕任意水平轴 π）与一般情况。

## 单元测试描述

- `a_forward_return_lands_a_metre_ahead` — 水平头光轴附近 1 米回波落在 trunk 前约 1 米，全管道约定的健全锚点
- `the_floor_is_floor_and_a_thing_before_it_is_not` — 低头：底行光束在地板距离是地板，中途被挡是障碍
- `an_upward_beam_is_never_the_floor` — 水平头顶行光束向上，无论多远都不是地板
- `the_noise_band_and_the_silence_are_named` — 过近是 `TooClose`，无回波是 `Empty`
- `a_leaning_trunk_makes_a_forward_beam_floor` — 躯干前倾（IMU 报告）使前向光束判为地板；同样回波在直立躯干是障碍
- `the_head_pose_steers_the_points` — 头向左转使重投影点向左（+y），FK 确实在循环中

## 关键摘要

`tof.rs` 把 VL53L5CX/L8CX 的 8×8 斜距重投影为 trunk 帧中的点，并过滤地板（用 IMU 重力水平化 + 安全系数 0.85）与过近噪声（<10cm）。传感器 pose 来自头链 FK，躯干高度来自 MJCF，无硬编码常量。放在 `kinematics` 是为了避免客户端链接 ST 驱动。
