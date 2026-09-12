# head.rs 文件解析

## 文件位置

`d:\microduck\kinematics\src\head.rs`

## 核心设计决策

头部链 FK，输出到视觉栈实际使用的坐标系。

模型给出 MJCF site pose；消费者——相机重投影、ToF 点云——需要 cv2 相机轴（+x 右、+y 下、+z 前）或 ToF 传感器自身的轴（+x 沿光轴向前、+y 左、+z 上）。这里的两个常量四元数就是这个翻译层，从原型运行时搬来（在真实图像上调过）；底部的符号锁定测试是防止 MJCF 更新让机器人悄悄点头方向错误的保障。

## 常量

- `SITE_TO_CV2: Quat = Quat::new(0.5, -0.5, 0.5, -0.5)` — MJCF `head_camera` site → cv2 相机帧。跨版本常量：各版本 site 四元数仅因绝对帧选择不同，`q_site⁻¹ * q_cv2` 结果相同
- `SENSOR_IN_CV2_Q: Quat = Quat::new(0.5, 0.5, -0.5, 0.5)` — 传感器帧（+x 前、+y 左、+z 上，VL53L5CX/L8CX 集成约定）→ cv2 相机帧
- `HEAD_JOINTS: [&str; 4] = ["neck_pitch", "head_pitch", "head_yaw", "head_roll"]` — 头部关节顺序

## `HeadFk` 结构体

名字一次性解析，使每帧 FK 不做任何字符串操作。

- `model: &'static Model`
- `camera: SiteId` — `head_camera`
- `tof: Option<SiteId>` — MJCF 自身的 `tof` site（若资产有），传感器真实位置，离相机几厘米
- `joints: [usize; 4]` — 四个头关节的索引

### 方法

- `new(model)` / `alpha()` — 解析头链；缺关节则 panic（嵌入资产由测试覆盖）
- `camera_in_trunk_cv2(joints)` — trunk 帧中相机 pose（cv2 轴）。`joints = [neck_pitch, head_pitch, head_yaw, head_roll]` 弧度；其余关节取零（头链挂在 trunk 上，腿在 trunk 帧内无法移动它）
- `tof_in_trunk(joints)` — trunk 帧中 ToF 传感器 pose。优先锚定 MJCF 自身 `tof` site；回退借用相机位置（原型资产早于该 site）。方向约定两种情况相同
- `site_in_trunk(site, joints)` — 内部辅助：用 32 长零数组（留余量给未来鸭子，永不分配）填四个头关节后调 `site_pose`

### `look_at(target_in_trunk, neck_pitch) -> Gaze`

凝视 IK：让相机指向 trunk 帧中某点的头关节。

- 只有 `head_pitch` 与 `head_yaw` 负责瞄准；`neck_pitch` 是调用者选的姿态，`head_roll` 保持水平
- 两者真正耦合：`head_pitch` 在链中位于 `head_yaw` **上游**，pitch 会倾斜 yaw 转动的平面，在 yaw ±90° 附近 pitch 关节失去仰角权限（轴与相机前向对齐）。逐轴更新会在此卡住，因此正确求解 2×2 系统：阻尼 Gauss-Newton 对抗真实 FK，Jacobian 用有限差分（FK 约 50ns，整个求解几微秒）
- 每步钳制到 MJCF 行程限制（舵机机械执行，未钳制的答案是机器人无法保持的姿态）
- `clamped` 报告结果仍未命中目标——无论是行程限制还是万向节几何：调用者的"机器人已尽可能靠近地看"

## `Gaze` 结构体

- `joints: [f64; 4]` — 可直接给 `robot.head` 意图
- `clamped: bool` — 目标超出头部行程；`joints` 是限制允许的最近凝视

## 单元测试描述

- `poses_are_finite_and_unit` — pose 有限且四元数单位
- `head_pitch_moves_the_camera` — 动 `head_pitch` 必须移动相机（防止 MJCF 更新丢关节名）
- `alpha_head_axis_conventions` — 锁定 alpha 头符号约定：+head_yaw 向左（+y），+head_pitch 向下（-z）
- `look_at_actually_looks_at_the_target` — IK 用自身 FK 验证：可达目标的光轴穿过目标
- `look_at_keeps_the_neck_it_was_given` — 姿态参数被保持而非求解；head_roll 保持零
- `a_target_behind_the_robot_clamps_at_the_yaw_limit` — 正后方需 180° yaw 但 MJCF 允许 ±170°，答案钳制在 yaw 极限并 `clamped=true`
- `the_tof_pose_sits_on_the_mjcf_tof_site` — ToF pose 精确落在 MJCF `tof` site 上，方向约定与借用相机路径一致

## 关键摘要

`head.rs` 提供头链 FK 与凝视 IK，输出到 cv2/传感器轴。两个常量四元数是 MJCF→cv2→传感器轴的翻译层。`look_at` 用阻尼 Gauss-Newton（2×2，有限差分 Jacobian）求解 head_pitch/head_yaw 耦合系统，每步钳制到行程限制。符号约定由测试锁定，防止 MJCF 更新导致点头方向错误。
