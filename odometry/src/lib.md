# lib.rs 文件解析

## 文件位置

`d:\microduck\odometry\src\lib.rs`

## 核心设计决策

这是一个纯结构体、无守护进程、无服务的里程计：输入就是控制循环已有的样本（关节角 + IMU 四元数），由 `robotd` 在其控制循环内直接 tick。

**核心算法：足底角点锚定（sole-corner anchoring）**

任何时刻，某个脚底的一个角点是机器人与地面的接触点。将该点锚定到世界（其 Z=0，X/Y 不变），用 IMU 方向定向躯干，则躯干的世界位置由正运动学推出。当另一个角点降到锚点下方——即落步——锚点移到那里，且保留其当前世界 X/Y，因此估计不会跳变。

**相对原型的两处改进**

1. 足链来自 `kinematics` crate 的 MJCF 模型，而非手抄的节段表——几何只有一个真相源。
2. 每只脚的链每 tick 只求一次，四个角点复用该结果；原型每个角点都重走一遍整条腿链（9 次链求值 → 现在 2 次）。

**世界坐标系**

航向就是 IMU 积分的 yaw——没有磁力计，因此世界坐标系是"开机时机器人朝哪"，相对运动的消费者已足够。

**alpha only**，与 `robotd` 一致：v1/v1.5 几何留在原型。

## 常量分析

| 常量 | 值 | 含义 |
|---|---|---|
| `SOLE_HALF_LEN` | 0.0270 | 足底沿 foot-site 帧 X（前后）的半长。占位值（v1.5 足底 bbox），待 alpha 的 `sole_left.stl` 实测 |
| `SOLE_HALF_WIDTH` | 0.0206 | 足底沿 Y（左右）的半宽。占位值，同上 |
| `SWITCH_MARGIN` | -0.010 | 候选角点必须低于世界 Z = `-SWITCH_MARGIN` 才能竞标锚点。锚点本身 Z=0，因此此裕度是给 FK 和 IMU 噪声的 slack，不是物理深度 |
| `SWITCH_CONFIRM_TICKS` | 2 | 候选必须保持为最低点的 tick 数。在 50 Hz 控制循环下为 40 ms——远在支撑相位内、远超一 tick 毛刺 |
| `LEFT` / `RIGHT` | 0 / 1 | 左右脚索引 |

## 结构体：Odometry

| 字段 | 含义 |
|---|---|
| `model: &'static Model` | MJCF 模型的静态引用 |
| `feet: [SiteId; 2]` | 左右脚的 site id |
| `joint_map: [Option<usize>; JOINT_NAMES.len()]` | `JOINT_NAMES` 位置 → model 关节索引；嘴为 `None`（它不驱动腿） |
| `angles: Vec<f64>` | 模型顺序的角度缓冲区，每 tick 复用 |
| `anchor_foot: usize` | 锚点在哪只脚 |
| `anchor_local: [f64; 3]` | 接触点在该脚 site 帧中的坐标 |
| `anchor_xy: [f64; 2]` | 接触点的世界 X/Y；Z 定义为 0 |
| `position: [f64; 3]` | 躯干世界位置 |
| `yaw: f64` | 躯干航向（IMU 积分 yaw） |
| `pending: Option<(usize,[f64;3],[f64;2])>` | 竞标中的候选：(脚, 局部点, 世界 XY) |
| `pending_ticks: u32` | 候选已持续的 tick 数 |
| `needs_init: bool` | 首次 update 用实际启动姿态播种 `anchor_xy`，使躯干从 (0,0) 开始而非被脚偏移 |

## 方法分析

### `new(model)` / `alpha()`

从 `Model` 查找 `left_foot`/`right_foot` site（不存在则 panic）；建立 `joint_map`（嘴 → None）；分配 `angles` 缓冲区。

### `update(joints, quat_wxyz)`

主循环单步：

1. 将 `JOINT_NAMES` 顺序的关节角通过 `joint_map` 散到模型顺序的 `angles`（嘴被跳过）。
2. 归一化 IMU 四元数得到 `rot`。
3. 求两只脚的 `site_pose`（各一次）。
4. 若 `needs_init`：用当前锚脚的世界 X/Y 播种，清零 `needs_init`。
5. `reproject` 计算躯干位置。
6. 找最低点 `lowest_corner`：
   - 若候选与当前 `pending` 同一只脚，`pending_ticks += 1`；否则重置为新候选，计数 = 1。
   - 计数达到 `SWITCH_CONFIRM_TICKS`：切换锚脚、锚局部点、锚世界 XY（保留该角点已有的世界 XY → 跨切换连续），重新 `reproject`，清零计数。
7. `yaw = rot.yaw()`。

### `reproject(rot, feet)`

接触点在躯干帧 = `feet[anchor_foot].transform_point(anchor_local)`；世界接触点 = `rot.rotate(...)`；躯干位置 = `anchor_xy - contact.xy`, `-contact.z`。

### `lowest_corner(rot, feet) -> Option<(foot, local, world_xy)>`

遍历 2 脚 × 4 角点，世界 Z 最低且低于 `-SWITCH_MARGIN` 者胜出。注意角点世界 X/Y 由 `self.position` 推出——用的是当前躯干估计。

### 访问器

- `position()` → 躯干世界位置（Z 是锚点定义的地面以上高度）
- `yaw()` → 躯干航向
- `anchor_xy()` → 锚点世界 XY（遥测用）
- `anchor_foot()` → 0=左, 1=右

## 单元测试

| 测试 | 断言意图 |
|---|---|
| `standing_still_stays_at_the_origin` | 静止站立应在原点不漂移；首次 update 播种锚点使躯干从 (0,0) 开始；躯干 z > 0.02；yaw=0 |
| `yaw_follows_the_imu` | yaw 完全跟随 IMU 四元数（0.3 rad） |
| `the_mouth_moves_no_odometry` | 嘴（index 9）动 42 rad 与不动结果一致——嘴不进入 FK |
| `the_anchor_switches_only_after_the_claim_holds` | 一 tick 的躯干横滚干扰不能偷走锚点（时间确认起作用）；持续 5 tick 才能切换 |
| `stance_leg_motion_translates_the_trunk_continuously` | 扫动支撑腿髋关节，躯干连续平移（每 tick 步长 < 0.02m），总位移 > 0.005m，全程有限值——无瞬移 |

## 关键摘要

- **无守护进程**：纯结构体，`robotd` 在控制循环内直接调用；输入即循环已有样本。
- **单真相源**：足链来自 `kinematics` 的 MJCF 模型，几何不再手抄。
- **连续估计**：锚点切换时保留角点的世界 XY，估计不跳变。
- **抗噪**：`SWITCH_CONFIRM_TICKS=2`（40ms）防止 FK/IMU 毛刺搬移锚点；`SWITCH_MARGIN` 给噪声裕度。
- **嘴隔离**：`joint_map` 中嘴为 None，不参与里程计。
- **相对世界**：航向 = IMU 积分 yaw，无磁力计，世界 = 开机朝向。
