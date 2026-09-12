# math.rs 文件解析

## 文件位置

`d:\microduck\kinematics\src\math.rs`

## 核心设计决策

FK 实际需要的最小刚体代数：一个四元数和一个 pose。

**手写而非引入 nalgebra**：全部需求是"组合十几个刚体变换并旋转几个向量"——约一百行——而 `tests/` 中的 MuJoCo fixture 把每一个都锁定到 1e-6，这比依赖的名字更有力。nalgebra 会增加编译时间而非信心。

### 约定（与 MJCF 和原型运行时共享）

- 四元数是 Hamilton 积，标量在前：`[w, x, y, z]`
- `a * b` 表示先应用 `b` 再应用 `a`——父乘子顺序，因此链从左到右读为 root→tip

## 类型分析

### `Quat`

旋转。在本 crate 构造处均为单位四元数；`new` 用于拼写常量并信任调用者。

- `IDENTITY` — `[1, 0, 0, 0]`
- `new(w, x, y, z)`
- `normalized()` — 单位长度。退化（近零）输入变为 identity 而非 NaN——只能来自手写 XML 属性，忽略坏四元数的模型可诊断，而满是 NaN 的模型不可诊断
- `from_axis_angle(axis, angle)` — 绕**单位**轴旋转 `angle` 弧度
- `rotate(v)` — 旋转向量：`q v q⁻¹`，用叉积形式跳过完整四元数三明治
- `conjugate()` — 逆旋转（对单位四元数即共轭）
- `wxyz()` — `[w, x, y, z]`，MJCF 与所有线格式使用的顺序
- `yaw()` — 绕世界 +z 的偏航，供报告 heading 为单角度的估计器使用

`Mul` 实现：标准 Hamilton 积。

### `Pose`

刚体变换：先按 `quat` 旋转，再按 `pos` 平移。

- `IDENTITY` — 零平移 + identity 旋转
- `new(pos, quat)`
- `transform_point(p)` — 把此 pose 帧中的点表达到父帧
- `Mul` — `self * b`：`pos = self.transform_point(b.pos)`，`quat = self.quat * b.quat`

## 单元测试描述

- `quarter_turn_about_z_sends_x_to_y` — 绕 +z 四分之一圈把 +x 送到 +y；两个四分之一圈合成半圈把 +x 送到 -x；`yaw()` 正确
- `pose_composition_applies_the_right_operand_first` — `a * b` 意为"先 b 后 a"：先转后移 vs 先移后转的区别，锁定链 fold 用的是哪种
- `a_broken_quat_normalizes_to_identity_not_nan` — 零四元数归一化为 identity

## 关键摘要

`math.rs` 是约 140 行的手写刚体代数，仅含 `Quat` 与 `Pose`。刻意不依赖 nalgebra：需求小，且由 MuJoCo fixture 提供比依赖名更强的正确性保证。核心约定是 Hamilton 标量前四元数与"父乘子"组合顺序。
