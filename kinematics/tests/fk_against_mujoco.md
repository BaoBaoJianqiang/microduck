# fk_against_mujoco.rs 文件解析

## 文件位置

`d:\microduck\kinematics\tests\fk_against_mujoco.rs`

## 核心设计决策

FK 的真值来源：MuJoCo 自己的 `mj_kinematics`。

fixture 是 64 个随机关节配置，每个 site 的 pose 由 MuJoCo 计算，由 `microduck_kinematics_rs` 仓库的 `scripts/gen_fixtures.py`（通过 `uv` 运行）针对嵌入此处的同一 `robot_walk.xml` 生成。MJCF 变化时重新生成——奇偶校验是精确的（1e-6），任何真正的分歧都会大声失败。

## 测试结构

`alpha_matches_mujoco_on_every_site_of_64_random_poses`

- 容差：位置 `1e-6` 米，四元数 `1e-6`
- 加载 `fixtures/fk_alpha.json`
- 对每个样本：把关节名→角度映射到模型角度切片
- 对每个 site：计算 `site_pose`，与期望比较
  - 位置：逐分量最大绝对误差 `< POS_TOL`
  - 四元数：`q` 与 `-q` 是同一旋转，接受 MuJoCo 选择的任一符号，取 `err(1.0).min(err(-1.0))`

## 关键摘要

这是 `kinematics` FK 的核心正确性测试。通过与 MuJoCo 的 `mj_kinematics` 在 64 个随机姿态上逐 site 比对（位置 1e-6、四元数 1e-6 并处理 q/-q 二义性），保证手写刚体代数与 MJCF 解析与工业级物理引擎一致。
