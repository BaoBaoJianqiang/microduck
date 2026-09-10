# `obs.rs` 解读

## 概述

`obs.rs`（391 行）构建策略网络所见的**观测向量**——一个 61 维的 `f32` 扁平数组。文件开头的文档注释开门见山：**这是整个 crate 中风险最高的代码**。

核心原因：61 个浮点数的**每一个索引**都必须与策略训练时的输入布局严格匹配。偏移错误**不会大声报错**——它产生一个看起来正常的机器人然后摔倒，症状看起来像调参或时序问题，而不是索引问题。

所有 alpha 策略都是 `obs[1,61] → actions[1,14]`，在行走、站立、捡球、踢球、坐下五种行为上验证过。因此只有**一种**布局，而非原型机携带的五种（51/54 维 legacy、49 维轮式、85 维跟踪都是 v1/v1.5 历史）。

### 观测布局总览

```text
index   width  contents
0..3        3  gyro, trunk frame, rad/s          —— 陀螺，躯干坐标系，rad/s
3..6        3  projected gravity, trunk frame    —— 投影重力，躯干坐标系，单位向量
6..20      14  joint position minus home pose    —— 关节位置减 home 位姿，不含嘴
20..34     14  joint velocity                     —— 关节速度，不含嘴
34..48     14  previous action                    —— 上一动作，不含嘴
48..61     13  command (below)                    —— 命令（见下）
```

命令块（没有第二份真值来源的部分）：

```text
48..51      3  vx, vy, vyaw
51..55      4  neck_pitch, head_pitch, head_yaw, head_roll
55..57      2  body x, y      — always zero, unbound in training  —— 恒零，训练未绑定
57          1  body z
58          1  body roll
59          1  body pitch
60          1  body yaw       — always zero, unbound in training  —— 恒零，训练未绑定
```

两个容易搞错的点（已对照 `microduck_runtime` 的 `control_step` 确认，非假设）：

1. **body x/y/yaw 硬编码为零**——它们在训练环境中未绑定，全零是*名义*编码，而非占位符。
2. **头目标在命令中，不在策略输出上叠加**——原型机在不同模式下两种做法都做过，并在事后叠加逻辑前用 `if !new_cmd_obs` 门控，注释"head_offsets are a COMMAND fed via the obs vector instead — don't double-add it here"。两种都做会让头弯曲两次。

另外注意 body 块内部顺序是 `z, roll, pitch`，**不是** `z, pitch, roll`。

## 关键常量

```rust
/// Total width of the observation. —— 观测总宽度。
pub const OBS_LEN: usize = 61;

/// Actions a policy returns — the 15 joints minus the mouth.
/// —— 策略返回的动作——15 关节减去嘴。
pub const ACTION_LEN: usize = 14;

/// Joints that appear in the observation: all but the mouth.
/// —— 出现在观测中的关节：除嘴外的所有。
pub const OBS_JOINTS: usize = NUM_JOINTS - 1;

/// Width of the trailing command block. —— 尾部命令块宽度。
pub const COMMAND_LEN: usize = 13;
```

## 关键结构体

### `Command`（第 66–74 行）

```rust
/// What a client is asking the robot to do, in the form the policy consumes.
/// —— 客户端要求机器人做什么，以策略消费的形式。
///
/// Held as physical units in the trunk frame — the conversion to the flat command block
/// happens in [`Observation::build`] and nowhere else.
/// —— 以躯干坐标系中的物理单位保存——到扁平命令块的转换
///    只在 [`Observation::build`] 中发生，不在别处。
#[derive(Debug, Clone, Copy, Default, PartialEq)]
pub struct Command {
    /// Forward, left, yaw-rate. —— 前向、左向、偏航速率。
    pub twist: [f64; 3],
    /// neck_pitch, head_pitch, head_yaw, head_roll. —— 颈俯仰、头俯仰、头偏航、头横滚。
    pub head: [f64; 4],
    /// Standing body pose: z, roll, pitch. Zero is the nominal stance.
    /// —— 站立身体位姿：z、roll、pitch。零为标准站姿。
    pub body: BodyPose,
}
```

—— 命令以**物理单位**保存，不直接是扁平数组。到 13 维命令块的转换集中在 `Observation::build` 一处，保证不会有第二处编码逻辑漂移。

```rust
impl Command {
    /// Magnitude of the velocity command, which is what selects walking versus standing.
    /// —— 速度命令的幅值，用于选择行走还是站立。
    pub fn twist_magnitude(&self) -> f64 {
        self.twist.iter().map(|v| v * v).sum::<f64>().sqrt()
    }
}
```

—— 行走 vs 站立由 twist 幅值决定，头和身体运动**不计入**——否则转头会让机器人误以为在行走。

### `BodyPose`（第 85–90 行）

```rust
/// Standing body pose offsets. Not commandable in slice 2 — carried so the layout is
/// complete and so the field exists when a `pose` intent lands.
/// —— 站立身体位姿偏移。在 slice 2 中不可命令——保留它是为了布局完整，
///    以及当 `pose` 意图到达时字段已存在。
#[derive(Debug, Clone, Copy, Default, PartialEq)]
pub struct BodyPose {
    pub z: f64,
    pub roll: f64,
    pub pitch: f64,
}
```

### `Observation`（第 133–136 行）

```rust
/// A built observation, ready to hand to the policy.
/// A fixed array rather than a `Vec`: it is rebuilt 50 times a second on a thread that
/// should not be visiting the allocator.
/// —— 已构建的观测，可直接交给策略。
///    用固定数组而非 `Vec`：它在一个每秒重建 50 次的线程上，
///    那个线程不应碰分配器。
#[derive(Debug, Clone, Copy)]
pub struct Observation {
    data: [f32; OBS_LEN],
}
```

—— 固定数组避免热路径上的堆分配。`f32` 是因为 ONNX 推理输入就是 `f32`。

## 关键函数

### `policy_joints`（第 100–105 行）

```rust
/// The joints a policy sees, in order, with the mouth skipped.
/// —— 策略看到的关节，按序排列，跳过嘴。
///
/// One definition, used for positions, velocities and the home pose alike — so those three
/// blocks cannot disagree about which joints they cover or what order they are in.
/// —— 一个定义，位置、速度和 home 位姿都用它——这样三个块
///    不可能在覆盖哪些关节或什么顺序上产生分歧。
///
/// Returns an array rather than an iterator so its width is part of its type.
/// A filtered iterator is not `ExactSizeIterator` — `Filter` cannot know how many elements
/// pass — and that is what would force [`fill`] to count at runtime instead of being
/// checked here.
/// —— 返回数组而非迭代器，使其宽度成为类型的一部分。
///    过滤迭代器不是 `ExactSizeIterator`——`Filter` 不知道有多少元素通过——
///    那会迫使 [`fill`] 在运行时计数，而不是在这里被编译器检查。
fn policy_joints(values: &[f64; NUM_JOINTS]) -> [f64; OBS_JOINTS] {
    // Skipping one valid index leaves exactly one fewer. Stated as a compile-time check
    // so the reasoning is enforced rather than merely believed.
    // —— 跳过一个有效索引恰好少一个。写成编译期检查，
    //    使推理被强制执行而非仅仅被相信。
    const { assert!(OBS_JOINTS == NUM_JOINTS - 1) };
    std::array::from_fn(|slot| values[joint_of(slot)])
}
```

—— 返回**定长数组**而非迭代器，宽度 `OBS_JOINTS` 编入类型。这使得下游的 `fill<const N>` 可以在编译期验证宽度匹配。

### `joint_of`（第 113–116 行）

```rust
/// The joint a policy slot refers to.
/// —— 策略槽位对应的关节。
///
/// Slots below the mouth map straight through; at and above, everything shifts up by one.
/// Written once and used in both directions — [`policy_joints`] reads through it,
/// [`Observation::scatter_action`] writes through it — so the two cannot disagree about
/// where the policy's n-th value belongs.
/// —— 嘴以下的槽位直通；嘴及以上，全部上移一位。
///    写一次，双向使用——[`policy_joints`] 通过它读，
///    [`Observation::scatter_action`] 通过它写——
///    两者不可能在策略第 n 个值属于哪个关节上产生分歧。
#[inline]
const fn joint_of(slot: usize) -> usize {
    if slot < MOUTH_INDEX { slot } else { slot + 1 }
}
```

—— 嘴索引（9）之前直通，之后 +1。这是**映射的唯一真相来源**，读和写都走它。

### `fill`（第 125–127 行）

```rust
/// Write one block, narrowing to `f32` on the way in.
/// —— 写入一个块，在写入时收窄为 `f32`。
///
/// Both sides are fixed-size arrays of the same `N`, so a block and its source cannot
/// disagree about width — the compiler rejects it. That matters more than it sounds:
/// `zip` stops at the shorter side, so a mismatch would otherwise leave the tail silently
/// at zero, and a zero in the observation is a joint sitting at its home pose.
/// The policy would act on a plausible robot that does not exist.
/// —— 两侧都是相同 `N` 的定长数组，块和其源不可能在宽度上分歧——编译器拒绝。
///    这比听起来重要：`zip` 在较短侧停止，宽度不匹配否则会静默地把尾部留零，
///    而观测中的零意味着关节停在 home 位姿。策略会基于一个不存在的、
///    看起来合理的机器人行动。
fn fill<const N: usize>(block: &mut [f32; N], values: [f64; N]) {
    *block = values.map(|value| value as f32);
}
```

—— `const N` 泛型使宽度不匹配在**编译期**报错，而非运行时静默留零。这防止了"观测尾部静默为零"这种灾难性错误。

### `Observation::build`（第 158–214 行）

```rust
/// Assemble the observation.
/// —— 组装观测。
///
/// `joint_positions` are absolute; the policy sees them relative to the home pose,
/// because that is what it was trained on.
/// —— `joint_positions` 是绝对的；策略看到的是相对 home 位姿的，
///    因为它就是这么训练的。
///
/// `last_action` is the previous *policy output* — raw, before action scaling —
/// in 14-wide policy order.
/// —— `last_action` 是上一*策略输出*——原始的、动作缩放前的——
///    按 14 宽策略顺序。
pub fn build(
    imu: &ImuData,
    joint_positions: &[f64; NUM_JOINTS],
    joint_velocities: &[f64; NUM_JOINTS],
    home_pose: &[f64; NUM_JOINTS],
    last_action: &[f32; ACTION_LEN],
    command: &Command,
) -> Self {
    let mut data = [0.0f32; OBS_LEN];

    // Carve the buffer into the blocks of the layout table above, by name — as *arrays*,
    // so each block's width is in its type and [`fill`] can only be handed a source of
    // matching length.
    // —— 按名称将缓冲区切分为上表中的块——作为*数组*，
    //    使每个块的宽度在其类型中，[`fill`] 只能收到长度匹配的源。
    let (gyro, rest) = data.split_first_chunk_mut::<3>().expect(LAYOUT);
    let (gravity, rest) = rest.split_first_chunk_mut::<3>().expect(LAYOUT);
    let (positions, rest) = rest.split_first_chunk_mut::<OBS_JOINTS>().expect(LAYOUT);
    let (velocities, rest) = rest.split_first_chunk_mut::<OBS_JOINTS>().expect(LAYOUT);
    let (previous_action, rest) = rest.split_first_chunk_mut::<OBS_JOINTS>().expect(LAYOUT);
    let (command_block, rest) = rest.split_first_chunk_mut::<COMMAND_LEN>().expect(LAYOUT);
    debug_assert!(rest.is_empty(), "{LAYOUT}");

    fill(gyro, imu.gyro);
    fill(gravity, imu.gravity);
    let angles = policy_joints(joint_positions);
    let home = policy_joints(home_pose);
    fill(positions, std::array::from_fn(|i| angles[i] - home[i]));  // 相对 home
    fill(velocities, policy_joints(joint_velocities));

    *previous_action = *last_action;  // 已是 f32，同宽，直接拷贝

    // 命令块：按文档表格顺序读取，可肉眼核对
    fill(
        command_block,
        [
            command.twist[0], command.twist[1], command.twist[2],
            command.head[0], command.head[1], command.head[2], command.head[3],
            0.0,  // body x — unbound in training  —— 训练未绑定
            0.0,  // body y — unbound              —— 训练未绑定
            command.body.z,
            command.body.roll,
            command.body.pitch,
            0.0,  // body yaw — unbound            —— 训练未绑定
        ],
    );

    Self { data }
}
```

—— 关键设计点：
- 用 `split_first_chunk_mut` 将缓冲区切分为**类型化的块**，每个块宽度由类型保证。
- 关节位置**减 home 位姿**——策略训练时看到的就是相对量。
- 上一动作直接拷贝（已是 `f32`，同宽）。
- 命令块按文档表格顺序排列，可肉眼核对。

### `Observation::scatter_action`（第 221–229 行）

```rust
/// Map a policy's 14 outputs onto the 15 joints, leaving the mouth untouched.
/// —— 将策略的 14 个输出映射到 15 个关节上，不动嘴。
///
/// The mouth is absent from every alpha policy, so its slot stays at whatever the caller
/// had. Getting this wrong shifts every joint after index 9 by one,
/// which is both catastrophic and completely silent.
/// —— 嘴在所有 alpha 策略中缺席，因此其槽位保持调用方原值。
///    搞错会把索引 9 之后的每个关节偏移一位，
///    既是灾难又是完全静默的。
pub fn scatter_action(action: &[f32; ACTION_LEN]) -> [f64; NUM_JOINTS] {
    let mut out = [0.0f64; NUM_JOINTS];
    for (slot, value) in action.iter().enumerate() {
        out[joint_of(slot)] = *value as f64;
    }
    out
}
```

—— `policy_joints` 的逆操作：通过同一个 `joint_of` 映射写回，嘴槽位保持为 0（由调用方负责嘴的独立控制）。

### `zeroed`（第 147–151 行）

```rust
/// An all-zero observation, for warming a session up before the control loop starts.
/// Not a valid robot state — it is only ever fed to an inference whose output is discarded,
/// to pay the first-call cost off the hot path.
/// —— 全零观测，用于在控制循环开始前预热会话。
///    不是有效的机器人状态——它只被喂给一个输出被丢弃的推理，
///    以在热路径之外支付首次调用的成本。
pub fn zeroed() -> Self {
    Self { data: [0.0; OBS_LEN] }
}
```

## 重要常量汇总

| 常量 | 值 | 含义 |
|------|-----|------|
| `OBS_LEN` | 61 | 观测向量总宽度 |
| `ACTION_LEN` | 14 | 策略输出动作维度（15 关节减嘴） |
| `OBS_JOINTS` | 14 | 观测中的关节数（= `NUM_JOINTS - 1`） |
| `COMMAND_LEN` | 13 | 尾部命令块宽度 |

## 测试要点（第 232–390 行）

### `the_layout_widths_sum_to_the_declared_input`

验证 `3 + 3 + 14*3 + 13 = 61`，且 `OBS_JOINTS == ACTION_LEN`。宽度不匹配会被 ONNX 运行时拒绝，但在这里失败远比在机器人上运行时失败好。

### `every_block_lands_at_its_documented_offset`

用可区分的值钉住全部六个边界：
- `d[0..3]` = 陀螺 `[1,2,3]`
- `d[3..6]` = 重力 `[4,5,6]`
- `d[6]` = 关节 0 相对 home 偏移 0.25
- `d[20]` = 关节速度块起点（0）
- `d[34]` = 上一动作第一元素 -0.5
- `d[47]` = 上一动作最后元素 0.75
- `d[48..51]` = twist `[0.1,0.2,0.3]`
- `d[51..55]` = 头 `[0.4,0.5,0.6,0.7]`

一个块移了位会表现为特定索引的错误值，而非"机器人走得差"。

### `unbound_body_axes_are_always_zero`

body x（55）、y（56）、yaw（60）无论调用方给什么都必须为零——否则策略看到一个它从未训练过的信号。

### `the_body_block_is_z_roll_pitch`

验证 body 块顺序是 `z(57), roll(58), pitch(59)`，**不是** `z, pitch, roll`。交换后两个会让机器人被要求前倾时向侧歪。

### `joint_positions_are_relative_to_the_home_pose`

home 位姿时 14 个位置槽位全为零。喂绝对角度会给 14 个输入叠加训练中从未见过的恒定偏移。

### `the_mouth_is_excluded_from_the_observation`

把嘴移一个很大的值（+1.0），观测中 6..20 区间不应有任何变化——嘴被排除在观测外。

### `scattering_an_action_skips_the_mouth`

用 `1,2,...,14` 填充 14 维动作，验证：
- 嘴槽位（9）保持 0
- 嘴之前的关节一对一映射（`scattered[0]=1, scattered[8]=9`）
- 嘴之后偏移一位（`scattered[10]=10, scattered[14]=14`）

### `twist_magnitude_ignores_head_and_body`

验证 `twist_magnitude` 只看 twist，头和身体运动不影响行走/站立判定。

## 与其他模块的关系

- **依赖 `model`**：`MOUTH_INDEX`、`NUM_JOINTS`、`DEFAULT_POSITION`。
- **依赖 `imu::ImuData`**：观测中的陀螺和重力来自 IMU。
- **被 `policy` 消费**：`Observation::build` 构建的 `&[f32]` 直接喂给 ONNX 推理；`scatter_action` 将推理结果映射回 15 关节。
- **被 `io`/控制循环调用**：每个 tick 调用 `Observation::build`，将传感器数据组装为策略输入。
- **被 `lib.rs` 重新导出**：`OBS_LEN`、`ACTION_LEN`、`Command`、`Observation` 是公共 API。
- **不依赖 `safety` 或 `fall`**：观测构建纯粹是数据组装，不涉及安全判定。
#（注：内容由AI生成）
