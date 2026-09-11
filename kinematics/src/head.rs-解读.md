# head.rs 源码解读与架构梳理

> 分析对象：`head.rs`（353 行），kinematics crate 的头部链 FK 模块——在视觉栈实际使用的坐标系中提供头部相机/ToF 位姿，以及凝视 IK。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：`head.rs` 提供 `HeadFk`——头部链（neck_pitch → head_pitch → head_yaw → head_roll）的前向运动学，输出在**视觉栈实际使用的坐标系**中：cv2 相机轴（+x 右, +y 下, +z 前）或 ToF 传感器轴（+x 前, +y 左, +z 上）。两个常量四元数是翻译层，从原型运行时继承并针对真实图像调优；底部的 sign-pin 测试防止 MJCF 更新让机器人默默点错方向。

此外提供 `look_at`——凝视 IK：用阻尼 Gauss-Newton 求解 2×2 系统（head_pitch + head_yaw），Jacobian 用有限差分（FK ~50ns，整个求解几微秒）。neck_pitch 是调用方选择的姿态，head_roll 保持水平。

**关键设计决策**：
- **两个常量四元数做坐标翻译**：`SITE_TO_CV2`（MJCF site→cv2）和 `SENSOR_IN_CV2_Q`（传感器帧→cv2），跨机器人版本不变。
- **名称解析一次**：`HeadFk::new` 解析 site/joint 名称为索引，每帧 FK 不做字符串操作。
- **ToF 优先用 MJCF 的 tof site**：资产有时用相机位置兜底（原型资产没有 tof site），方向约定一致。
- **凝视 IK 解 2×2 耦合系统**：head_pitch 在 head_yaw 上游，pitch 倾斜 yaw 摆动的平面，近 ±90° yaw 时 pitch 失去仰角权限——逐轴更新会卡住，所以正确求解 2×2。
- **每步 clamp 到 MJCF 行程限制**：舵机机械执行这些限制，未 clamp 的答案是机器人无法保持的姿态。
- **32 元素角度数组**：alpha 有 14 关节，32 给未来鸭子留余地，永远不碰分配器。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 head.rs |
| 文件类型 | Rust 公开模块（pub mod head，lib.rs L25） |
| 所属 crate | kinematics |
| 行数 / 已读范围 | 353 行，全文已完整读取（L1–L353） |
| 主要证据 | 文件本身；lib.rs（Model/SiteId/Pose/Quat）；math.rs |
| 不可读 / 未提供 | alpha MJCF 资产、相机重投影代码、激光追踪代码 |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 输出在 cv2 相机轴或 ToF 传感器轴 | L1–L9 | 模块文档 | confirmed |
| F2 | SITE_TO_CV2 = Quat(0.5,-0.5,0.5,-0.5)，跨版本不变 | L13–L16 | 常量 + 注释 | confirmed |
| F3 | SENSOR_IN_CV2_Q = Quat(0.5,0.5,-0.5,0.5) | L18–L20 | 常量 | confirmed |
| F4 | 头部关节顺序：neck_pitch, head_pitch, head_yaw, head_roll | L23 | 常量 | confirmed |
| F5 | HeadFk 持有 &'static Model、camera SiteId、tof Option<SiteId>、joints [usize;4] | L27–L35 | 结构体 | confirmed |
| F6 | new() 解析名称，缺失则 panic（资产损坏） | L38–L50 | 实现 | confirmed |
| F7 | camera_in_trunk_cv2：site pose × SITE_TO_CV2 | L61–L64 | 实现 | confirmed |
| F8 | tof_in_trunk：有 tof site 用 site，否则借用相机位置 | L74–L85 | 实现 | confirmed |
| F9 | site_in_trunk：32 元素角度数组，只设头部 4 关节，其余零 | L89–L99 | 实现 | confirmed |
| F10 | look_at：阻尼 Gauss-Newton，2×2 系统，有限差分 Jacobian | L101–L190 | 实现 | confirmed |
| F11 | look_at 参数：TOLERANCE=1e-4, STEP_H=1e-5, LAMBDA=1e-3, MAX_STEP=0.7, 最多 30 步 | L119–L125 | 常量 | confirmed |
| F12 | 误差在相机轴中计算：[yaw, pitch] = [atan2(x,z), atan2(y, flat)] | L135–L146 | 实现 | confirmed |
| F13 | 每步 clamp 到 joint_range，clamped 报告结果仍未命中 | L113–L117, L183–L184, L186–L189 | 实现 | confirmed |
| F14 | Gaze 结构体：joints [f64;4] + clamped bool | L194–L202 | 结构体 | confirmed |
| F15 | 测试：poses finite and unit | L209–L215 | 测试 | confirmed |
| F16 | 测试：head_pitch 移动相机（防 MJCF 更新丢关节名） | L219–L231 | 测试 | confirmed |
| F17 | 测试：alpha 头部符号约定（+yaw 看左, +pitch 看下） | L237–L248 | 测试 | confirmed |
| F18 | 测试：look_at 实际看向目标（5 个方向，未 clamp） | L253–L284 | 测试 | confirmed |
| F19 | 测试：look_at 保持给定 neck_pitch，head_roll=0 | L288–L295 | 测试 | confirmed |
| F20 | 测试：正后方目标 clamp 在 yaw 极限 | L299–L313 | 测试 | confirmed |
| F21 | 测试：tof pose 精确在 MJCF tof site 上，方向与相机借用路径一致 | L320–L351 | 测试 | confirmed |

---

## 四、源码逐段解读

### 4.1 模块文档（L1–L9）

模型给出 MJCF site pose；消费者（相机重投影、ToF 点云）想要 cv2 相机轴或 ToF 传感器轴。两个常量四元数是翻译层，从原型运行时继承并针对真实图像调优。sign-pin 测试防止 MJCF 更新让机器人默默点错方向。

### 4.2 坐标转换常量（L13–L20）

```rust
pub const SITE_TO_CV2: Quat = Quat::new(0.5, -0.5, 0.5, -0.5);
pub const SENSOR_IN_CV2_Q: Quat = Quat::new(0.5, 0.5, -0.5, 0.5);
```

- `SITE_TO_CV2`：MJCF `head_camera` site → cv2 相机帧。跨机器人版本不变——各版本 site quat 只相差绝对帧选择，`q_site⁻¹ * q_cv2` 结果相同。
- `SENSOR_IN_CV2_Q`：传感器帧（+x 前, +y 左, +z 上，VL53L5CX/L8CX 集成约定）→ cv2 相机帧。

### 4.3 HeadFk 结构体（L27–L35）

```rust
pub struct HeadFk {
    model: &'static Model,
    camera: SiteId,
    tof: Option<SiteId>,
    joints: [usize; 4],
}
```

名称解析一次，per-frame FK 不做字符串工作。`tof` 是 Option——资产有 tof site 时用传感器真实位置（离相机几厘米），没有时借用相机位置。

### 4.4 new / alpha（L37–L54）

`new()` 解析 `head_camera` site（必须存在，否则 panic——资产损坏）、`tof` site（可选）、4 个头部关节（必须存在）。`alpha()` 便捷构造 `Model::alpha()`。

### 4.5 camera_in_trunk_cv2（L56–L64）

```rust
pub fn camera_in_trunk_cv2(&self, joints: [f64; 4]) -> Pose {
    let site = self.site_in_trunk(self.camera, joints);
    Pose::new(site.pos, site.quat * SITE_TO_CV2)
}
```

取 site 在躯干帧中的 pose，然后旋转到 cv2 轴。位置不变（site 的位置就是相机的位置），只有方向需要转换。

### 4.6 tof_in_trunk（L66–L85）

两条路径：
- **有 tof site**：`site.quat * SITE_TO_CV2 * SENSOR_IN_CV2_Q`——先转到 cv2，再转到传感器轴。
- **无 tof site（兜底）**：借用相机位置，`cam.quat * SENSOR_IN_CV2_Q`。

注释解释：原型借用相机位置因为其资产早于 tof site；兜底保留，方向约定两种方式一致（pin 测试保持两个安装平行）。

### 4.7 site_in_trunk（L87–L99）

```rust
let mut angles = [0.0f64; 32];
for (idx, angle) in self.joints.into_iter().zip(joints) {
    angles[idx] = angle;
}
self.model.site_pose(site, &angles[..self.model.num_joints()])
```

32 元素数组（alpha 有 14 关节，留余地），只设头部 4 关节，其余为零。头部链挂在躯干上，所以腿不能在躯干帧内移动它。

### 4.8 look_at（L101–L190）

凝视 IK：求解让相机光轴指向躯干帧目标点的头部关节。

**问题结构**：4 个关节中，head_pitch 和 head_yaw 做瞄准，neck_pitch 是调用方选择的姿态，head_roll 保持水平。两个关节真正耦合：head_pitch 在 head_yaw 上游，pitch 倾斜 yaw 摆动的平面，近 ±90° yaw 时 pitch 轴与相机前向对齐，失去仰角权限。逐轴更新会卡住，所以解 2×2 系统。

**算法**：
1. 误差函数：当前 pose 的相机轴中目标的 [yaw, pitch] 偏差。
2. 初始化：neck_pitch clamp 到范围，其余为 0。
3. 最多 30 步：
   - 计算误差，若 max(|yaw|,|pitch|) < 1e-4 则收敛。
   - 有限差分计算 2×2 Jacobian（对 head_pitch/head_yaw 各扰动 1e-5）。
   - 解 `(JᵀJ + λI)Δ = -Jᵀe`（Levenberg 阻尼，λ=1e-3）。
   - 步长缩放：不超过 0.7 rad/步。
   - clamp 到关节范围。
4. 返回 Gaze { joints, clamped }——clamped 表示结果仍未命中目标（行程限制或万向节几何）。

**为什么有限差分**：FK ~50ns，整个求解几微秒。解析 Jacobian 更复杂，有限差分足够快且不易错。

### 4.9 Gaze（L193–L202）

```rust
pub struct Gaze {
    pub joints: [f64; 4],  // [neck_pitch, head_pitch, head_yaw, head_roll]
    pub clamped: bool,     // 目标超出可达范围，joints 是最接近的凝视
}
```

### 4.10 测试（L204–L352）

1. **poses_are_finite_and_unit**：零位姿时位置有限、四元数单位。
2. **head_pitch_moves_the_camera**：head_pitch=0.5 时相机位置移动 >1mm——防 MJCF 更新丢关节名。
3. **alpha_head_axis_conventions**：+head_yaw 看左（+y），+head_pitch 看下（-z）。这是 gaze 和激光追踪符号选择的基准。
4. **look_at_actually_looks_at_the_target**：5 个目标方向（正前/上左/下右/硬左/陡下），验证相机光轴通过目标（点积 > 1-1e-6），且未 clamp。
5. **look_at_keeps_the_neck_it_was_given**：neck_pitch=0.3 保持不变，head_roll=0。
6. **a_target_behind_the_robot_clamps_at_the_yaw_limit**：正后方需 180° yaw，MJCF 允许 ±170°，结果 clamp 在 yaw 极限，clamped=true。
7. **the_tof_pose_sits_on_the_mjcf_tof_site**：tof pose 精确在 tof site 上（1e-12），方向与相机借用路径一致（1e-9）。

---

## 五、控制流（ROUTE）

**FK 路径**：
```
joints [neck_pitch, head_pitch, head_yaw, head_roll]
    → site_in_trunk: 32 元素数组，只设头部关节
    → Model::site_pose: fold 链
    → 坐标转换：× SITE_TO_CV2 (相机) 或 × SITE_TO_CV2 × SENSOR_IN_CV2_Q (ToF)
    → Pose (躯干帧，目标坐标系)
```

**IK 路径**：
```
target_in_trunk + neck_pitch
    → 初始化 joints = [clamp(neck), 0, 0, 0]
    → 最多 30 步阻尼 Gauss-Newton:
        误差 = 相机轴中 [yaw, pitch] 偏差
        Jacobian = 有限差分 (STEP_H=1e-5)
        解 (JᵀJ + λI)Δ = -Jᵀe
        步长缩放 (MAX_STEP=0.7) + clamp 到关节范围
    → Gaze { joints, clamped }
```

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："look_at 收敛后相机光轴通过目标"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | look_at 返回的 Gaze.joints | L118–L190 |
| 触发者 | 调用方传入目标点和 neck_pitch | 外部 |
| 当前装配/选择/开关 | 阻尼 Gauss-Newton，TOLERANCE=1e-4，最多 30 步 | L119–L125, L150 |
| 实际执行者 | 2×2 线性求解 + 有限差分 FK | — |
| 成功副作用与观察点 | 残差 < 1e-4，相机光轴点积 > 1-1e-6 | 测试 L253–L284 |
| 失败是否返回且被检查 | clamped=true 报告未命中；行列式 < 1e-12 时 break | L178–L179, L188 |
| 不能覆盖的对象 | ① 目标在可达范围外时 clamp（非失败）；② 初始值在奇异点附近可能收敛慢 | 边界 |
| status | **confirmed**（5 个可达目标测试通过；不可达目标 clamped） | — |

### 6.2 主张二："SITE_TO_CV2 跨机器人版本不变"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | SITE_TO_CV2 常量 | L13–L16 |
| 触发者 | 所有 camera_in_trunk_cv2 调用 | 外部 |
| 当前装配/选择/开关 | 硬编码常量 Quat(0.5,-0.5,0.5,-0.5) | L16 |
| 实际执行者 | 四元数乘法 | — |
| 成功副作用与观察点 | 不同版本 MJCF 的 site quat 只差绝对帧选择 | 注释 L14–L15 |
| 失败是否返回且被检查 | 无运行时检查；sign-pin 测试防止方向错误 | 测试 L237–L248 |
| 不能覆盖的对象 | ① 如果新版本 MJCF 改变了相机安装方向（非仅绝对帧），常量需更新；② 非 alpha 机器人可能不同 | 范围外 |
| status | **conditional**（alpha  confirmed；其他版本需验证） | — |

---

## 七、边界与反例（BREAK）

1. **look_at 只优化 head_pitch 和 head_yaw**：neck_pitch 固定为调用方给定值，head_roll 固定为 0。如果目标需要 neck 配合才能看到，无法达到。
2. **初始值总是 [neck, 0, 0, 0]**：不做热启动。如果从上一帧的 gaze 继续，可能更快收敛，但当前实现总是从零开始。
3. **有限差分 Jacobian 的 STEP_H=1e-5**：如果 FK 对关节的灵敏度极低（接近奇异），有限差分可能有数值噪声。
4. **MAX_STEP=0.7 rad/步**：大误差时最多转 0.7 rad（约 40°），需要多步。对于 180° 目标，至少需要 3 步。
5. **clamped 只报告残差 >= TOLERANCE**：不区分是行程限制还是万向节几何导致的未命中。
6. **joint_range 为 None 时回退 ±π**：如果舵机实际有物理限制但 MJCF 未声明，IK 可能给出无法执行的答案。
7. **ToF 兜底路径借用相机位置**：如果实际 ToF 传感器离相机几厘米，兜底路径有几厘米的位置误差。方向约定一致（pin 测试），但位置不精确。
8. **32 元素角度数组**：如果未来机器人超过 32 关节，会 panic（assert L93）。
9. **site_in_trunk 设所有非头部关节为零**：如果腿/躯干关节影响头部位置（在躯干帧内不影响，但在世界帧中影响），这里不考虑。
10. **look_at 不考虑速度/加速度限制**：只给出目标关节角，不规划轨迹。舵机可能无法瞬时到达。

---

## 八、结论（按状态分级）

### confirmed
- C1：HeadFk 提供 cv2 和 ToF 传感器坐标系的头部 FK。
- C2：两个常量四元数做坐标翻译，SITE_TO_CV2 跨版本不变。
- C3：名称解析一次，per-frame FK 无字符串操作。
- C4：ToF 优先用 MJCF tof site，无则借用相机位置。
- C5：look_at 用阻尼 Gauss-Newton 解 2×2 耦合系统，有限差分 Jacobian。
- C6：每步 clamp 到 MJCF 行程限制，clamped 报告未命中。
- C7：neck_pitch 是姿态参数，head_roll 保持水平。
- C8：测试覆盖：有限/单位、head_pitch 有效、符号约定、look_at 精度、neck 保持、clamp、tof site 精度。

### inferred
- I1：look_at 被 robotd 或行为系统用于注视目标（人/物）。
- I2：camera_in_trunk_cv2 被相机重投影代码使用。
- I3：tof_in_trunk 被 tof.rs 的 Reprojector 使用。
- I4：alpha 模型的 head_yaw 范围是 ±170°（测试 L304–L307 暗示）。

### unknown
- U1：look_at 的实际收敛时间（微秒级，需基准测试）。
- U2：是否有其他机器人版本（beta）的头部链。
- U3：SITE_TO_CV2 如何从原型运行时调优得到。
- U4：look_at 是否在 50Hz 循环中每帧调用。

---

## 附录 A　资料来源

1. 原文件：head.rs（本地附件，353 行，全文已读）。
2. 同 crate 文件：lib.rs（Model）、math.rs（Quat/Pose）、tof.rs（使用 HeadFk）。
3. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-10 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
