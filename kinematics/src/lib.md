# lib.rs 文件解析

## 文件位置

`d:\microduck\kinematics\src\lib.rs`

## 核心设计决策

由 MJCF 驱动的正运动学（FK），为控制循环编译。原型 `microduck_kinematics` crate 证明了该方法——解析训练用的 MJCF、遍历树——但其查询路径是为方便而建：关节角度以 `HashMap<String, f64>` 传递，查询一个 site 会重新计算机器人的每个 body 到新分配的 `Vec` 中。在 bench 节奏下没问题，但在每 tick 都要查两只脚的 50Hz 循环里很浪费。

此处模型在加载时**编译一次**：名字解析为索引，每个 site 获得自己的扁平化 root→site 链。查询是对该链的 fold——无哈希、无分配、无 site 不悬挂的 body。角度是按 `Model` 关节顺序索引的普通 `&[f64]`；用 `Model::joint_index` 把名字解析为索引一次，而非每次查询。

正确性由两种方式锁定：`tests/fk_against_mujoco.rs` 在 64 个随机姿态上把每个 site 与 MuJoCo 自己的 `mj_kinematics` 比较；`head` 模块锁定系统其余部分所依赖的符号约定。

## 常量与类型

- `ALPHA_MJCF: &str` — 嵌入的 `assets/alpha/robot_walk.xml`，与行走策略训练用的同一 `mjlab_microduck` 场景。更新机械结构意味着替换此文件并重跑 fixture 生成器，无需改 Rust
- `SiteId(usize)` — 解析后的 site，廉价拷贝，仅在生成它的模型中有意义
- `Link` — site 链的一跳：进入 body 帧的固定变换 `rest: Pose`，若 body 有关节则为绕铰链轴的旋转 `joint: Option<(usize, [f64; 3])>`
- `Model` — 已解析并编译的运动学模型

### `Model` 字段

- `joint_names: Vec<String>` — 角度切片顺序的关节名
- `joint_ranges: Vec<Option<(f64, f64)>>` — 每关节 `[lo, hi]` 行程限制（弧度），直接来自 MJCF
- `site_names: Vec<String>`
- `chains: Vec<Box<[Link]>>` — 每 site 的扁平化 root→site 链（site 自身 rest pose 是最后一个无关节 link）
- `trunk_height: f64` — 躯干站立高度（米），即场景中 `trunk_base` 的下落高度

## 关键方法

- `parse(xml)` — 编译模型：按树顺序分配关节索引（确定性，访问 body 时即可解析其关节）；对每个 site 从 tip 到 root 收集祖先链（跳过自身为 identity 无关节的 root），反转后追加 site 自身的 rest link
- `alpha()` — 进程内解析一次的 alpha 模型（`LazyLock`），嵌入资产由 fixture 测试覆盖，运行时不可能失败
- `num_joints()` — 角度切片长度
- `joint_names()` / `joint_index(name)` — 关节名与索引
- `joint_range(joint)` — 关节行程限制，IK 必须钳制的范围
- `trunk_height_m()` — 躯干离地高度，地板过滤器的偏移
- `site(name)` / `site_names()` — site 解析
- `site_pose(site, angles)` — trunk 帧中 site 的 pose。`angles` 必须覆盖所有关节，短切片是调用点 bug 而非机器人状态，因此 panic 而非静默读零。折叠链时：`t = t * link.rest`，若有关节则 `t.quat = t.quat * Quat::from_axis_angle(axis, angles[idx])`（纯就地旋转，不组合零平移的完整 Pose）

## 单元测试描述

- `a_planar_arm_folds_like_high_school_trig` — 三节平面臂，伸直时 tip 在 x=3，肘 90° 时 x=2、y=1
- `the_trunk_frame_ignores_where_mujoco_drops_the_robot` — root 的世界放置 `pos="9 9 9"` 不渗入 trunk 帧 FK
- `the_embedded_alpha_model_has_what_the_daemon_asks_for` — alpha 模型含 `left_foot`/`right_foot`/`head_camera`/`tof` site 与关键关节
- `a_short_angle_slice_is_a_bug_not_a_zero` — 短角度切片 panic
- `a_model_without_a_trunk_is_refused_with_a_name` — 无 `trunk_base` 时拒绝并在错误中命名

## 关键摘要

`lib.rs` 是 `kinematics` crate 的入口，核心是"编译一次、查询零分配"：MJCF 在加载时解析为每 site 的扁平化链，`site_pose` 是对链的纯 fold。所有 pose 都在 trunk 帧中（root 锚定为 identity，不继承 MuJoCo 的世界放置）。正确性由 MuJoCo fixture 与 head 符号约定测试双重保证。
