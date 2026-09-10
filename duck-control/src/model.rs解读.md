# `model.rs` 解读

## 概述

`model.rs`（226 行）是 `duck-control` crate 的**机器人数据模型层**——把"这台机器人是什么"编码为一组常量和纯函数。核心设计意图：

- **单一变体 alpha**：只存在一台机器人，所有发布的策略都是 `alpha_*`；v1/v1.5/v1.6 都是历史。第二个修订版本会变成第二组表——诚实地说，在有第二台机器人可以泛化之前，不值得为此抽象。
- **数值来自硬件实测**：这里的数字从 `microduck_runtime` 的 `motor.rs` 搬来，是对着硬件测量出来的，不是从手册推导的。从手册重新推导正是那种"看起来对、走起来错"的改动。

文件涵盖：关节数量与 ID 映射、home 位姿、嘴部行程、IMU 总线参数、EEPROM 寄存器断言、电池电压映射。

## 关键常量与函数

### 关节拓扑

```rust
/// Left leg (5) · neck/head/mouth (5) · right leg (5).
/// —— 左腿(5) · 颈/头/嘴(5) · 右腿(5)。
pub const NUM_JOINTS: usize = 15;
```

```rust
/// Dynamixel IDs, indexed as [`JOINT_NAMES`].
/// —— Dynamixel ID，与 [`JOINT_NAMES`] 同序索引。
pub const JOINT_IDS: [u8; NUM_JOINTS] = [
    20, 21, 22, 23, 24, // left leg   —— 左腿
    30, 31, 32, 33, 34, // neck, head, mouth —— 颈、头、嘴
    10, 11, 12, 13, 14, // right leg  —— 右腿
];
```

—— 关节分三组：左腿 ID 20–24，颈/头/嘴 ID 30–34，右腿 ID 10–14。注意右腿 ID 比左腿小——这是 Dynamixel 总线的物理布线顺序决定的。

```rust
/// Joint names, from the protocol crate — the wire indexes `joints` and `targets`
/// positionally, so that order and this one cannot be allowed to drift apart.
/// The assertion below is what makes "cannot" true.
/// —— 关节名称，来自协议 crate——线上按位置索引 `joints` 和 `targets`，
///    因此线上顺序和这个顺序不能允许漂移。下面的断言使"不能"成真。
pub use duck_ipc_proto::JOINT_NAMES;

const _: () = assert!(JOINT_NAMES.len() == NUM_JOINTS);
```

—— `JOINT_NAMES` 从外部协议 crate 重新导出。编译期断言保证名称表长度与 `NUM_JOINTS` 一致，防止两张表静默漂移。

### `MOUTH_INDEX`（第 31 行）

```rust
/// The mouth is absent from every alpha policy — they are all 61-D observation, 14-action,
/// and the action vector skips this index. Named so that omission is deliberate rather
/// than an off-by-one someone has to rediscover.
/// —— 嘴在所有 alpha 策略中缺席——它们都是 61 维观测、14 维动作，
///    动作向量跳过这个索引。命名它是为了让这个遗漏是刻意的，
///    而不是某个后来者不得不重新发现的 off-by-one。
pub const MOUTH_INDEX: usize = 9;
```

—— 嘴关节（索引 9）不在任何策略中。14 维动作向量跳过此索引，15 个关节中有一个不被策略驱动。命名为常量使跳过行为显式化。

### `DEFAULT_POSITION`（第 39–55 行）

```rust
/// Home pose. The trunk sits ~5 mm further forward than the v1.5 pose
/// so the CoM is over the ankle axis; the old pose biased the robot backwards.
/// —— Home 位姿。躯干比 v1.5 位姿前移约 5mm，使重心投影落在踝关节轴上方；
///    旧位姿使机器人向后偏。
///
/// Must match `HOME_FRAME` in the training env — a policy is trained against these angles
/// and observes joint positions *relative* to them, so a discrepancy here is a constant
/// offset on 14 observation slots.
/// —— 必须与训练环境中的 `HOME_FRAME` 匹配——策略针对这些角度训练，
///    并*相对*它们观测关节位置，因此这里的不一致会在 14 个观测槽位上
///    叠加一个恒定偏移。
pub const DEFAULT_POSITION: [f64; NUM_JOINTS] = [
    0.0,     // left_hip_yaw
    -0.0873, // left_hip_roll     —— -5°
    -0.4579, // left_hip_pitch    —— ~-26.2°
    -0.0049, // left_knee
    0.4530,  // left_ankle
    0.3491,  // neck_pitch        —— ~20°
    0.3491,  // head_pitch        —— ~20°
    0.0,     // head_yaw
    0.0,     // head_roll
    0.0,     // mouth
    0.0,     // right_hip_yaw
    0.0873,  // right_hip_roll    —— +5°（与左腿镜像）
    0.4579,  // right_hip_pitch   —— +26.2°（与左腿镜像）
    0.0049,  // right_knee        —— 与左腿镜像
    -0.4530, // right_ankle       —— 与左腿镜像
];
```

—— 这是 home 位姿。左右腿的 roll/pitch/ankle 严格等大反向（镜像对称）。躯干前移 ~5mm 是为了让重心落在踝关节轴上方。**与训练环境的 `HOME_FRAME` 必须严格一致**，否则 14 个观测输入全部带上恒定偏移。

### 嘴部控制

```rust
/// Mouth travel, radians: closed and fully open. The alpha reuses the v1.6 range,
/// −5°..+30°, from `microduck_runtime`'s `variant.rs`.
/// —— 嘴行程，弧度：闭合和完全张开。alpha 复用 v1.6 的范围 −5°..+30°。
pub const MOUTH_CLOSED: f64 = -5.0 * std::f64::consts::PI / 180.0;
pub const MOUTH_OPEN: f64 = 30.0 * std::f64::consts::PI / 180.0;
```

```rust
/// Joint angle for a mouth opening fraction. 0 is closed, 1 is fully open;
/// anything outside is clamped rather than fed to a servo as an out-of-travel target.
/// —— 嘴开度分数对应的关节角度。0 闭合，1 完全张开；
///    超出范围的值会被钳位，而非作为超行程目标发给舵机。
pub fn mouth_target(open: f64) -> f64 {
    let open = if open.is_finite() {
        open.clamp(0.0, 1.0)
    } else {
        0.0
    };
    MOUTH_CLOSED + open * (MOUTH_OPEN - MOUTH_CLOSED)
}
```

—— 嘴不在任何策略中，所以这两个常量加 `mouth_target` 函数就是嘴部控制的全部。非有限值（NaN）安全地映射到闭合。

### 总线与 IMU

```rust
/// The `imu_to_dxl` v2 board's Dynamixel ID.
/// It rides the motor bus and is read in the same transaction as the servos.
/// —— `imu_to_dxl` v2 板卡的 Dynamixel ID。它挂在电机总线上，
///    与舵机在同一次事务中被读取。
pub const IMU_DXL_ID: u8 = 200;

pub const BAUD_RATE: u32 = 1_000_000;
```

—— IMU 板共享舵机总线，ID 200 必须不与任何关节 ID 冲突。波特率 1 Mbps。

### `EXPECTED_REGISTERS`（第 89–94 行）

```rust
/// EEPROM registers asserted (and corrected) at startup.
/// —— 启动时断言（并纠正）的 EEPROM 寄存器。
///
/// `return_delay_time` is the load-bearing one: the XL330 ships at 250, which is 500 µs
/// of turnaround *per device*. Across 16 devices that is 8 ms per tick — 40% of a 20 ms
/// budget — spent waiting for servos to get around to answering.
/// —— `return_delay_time` 是承重的那个：XL330 出厂值为 250，即*每设备* 500µs 周转。
///    16 个设备就是每 tick 8ms——20ms 预算的 40%——花在等舵机应答上。
pub const EXPECTED_REGISTERS: &[(&str, u8)] = &[
    ("return_delay_time", 0),
    ("baud_rate", 3), // 3 = 1 Mbps, must agree with BAUD_RATE
    ("pwm_slope", 255),
    ("shutdown", 52),
];
```

—— 启动时断言并纠正的 EEPROM 寄存器。`return_delay_time = 0` 是最关键的：XL330 出厂值 250 导致每设备 500µs 周转，16 个设备累计 8ms/tick（20ms 预算的 40%）。`shutdown = 52` 是过载、过温、输入电压故障的锁存错误掩码。

### `joint_index`（第 97–99 行）

```rust
/// Index of a joint by name. Linear scan over 15 entries, used at startup and in tests.
/// —— 按名称查关节索引。对 15 个条目线性扫描，在启动和测试中使用。
pub fn joint_index(name: &str) -> Option<usize> {
    JOINT_NAMES.iter().position(|n| *n == name)
}
```

—— 15 个条目线性扫描足够，启动时调用一次，不影响热路径。

### 电池

```rust
/// Off a full charge, under load. NP-F550, 2S Li-ion.
/// —— 满电后，负载下的电压。NP-F550，2S 锂离子。
pub const BATTERY_FULL_V: f64 = 8.2;

/// The sag floor: below this the robot starts struggling, well before the pack's own
/// protection trips. Empty for our purposes, not empty for the cells'.
/// —— 塌陷下限：低于此值机器人开始吃力，远在电池自身保护触发之前。
///    对我们来说是空了，对电芯来说不是。
pub const BATTERY_EMPTY_V: f64 = 6.6;
```

—— 没有电量计、没有 ADC。唯一可用的测量是舵机自报的供电电压——总线看到的电池包电压，**负载下会塌陷，静止时恢复**。因此这个电压区间是**负载下可用范围**，而非电芯化学范围。

```rust
/// Fraction of a pack, 0–100, for a bus voltage.
/// Linear, and the numbers come from `microduck_runtime`'s `check_battery`,
/// where they were arrived at by running a duck flat.
/// —— 总线电压对应的电池电量百分比 0–100。
///    线性映射，数字来自 `microduck_runtime` 的 `check_battery`，
///    是把一只鸭子跑到没电跑出来的。
///
/// A non-finite or non-positive reading is 0 — those mean "no answer from the bus",
/// and the caller is expected to report that as unknown rather than to display this number.
/// —— 非有限或非正读数返回 0——那些意味着"总线无应答"，
///    调用方应将其报告为未知，而非显示这个数字。
pub fn battery_percent(volts: f64) -> f64 {
    if !volts.is_finite() || volts <= 0.0 {
        return 0.0;
    }
    ((volts - BATTERY_EMPTY_V) / (BATTERY_FULL_V - BATTERY_EMPTY_V)).clamp(0.0, 1.0) * 100.0
}
```

—— 线性映射，单一真值来源放在这里（而非散落在 CLI 和应用中），避免两个屏幕对同一块电池显示不同电量。非有限/非正读数安全返回 0。

## 重要常量汇总

| 常量 | 值 | 含义 |
|------|-----|------|
| `NUM_JOINTS` | 15 | 关节总数（左腿5 + 颈/头/嘴5 + 右腿5） |
| `MOUTH_INDEX` | 9 | 嘴关节在 15 关节数组中的索引（策略跳过） |
| `MOUTH_CLOSED` | -5° | 嘴闭合角度 |
| `MOUTH_OPEN` | +30° | 嘴完全张开角度 |
| `IMU_DXL_ID` | 200 | IMU 板 Dynamixel ID（与关节共享总线） |
| `BAUD_RATE` | 1,000,000 | 总线波特率 1 Mbps |
| `BATTERY_FULL_V` | 8.2 V | 负载下满电电压 |
| `BATTERY_EMPTY_V` | 6.6 V | 负载下空电（开始吃力）电压 |

## 测试要点（第 131–225 行）

### `tables_agree_on_length`

`JOINT_IDS`、`JOINT_NAMES`、`DEFAULT_POSITION` 三张表在整个 crate 中按同一个整数索引。如果长度不一致，每次查找都会静默读错关节。

### `ids_are_unique`

重复的 Dynamixel ID 会导致 `sync_read` 返回的块无法对应回关节，`sync_write` 会同时命令两个关节——两者故障表现都像接线问题。

### `imu_id_does_not_collide_with_a_joint`

IMU 板与舵机共享总线，其 ID 不能与任何关节冲突。

### `mouth_index_names_the_mouth`

`MOUTH_INDEX` 用于在 14→15 映射时跳过一个槽位。指错关节会把其后的每个动作偏移一位。

### `battery_percent_spans_the_usable_range` / `clamps_rather_than_extrapolating` / `treats_no_reading_as_zero`

验证电池映射的端点、中点、钳位行为、以及无读数（0V、NaN、负值）安全返回 0。

### `mouth_target_spans_the_prototype_range`

验证嘴范围端点正确，超范围/NaN 安全钳位。

### `home_pose_legs_are_mirrored`

左右腿的 roll/pitch/ankle 四对角度必须等大反向。home 位姿中的符号笔误靠肉眼检查不可见，会让机器人站歪。

## 与其他模块的关系

- **被几乎所有模块依赖**：`NUM_JOINTS`、`JOINT_IDS`、`DEFAULT_POSITION` 是整个 crate 的物理基础常量。
- **被 `obs` 消费**：`OBS_LEN` 布局中的关节数、`MOUTH_INDEX` 的跳过逻辑都源自本模块。
- **被 `bus` 消费**：`JOINT_IDS` 用于 `sync_read`/`sync_write` 的总线事务；`EXPECTED_REGISTERS` 用于启动时校验；`IMU_DXL_ID` 区分 IMU 板与舵机。
- **被 `io`/`safety` 消费**：`NUM_JOINTS` 决定 `Sensors`/`JointTargets` 数组宽度。
- **被 `policy` 消费**：`DEFAULT_POSITION` 作为观测中的 home 参考点。
- **被 `lib.rs` 重新导出**：核心常量是 crate 公共 API 的一部分。
#（注：内容由AI生成）
