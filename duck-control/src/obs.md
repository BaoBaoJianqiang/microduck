# obs.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\obs.rs`

## 核心设计决策

`obs.rs` 构建策略所见的**观测向量**——这是 crate 中**风险最高的代码**。它是一个 61 个浮点数的扁平数组，每个索引都必须与策略训练时一致。错误的偏移不会大声失败——它会产生一个看起来合理但会摔倒的机器人，症状看起来像调参或时序问题而非索引问题。

所有 alpha 策略都是 `obs[1,61] → actions[1,14]`，已在行走、站立、地面拾取、踢球、坐下中验证。因此只有一种布局。

## 观测布局

```
index   width  contents
0..3        3  陀螺，躯干坐标系，rad/s
3..6        3  投影重力，躯干坐标系，单位向量
6..20      14  关节位置 - 初始位姿，排除嘴
20..34     14  关节速度，排除嘴
34..48     14  上一动作，排除嘴
48..61     13  命令（见下）
```

### 命令块（无第二真相源的部分）

```
48..51      3  vx, vy, vyaw
51..55      4  neck_pitch, head_pitch, head_yaw, head_roll
55..57      2  body x, y      — 恒为零，训练中未绑定
57          1  body z
58          1  body roll
59          1  body pitch
60          1  body yaw       — 恒为零，训练中未绑定
```

两个易出错点（已对照 `microduck_runtime` 的 `control_step` 确认）：
1. **body x、y、yaw 硬编码为零**——它们在训练环境中未绑定，全零 body 命令是*标称*编码。
2. **头部目标走命令块，不在策略输出上叠加**——原型两种都做，叠加会让头弯两次。
3. body 块内部顺序是 `z, roll, pitch`，不是 `z, pitch, roll`。

## 常量与类型

- `OBS_LEN = 61`、`ACTION_LEN = 14`、`OBS_JOINTS = 14`（NUM_JOINTS-1）、`COMMAND_LEN = 13`
- `Command`：`twist`（前/左/偏航率）、`head`（4 个颈部/头部角度）、`body`（`BodyPose { z, roll, pitch }`）
- `BodyPose`：站立身体姿态偏移（slice 2 不可命令，仅保持布局完整）

## 函数分析

### `policy_joints(values) -> [f64; 14]`
返回策略所见关节（跳过嘴），用于位置、速度、初始位姿，使三块不会在覆盖哪些关节、顺序上产生分歧。返回数组（非迭代器）使宽度成为类型的一部分，编译期检查而非运行时计数。

### `joint_of(slot) -> usize`
策略槽位到关节索引的映射：嘴之前直通，嘴及之后 +1。读写双向共用同一映射，避免 `policy_joints`（读）与 `Observation::scatter_action`（写）不一致。

### `fill<const N>(block, values)`
写一块并在过程中窄化为 `f32`。两侧都是同宽定长数组，编译器拒绝宽度不匹配（`zip` 会静默停在较短侧，留下尾部为零——零在观测中意味着关节在初始位姿，策略会对一个不存在的机器人行动）。

### `Observation::build(...) -> Self`
组装观测。`joint_positions` 是绝对的，策略看到的是相对于初始位姿的差值。`last_action` 是上一策略输出（原始，未经动作缩放）。用 `split_first_chunk_mut::<N>` 按名将缓冲切成各块，每块宽度在类型中，`fill` 只能拿到等长源。

### `Observation::scatter_action(action) -> [f64; 15]`
将策略 14 个输出映射回 15 个关节，嘴保持不变。这是 `policy_joints` 的镜像，通过同一 `joint_of` 映射。

### `Command::twist_magnitude() -> f64`
速度命令的模长（仅 twist，不含头部和身体），用于选择行走 vs 站立。

## 单元测试描述

- `the_layout_widths_sum_to_the_declared_input`：块宽之和 = 61，且 `OBS_JOINTS == ACTION_LEN`。
- `every_block_lands_at_its_documented_offset`：用可区分值验证六个块边界。
- `unbound_body_axes_are_always_zero`：body x/y/yaw 恒为零。
- `the_body_block_is_z_roll_pitch`：body 块顺序为 z、roll、pitch。
- `joint_positions_are_relative_to_the_home_pose`：关节位置相对初始位姿（初始位姿下读 0）。
- `the_mouth_is_excluded_from_the_observation`：嘴的移动不影响观测关节槽。
- `scattering_an_action_skips_the_mouth`：动作散射跳过嘴且不错位。
- `twist_magnitude_ignores_head_and_body`：twist 模长只算 twist。

## 关键摘要

`obs.rs` 是 61→14 策略契约的编码层。通过定长数组切片、编译期断言、单一 `joint_of` 映射等手段，在类型系统层面保证观测布局与策略训练一致。嘴关节在观测和动作两侧都被跳过，body 未绑定轴恒零，头部目标走命令而非叠加。所有布局细节均对照 `microduck_runtime` 确认。
