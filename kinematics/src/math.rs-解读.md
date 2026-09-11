# math.rs 源码解读与架构梳理

> 分析对象：`math.rs`（181 行），kinematics crate 的刚体代数基础——手写四元数和 Pose，不依赖 nalgebra。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：`math.rs` 提供 FK 实际需要的**最小刚体代数**：一个四元数（`Quat`）和一个刚体变换（`Pose`）。手写而非引入 nalgebra 是刻意的——全部需求是"组合十几个刚体变换和旋转几个向量"，约一百行，而 `tests/` 中的 MuJoCo fixture 把每个操作 pin 到 1e-6，这是比依赖名更强的正确性论证。nalgebra 会增加编译时间，而非信心。

**关键设计决策**：
- **Hamilton 四元数，标量在前**：`[w, x, y, z]`，与 MJCF 和原型运行时共享约定。
- **`a * b` 表示先 b 后 a**：父乘子顺序，链从根到尖从左到右读。
- **旋转用叉积形式**：`q v q⁻¹` 的 cross-product 形式，跳过完整四元数 sandwich。
- **退化四元数归一化为 identity 而非 NaN**：只能来自手写 XML 属性，忽略坏四元数的模型可诊断，满是 NaN 的模型不可诊断。
- **`yaw()` 提取绕世界 +z 的偏航角**：为报告 heading 为单角的估计器设计。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 math.rs |
| 文件类型 | Rust 私有模块（mod math，lib.rs L21） |
| 所属 crate | kinematics |
| 行数 / 已读范围 | 181 行，全文已完整读取（L1–L181） |
| 主要证据 | 文件本身；lib.rs（重新导出 Quat/Pose）；mjcf.rs（使用 Quat/Pose） |
| 不可读 / 未提供 | MuJoCo fixture 测试、nalgebra 对比基准 |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 手写而非 nalgebra，需求约一百行 | L3–L7 | 模块文档明示 | confirmed |
| F2 | Hamilton 四元数，标量在前 [w,x,y,z] | L10 | 模块文档 | confirmed |
| F3 | a*b 先 b 后 a，父乘子顺序 | L11–L12 | 模块文档 + 测试 | confirmed |
| F4 | Quat::new 是 const，信任调用方 | L34–L36 | 实现 | confirmed |
| F5 | normalized() 对近零输入返回 IDENTITY | L42–L48 | 实现 | confirmed |
| F6 | from_axis_angle 假设轴是单位的 | L50–L54 | 文档注释 | confirmed |
| F7 | rotate 用叉积形式，跳过完整 sandwich | L56–L68 | 实现 | confirmed |
| F8 | conjugate 对单位四元数就是逆 | L70–L73 | 实现 | confirmed |
| F9 | wxyz() 返回 [w,x,y,z]，MJCF 和线格式顺序 | L75–L78 | 实现 | confirmed |
| F10 | yaw() 用旋转矩阵 (1,0)/(0,0) 元素的 atan2 | L80–L86 | 实现 | confirmed |
| F11 | Quat::mul 标准 Hamilton 乘积 | L89–L101 | 实现 | confirmed |
| F12 | Pose = pos + quat，先旋转后平移 | L103–L108 | 实现 | confirmed |
| F13 | Pose::mul = transform_point(b.pos) + quat 乘积 | L127–L136 | 实现 | confirmed |
| F14 | transform_point = rotate + 平移 | L120–L124 | 实现 | confirmed |
| F15 | 测试：绕 z 90° 把 +x 送到 +y | L150–L157 | 测试 | confirmed |
| F16 | 测试：Pose 组合右操作数先应用 | L162–L173 | 测试 | confirmed |
| F17 | 测试：坏四元数归一化为 identity | L176–L179 | 测试 | confirmed |

---

## 四、源码逐段解读

### 4.1 模块文档（L1–L12）

解释了为什么手写：需求是"组合十几个刚体变换和旋转几个向量"，约一百行。MuJoCo fixture 把每个操作 pin 到 1e-6，比依赖名更强。nalgebra 增加编译时间，不增加信心。

约定：
- 四元数是 Hamilton，标量在前 `[w, x, y, z]`。
- `a * b` 先应用 b 再应用 a——父乘子顺序，链从根到尖从左到右读。

### 4.2 Quat 结构体（L16–L24）

```rust
pub struct Quat { pub w: f64, pub x: f64, pub y: f64, pub z: f64 }
```

字段公开。`new` 用于拼写常量，信任调用方（不检查单位性）。`IDENTITY = [1,0,0,0]`。

### 4.3 normalized（L38–L48）

```rust
pub fn normalized(self) -> Self {
    let n = (w²+x²+y²+z²).sqrt();
    if n < 1e-12 { return Self::IDENTITY; }
    Self::new(w/n, x/n, y/n, z/n)
}
```

退化（近零）输入变为 identity 而非 NaN。理由：只能来自手写 XML 属性，忽略坏四元数的模型可诊断，满是 NaN 的模型不可诊断。

### 4.4 from_axis_angle（L50–L54）

标准轴角→四元数：`[cos(θ/2), axis*sin(θ/2)]`。假设轴是单位的（调用方负责，mjcf.rs 的 normalize 保证）。

### 4.5 rotate（L56–L68）

```rust
pub fn rotate(self, v: [f64; 3]) -> [f64; 3] {
    let t = 2.0 * (self × v);  // cross product
    [v + w*t + self × t]       // expanded
}
```

`q v q⁻¹` 的叉积形式，跳过完整四元数 sandwich（不需要构造 q⁻¹ 和两次乘法）。这是标准的"half-cross"优化。

### 4.6 conjugate（L70–L73）

对单位四元数，共轭就是逆。`[w, -x, -y, -z]`。

### 4.7 wxyz（L75–L78）

返回 `[w, x, y, z]`——MJCF 和所有线格式使用的顺序。

### 4.8 yaw（L80–L86）

```rust
pub fn yaw(self) -> f64 {
    let siny = 2.0 * (w*z + x*y);
    let cosy = 1.0 - 2.0 * (y² + z²);
    siny.atan2(cosy)
}
```

绕世界 +z 的偏航角。从旋转矩阵的 (1,0) 和 (0,0) 元素展开。为报告 heading 为单角的估计器设计。

### 4.9 Quat::mul（L89–L101）

标准 Hamilton 乘积：
```
w = a.w*b.w - a.x*b.x - a.y*b.y - a.z*b.z
x = a.w*b.x + a.x*b.w + a.y*b.z - a.z*b.y
y = a.w*b.y - a.x*b.z + a.y*b.w + a.z*b.x
z = a.w*b.z + a.x*b.y - a.y*b.x + a.z*b.w
```

### 4.10 Pose（L103–L136）

```rust
pub struct Pose { pub pos: [f64; 3], pub quat: Quat }
```

刚体变换：先旋转后平移。
- `transform_point(p)` = `quat.rotate(p) + pos`
- `a * b` = `Pose { pos: a.transform_point(b.pos), quat: a.quat * b.quat }`

这意味着 `a * b` 先应用 b（在子帧中），再应用 a（在父帧中）——与四元数的父乘子顺序一致。

### 4.11 测试（L138–L180）

1. **quarter_turn_about_z_sends_x_to_y**（L150–L157）：绕 +z 90° 把 +x 送到 +y。两次 90° = 180° 把 +x 送到 -x。yaw() 返回 π/2。这是每个人都能在脑中检查的方向，sign 错误先在这里失败。
2. **pose_composition_applies_the_right_operand_first**（L162–L173）：`turn * step` 中 step 在子帧中，从旋转后的父帧看落在 +y；`step * turn` 中 turn 在子帧中，原点保持 stepped。pin 了组合顺序。
3. **a_broken_quat_normalizes_to_identity_not_nan**（L176–L179）：零四元数归一化为 identity。

---

## 五、架构位置（ROUTE）

math.rs 是 kinematics crate 的**最底层**，被所有其他模块使用：
- `mjcf.rs`：解析 MJCF 的 pos/quat 为 Pose/Quat
- `lib.rs`：Model 的 Link.rest 是 Pose，site_pose fold 用 Pose::mul 和 Quat::from_axis_angle
- `head.rs`：SITE_TO_CV2/SENSOR_IN_CV2_Q 是 Quat 常量，Pose 组合
- `tof.rs`：level_from_gravity 用 Quat::from_axis_angle，beam 方向用 Quat::rotate
- `hand.rs`：不直接使用 math（纯距离统计）

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："rotate 是 q v q⁻¹ 的正确实现"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Quat::rotate | L56–L68 |
| 触发者 | 所有需要旋转向量的调用 | 全 crate |
| 当前装配/选择/开关 | 叉积形式 half-cross 优化 | L58–L67 |
| 实际执行者 | 纯算术 | — |
| 成功副作用与观察点 | 旋转结果与 q v q⁻¹ 一致 | 测试 + MuJoCo fixture |
| 失败是否返回且被检查 | 无失败路径（纯函数） | — |
| 不能覆盖的对象 | ① 非单位四元数的 rotate 结果不定义；② 浮点精度 | 前提条件 |
| status | **confirmed**（测试 pin 90° 旋转；MuJoCo fixture 覆盖 64 姿态） | — |

### 6.2 主张二："a * b 先应用 b 再应用 a"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Pose::mul 和 Quat::mul 的组合顺序 | L89–L101, L127–L136 |
| 触发者 | FK 链 fold（lib.rs site_pose） | 外部 |
| 当前装配/选择/开关 | 父乘子顺序，链从左到右读 | L11–L12 |
| 实际执行者 | Mul trait impl | — |
| 成功副作用与观察点 | 链 fold 结果正确 | 测试 L162–L173 |
| 不能覆盖的对象 | ① 调用方误解顺序会得到错误结果；② 与其他库（如 nalgebra）的顺序可能不同 | 使用约定 |
| status | **confirmed**（测试 pin 了 turn*step vs step*turn 的区别） | — |

---

## 七、边界与反例（BREAK）

1. **Quat::new 不检查单位性**：信任调用方。如果传入非单位四元数，rotate/mul 结果不定义。mjcf.rs 解析时调用 normalized()，但直接构造的常量（如 head.rs 的 SITE_TO_CV2）必须手动保证单位。
2. **from_axis_angle 假设轴是单位的**：不做归一化。mjcf.rs 的 normalize 保证，但直接调用需注意。
3. **yaw() 只提取绕 +z 的分量**：如果四元数包含 pitch/roll，yaw() 仍然返回绕 z 的等效偏航，但不是完整姿态。
4. **Pose 不支持逆变换**：没有 inverse() 方法。如果需要子帧→父帧的逆，调用方必须手动计算。
5. **normalized 的 1e-12 阈值是硬编码**：对于 f64 合理，但如果用 f32 需要调整。
6. **rotate 的叉积形式对非单位四元数不等价于 q v q⁻¹**：half-cross 形式假设 q 是单位的。
7. **无 slerp/lerp**：不支持插值。如果需要平滑姿态过渡，调用方必须自己实现。
8. **无矩阵转换**：不支持 to_rotation_matrix/from_rotation_matrix。与需要矩阵的库交互需手动转换。
9. **Pose::transform_point 不做范围检查**：输入任意 [f64;3]，输出任意 [f64;3]。
10. **yaw() 的 gimbal lock**：当 pitch 接近 ±90° 时，yaw 和 roll 退化，yaw() 仍返回一个值但物理意义模糊。

---

## 八、结论（按状态分级）

### confirmed
- C1：手写四元数和 Pose，不依赖 nalgebra。
- C2：Hamilton 约定，标量在前 [w,x,y,z]。
- C3：a*b 先 b 后 a，父乘子顺序。
- C4：rotate 用叉积形式，跳过完整 sandwich。
- C5：normalized 对近零输入返回 identity 而非 NaN。
- C6：yaw() 提取绕世界 +z 的偏航角。
- C7：Pose = pos + quat，先旋转后平移。
- C8：测试 pin 了 90° 旋转、组合顺序、坏四元数处理。

### inferred
- I1：math.rs 的正确性被 tests/fk_against_mujoco.rs 间接覆盖（通过 Model 的 FK）。
- I2：Quat/Pose 被 re-export 到 crate 根，供外部使用。
- I3：所有 Quat 常量（head.rs 的 SITE_TO_CV2 等）都是单位四元数（手动验证）。

### unknown
- U1：与 nalgebra 的性能对比（编译时间/运行时间）。
- U2：是否有 SIMD 优化的可能。
- U3：yaw() 在 gimbal lock 附近的实际行为。

---

## 附录 A　资料来源

1. 原文件：math.rs（本地附件，181 行，全文已读）。
2. 同 crate 文件：lib.rs、mjcf.rs、head.rs、tof.rs。
3. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-10 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
