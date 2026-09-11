# tof.rs 源码解读与架构梳理

> 分析对象：`tof.rs`（387 行），kinematics crate 的 ToF 重投影模块——8×8 距离网格变为躯干帧中的点，带地板过滤和太近过滤。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：`tof.rs` 提供 `Reprojector`——将 VL53L5CX/L8CX ToF 传感器的 8×8 斜距网格重投影为躯干帧中的 3D 点，并过滤两个系统性干扰：**地板回波**和**太近回波**。传感器位姿来自 `HeadFk::tof_in_trunk`（每帧），躯干高度来自 MJCF 本身（`Model::trunk_height_m`）——没有转录的常量。

**关键设计决策**：
- **放在 kinematics 而非 tof crate**：这是纯几何，tof crate 携带 vendored ST C 驱动，robotctl 等客户端绝不能链接。
- **地板过滤在重力水平帧中计算**："向下"是 IMU 说的，不是躯干的。躯干前倾时整个头链跟着倾斜，忽略这点会在机器人俯身时把地板标为障碍。
- **FLOOR_SAFETY = 0.85**：向下光束必须覆盖传感器高度的 85% 才算地板命中。低于 1.0 让 FK 和姿态误差使接近地板的回波读为地板而非障碍——原型调优值。
- **MIN_RANGE_M = 0.10**：低于 ~10cm 传感器读数不可信（盖玻璃串扰和脉冲堆积产生虚假短回波），丢弃为噪声。
- **波束方向预计算**：8×8 单位波束方向在构造时计算，row-major，与线格式一致。
- **Zone 枚举命名每种结果**：Empty/TooClose/Floor/Hit——调用方不需要猜测数字含义。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 tof.rs |
| 文件类型 | Rust 公开模块（pub mod tof，lib.rs L26） |
| 所属 crate | kinematics |
| 行数 / 已读范围 | 387 行，全文已完整读取（L1–L387） |
| 主要证据 | 文件本身；head.rs（HeadFk）；lib.rs（Model）；math.rs |
| 不可读 / 未提供 | tofd 源码、ST VL53L5CX 数据手册、障碍物规避代码 |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 8×8 网格，ROWS=COLS=8 | L28–L30 | 常量 | confirmed |
| F2 | FOV=45°×45°，ST 数据手册 | L32–L34 | 常量 + 注释 | confirmed |
| F3 | Zone 枚举：Empty/TooClose/Floor{point}/Hit{point,range} | L38–L58 | enum | confirmed |
| F4 | Posture：gravity（躯干帧投影重力）+ trunk_height_m（可选） | L64–L72 | 结构体 | confirmed |
| F5 | Posture::default：upright [0,0,-1]，trunk_height=None | L74–L82 | 实现 | confirmed |
| F6 | Reprojector 持有 HeadFk、beams [64]、trunk_height_m | L84–L90 | 结构体 | confirmed |
| F7 | FLOOR_SAFETY=0.85，MIN_RANGE_M=0.10 | L97, L101 | 常量 | confirmed |
| F8 | 波束中心均匀分布在 FOV 内，边缘半区 inset | L104–L119 | 实现 | confirmed |
| F9 | Row 0 是网格顶部，col 0 是传感器左侧 | L106–L07 | 注释 | confirmed |
| F10 | beams() 暴露单位波束方向（给 hand.rs 用） | L131–L137 | 实现 | confirmed |
| F11 | sensor_in_trunk 委托给 HeadFk::tof_in_trunk | L139–L143 | 实现 | confirmed |
| F12 | project() 输入：ranges_m [Option<f64>;64]、head_joints、posture | L156–L161 | 签名 | confirmed |
| F13 | 地板/距离判定在重力水平帧中计算，点输出在躯干帧 | L153–L155, L165–L167 | 注释 + 实现 | confirmed |
| F14 | 地板判定：above_floor>0 && downward>0 && r*downward >= floor_threshold | L182–L185 | 实现 | confirmed |
| F15 | 太近判定：horizontal < MIN_RANGE_M | L186–L190 | 实现 | confirmed |
| F16 | Hit.range 是水平距离（从躯干原点垂直轴） | L53–L56, L193 | 注释 + 实现 | confirmed |
| F17 | level_from_gravity：重力太小时返回 identity（不 leveling） | L203–L207 | 实现 | confirmed |
| F18 | level_from_gravity 处理完全倒置（任意水平轴 + π） | L217–L224 | 实现 | confirmed |
| F19 | 测试：前向 1m 回波落在躯干前方约 1m | L246–L257 | 测试 | confirmed |
| F20 | 测试：低头时地板是地板，地板前的物体是障碍 | L263–L303 | 测试 | confirmed |
| F21 | 测试：向上光束永远不是地板 | L308–L316 | 测试 | confirmed |
| F22 | 测试：噪声带和静默有命名 | L319–L324 | 测试 | confirmed |
| F23 | 测试：前倾躯干使前向光束变为地板（IMU 是唯一见证） | L330–L363 | 测试 | confirmed |
| F24 | 测试：头部偏航使重投影点左右移动 | L368–L385 | 测试 | confirmed |

---

## 四、源码逐段解读

### 4.1 模块文档（L1–L22）

`tofd` 发布传感器看到的——传感器自身帧中沿固定波束的斜距。消费者想要机器人可行动的几何：每个回波在躯干帧中的位置，过滤两个系统性干扰：
- **地板回波**：头向下看时每个距离都看到地板。光束斜距 × 向下分量达到传感器离地高度（乘安全系数，用于姿态误差）就是地板，不是障碍。
- **太近回波**：低于 ~10cm 传感器读数不可信（盖玻璃串扰和脉冲堆积），丢弃为噪声。

传感器位姿来自 HeadFk（每帧），躯干高度来自 MJCF——没有转录常量。

放在 kinematics 而非 tof crate：纯几何，tof crate 携带 vendored ST C 驱动，robotctl 等客户端绝不能链接。

### 4.2 常量（L27–L34）

```rust
pub const ROWS: usize = 8;
pub const COLS: usize = 8;
const FOV_DEG: f64 = 45.0;
```

VL53L5CX/L8CX 的 8×8 模式，45°×45° FOV（ST 数据手册，两代相同，原型波束表用的值）。

### 4.3 Zone 枚举（L36–L58）

```rust
pub enum Zone {
    Empty,                              // 无可用回波
    TooClose,                           // 短距噪声带内
    Floor { point: [f64; 3] },         // 光束到达地板
    Hit { point: [f64; 3], range: f64 }, // 有物体
}
```

`Hit.range` 是从躯干原点垂直轴的水平距离——障碍物规避比较的停止阈值。`Floor.point` 用于绘制机器人实际确认的地面，不用于规避。

### 4.4 Posture（L60–L82）

```rust
pub struct Posture {
    pub gravity: [f64; 3],         // 躯干帧中投影重力，upright ≈ [0,0,-1]
    pub trunk_height_m: Option<f64>, // 实测躯干高度，None 回退到模型站立高度
}
```

躯干自身的姿态——头部关节无法知道的一半几何。地板过滤推理世界"向下"；躯干前倾携带整个头链，忽略这点会在机器人俯身时把地板标为障碍。

### 4.5 Reprojector（L84–L198）

**构造（L103–L125）**：
- 预计算 64 个单位波束方向。区域中心均匀分布在 FOV 内，边缘半区 inset（8 区域行有 8 个中心，不是 9 个栅栏柱）。
- Row 0 是网格顶部，col 0 是传感器左侧——与原型 streamer 排序 ULD buffer 的方式一致。
- 波束方向在传感器帧中（+x 前, +y 左, +z 上）。

**beams()（L131–L137）**：暴露波束方向给需要推理波束本身的几何（如 hand.rs 的平面拟合）。

**project()（L156–L197）**：
1. 计算传感器在躯干帧中的 pose。
2. `level = level_from_gravity(posture.gravity)`——躯干→水平的旋转。
3. `sensor_level = level.rotate(sensor.pos)`——传感器在水平帧中的位置。
4. `above_floor = sensor_level[2] + trunk_height`——传感器离地高度。
5. `floor_threshold = above_floor * FLOOR_SAFETY`。
6. 对每个 zone：
   - None → Empty（跳过）。
   - `dir = sensor.quat.rotate(beam[i])`——波束在躯干帧中的方向。
   - `dir_level = level.rotate(dir)`——波束在水平帧中的方向。
   - `downward = -dir_level[2]`——光束在世界水平线下的程度（正=向下看）。
   - `point = sensor.pos + r * dir`——回波点在躯干帧中。
   - 地板判定：`above_floor > 0 && downward > 0 && r * downward >= floor_threshold` → Floor。
   - 太近判定：`horizontal = r * sqrt(dir_level_x² + dir_level_y²) < MIN_RANGE_M` → TooClose。
   - 否则 → Hit { point, range: horizontal }。

**关键**：点输出在躯干帧，但地板/距离判定在重力水平帧中——因为"向下"是 IMU 的，不是躯干的。

### 4.6 level_from_gravity（L200–L226）

计算将躯干帧重力转到正下方的旋转：
- 重力模 < 0.5 → identity（IMU 未收敛，不 leveling 而非随机 leveling）。
- 归一化重力，计算与 down=[0,0,-1] 的叉积轴和夹角。
- 轴模 < 1e-9：完全对齐（identity）或完全倒置（任意水平轴 + π）。
- 否则 `from_axis_angle(axis/s, atan2(s,c))`。

### 4.7 测试（L228–L386）

1. **a_forward_return_lands_a_metre_ahead**：水平头中心区域 1m 回波落在躯干前方 ~1m，高度约传感器高度，水平距离 0.8–1.05m。
2. **the_floor_is_floor_and_a_thing_before_it_is_not**：低头 0.3 rad，底部行光束在地板距离处 → Floor（z ≈ -trunk_height），3/4 距离处 → Hit。
3. **an_upward_beam_is_never_the_floor**：水平头顶行光束向上，4m 回波是 Hit 不是 Floor。
4. **the_noise_band_and_the_silence_are_named**：5cm 回波 → TooClose，无回波 → Empty。
5. **a_leaning_trunk_makes_a_forward_beam_floor**：躯干前倾 0.6 rad（手持倾斜），IMU 重力投影变化，中心光束在地板距离处 → Floor；同一回波在直立躯干上是 Hit。
6. **the_head_pose_steers_the_points**：头偏航 0.8 rad 左，重投影点 y 增加 >0.3m——FK 确实在循环中。

---

## 五、控制流（ROUTE）

```
ranges_m [Option<f64>; 64] + head_joints + posture
    → sensor = HeadFk::tof_in_trunk(head_joints)  [躯干帧]
    → level = level_from_gravity(posture.gravity)  [躯干→水平]
    → above_floor = level.rotate(sensor.pos)[2] + trunk_height
    → floor_threshold = above_floor * 0.85
    → 对每个 zone:
        dir = sensor.quat * beam[i]           [躯干帧]
        dir_level = level * dir                [水平帧]
        point = sensor.pos + r * dir           [躯干帧输出]
        if r * downward >= floor_threshold → Floor
        else if horizontal < 0.10 → TooClose
        else → Hit { point, range: horizontal }
    → [Zone; 64]
```

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："地板回波被正确过滤，不被标为障碍"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | project() 的 Floor 判定 | L182–L185 |
| 触发者 | 每帧 ToF 数据 | 外部 |
| 当前装配/选择/开关 | FLOOR_SAFETY=0.85，重力水平帧计算 | L97, L165–L167 |
| 实际执行者 | 几何比较 | — |
| 成功副作用与观察点 | 地板回波 → Zone::Floor，不进入障碍物规避 | 测试 L263–L303 |
| 失败是否返回且被检查 | 无失败路径；above_floor<=0 时不做地板判定 | L182 |
| 不能覆盖的对象 | ① 透明/反光表面可能不产生回波；② 非常矮的物体可能被误判为地板；③ 楼梯/台阶几何复杂 | 物理限制 |
| status | **confirmed**（低头/前倾测试通过；极端几何需实测） | — |

### 6.2 主张二："躯干前倾时地板仍被识别为地板"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | 前倾躯干的地板过滤 | L164–L167, L330–L363 |
| 触发者 | IMU 报告非 upright 重力 | 外部 |
| 当前装配/选择/开关 | level_from_gravity 将重力转到正下方，在水平帧中判定 | L165, L203–L226 |
| 实际执行者 | 四元数旋转 + 几何比较 | — |
| 成功副作用与观察点 | 前倾时前向光束在地板距离处 → Floor | 测试 L330–L363 |
| 失败是否返回且被检查 | 重力模 <0.5 时不 leveling（回退 identity） | L205–L207 |
| 不能覆盖的对象 | ① IMU 未收敛时可能误判；② 快速运动时重力投影有滞后 | 传感器限制 |
| status | **confirmed**（前倾 0.6 rad 测试通过） | — |

---

## 七、边界与反例（BREAK）

1. **ranges_m 长度固定 64**：`[Option<f64>; N_ZONES]`，不支持可变长度。如果传感器配置为 4×4 或 16×16，需要修改。
2. **状态码解释在调用方**：`ranges_m` 已经是 Option<f64>，调用方负责将 wire 的 status 码转换为 None/Some。这个模块不碰 status。
3. **FLOOR_SAFETY=0.85 是硬编码**：原型调优值。不同传感器安装角度或机器人姿态可能需要调整。
4. **MIN_RANGE_M=0.10 是水平距离**：不是斜距。一个正下方 5cm 的回波水平距离为 0，会被标为 TooClose；一个 15cm 前但 5cm 下的回波水平距离 ~14cm，不会被过滤。
5. **地板判定用 r * downward**：假设光束是直线，不考虑折射或多路径。
6. **above_floor 来自 posture.trunk_height_m 或模型静态高度**：如果机器人在跳跃/下落，瞬时高度可能不对。
7. **level_from_gravity 只处理俯仰和横滚**：不处理偏航（重力不含偏航信息）。偏航不影响地板过滤，所以可接受。
8. **重力模 < 0.5 时不 leveling**：如果 IMU 报告的重力持续偏小（如校准问题），永远不 leveling，地板过滤可能失效。
9. **Hit.range 是从躯干原点垂直轴的水平距离**：不是从传感器位置。如果传感器离躯干原点有水平偏移（头部前伸），range 不反映到传感器的距离。
10. **不做时间滤波**：每帧独立判定。闪烁的 zone 可能在 Floor/Hit/Empty 之间跳变。
11. **波束方向是理想的**：假设所有波束均匀分布在 45° FOV 内，实际传感器可能有透镜畸变。
12. **不处理传感器自身温度漂移**：ToF 距离可能随温度变化，这里不做校准。

---

## 八、结论（按状态分级）

### confirmed
- C1：8×8 网格重投影为躯干帧 3D 点。
- C2：地板过滤在重力水平帧中计算，FLOOR_SAFETY=0.85。
- C3：太近过滤 MIN_RANGE_M=0.10（水平距离）。
- C4：传感器位姿来自 HeadFk，躯干高度来自 MJCF，无转录常量。
- C5：波束方向预计算，row-major，与线格式一致。
- C6：Zone 枚举命名每种结果（Empty/TooClose/Floor/Hit）。
- C7：level_from_gravity 处理未收敛（identity）和完全倒置（+π）。
- C8：放在 kinematics 而非 tof crate，避免客户端链接 ST C 驱动。
- C9：测试覆盖：前向回波、地板/障碍区分、向上光束、噪声带、前倾躯干、头部偏航。

### inferred
- I1：Reprojector 被 robotd 的障碍物规避使用。
- I2：tofd 发布 8×8 距离帧和 status 码，调用方转换为 Option<f64>。
- I3：Posture.gravity 来自 robot.state 的 IMU 估计。
- I4：hand.rs 使用 beams() 获取波束方向做平面拟合。

### unknown
- U1：实际障碍物规避的停止阈值。
- U2：tofd 的采样率和延迟。
- U3：FLOOR_SAFETY=0.85 的调优过程。
- U4：是否有时间滤波或多帧一致性检查。

---

## 附录 A　资料来源

1. 原文件：tof.rs（本地附件，387 行，全文已读）。
2. 同 crate 文件：head.rs（HeadFk）、lib.rs（Model）、math.rs（Quat）、hand.rs（使用 beams()）。
3. 外部参考：ST VL53L5CX/L8CX 数据手册。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-10 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
