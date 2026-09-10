# `imu.rs` 解读

## 概述

`imu.rs`（356 行）负责解码 `imu_to_dxl` v2 板（LSM6DSV16X）通过 Dynamixel 总线吐出的 12 字节块。设计哲学是**一块板、一条代码路径**：IMU 板骑在 Dynamixel 总线上，它的 12 字节块与舵机在同一次 `sync_read` 中取到，因此主机上没有第二个传感器要轮询，也**不跑任何融合**——片上 SFLP 块直接吐游戏旋转四元数，陀螺零偏由芯片自己估计。

块布局（地址 124）：

| 字节 | 内容 |
|---|---|
| 0..6 | gyro x/y/z，`i16` 小端原始计数，±500 dps |
| 6..12 | SFLP 四元数 x/y/z，IEEE 半精度（binary16）；`w = √(1 − x² − y² − z²)` |

板子完整诊断块是 20 字节（还带原始加速度计、采样计数器、状态标志），控制循环只消费前 12 字节，好让这次读能和舵机塞进同一笔事务。

## 关键结构体 / 函数

### `ImuData`（第 24–45 行）

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct ImuData {
    /// Angular velocity in the trunk frame, rad/s. —— 躯干系下的角速度，rad/s
    pub gyro: [f64; 3],
    /// Projected gravity in the trunk frame, unit vector. Upright is `[0, 0, -1]`.
    /// 躯干系下的重力投影，单位向量。直立为 [0,0,-1]。
    /// This is what the policy observes, and what fall detection thresholds on.
    /// 这是策略观测的量，也是跌倒检测的阈值依据
    pub gravity: [f64; 3],
    /// Orientation, trunk→world, scalar-first `[w, x, y, z]`.
    /// 姿态：躯干→世界，scalar-first [w,x,y,z]
    pub quat: [f64; 4],
}

impl Default for ImuData {
    fn default() -> Self {
        Self {
            gyro: [0.0; 3],
            gravity: [0.0, 0.0, -1.0],   // 默认直立
            quat: [1.0, 0.0, 0.0, 0.0],  // 单位四元数（identity）
        }
    }
}
```

—— `gravity` 默认直立、`quat` 默认 identity。但注意：这个默认值只在"还没收到有效样本"时出现，**绝不能**把 identity 当成"机器人确实直立"来上报——这是跌倒检测最不能接受的谎言（见下文 `SflpDecoder` 的 `last_quat` 保持逻辑）。

### `SflpDecoder`（第 50–157 行）

```rust
pub struct SflpDecoder {
    mount: [f64; 4],                 // 传感器→躯干安装旋转（scalar-first）
    /// Last quaternion the board actually produced. Held across blocks that arrive before
    /// SFLP has written its table — snapping to identity would report a robot as upright
    /// when its orientation is simply unknown, which is the worst possible lie to tell
    /// fall detection.
    //  板子真正产出过的最后一个四元数。SFLP 表还没写好的块到达时继续保持它；
    //  直接弹回 identity 会在"姿态根本未知"时报成"机器人直立"，
    //  这是能对跌倒检测撒的最糟的谎
    last_quat: [f64; 4],
    /// Blocks carrying a live quaternion. Gates `ready`. —— 带活四元数的块数，门控 ready()
    quat_samples: u32,
    gyro_history: [[f64; 3]; 2],     // 中值滤波用的前两个陀螺样本
    gravity_history: [[f64; 3]; 2],   // 中值滤波用的前两个重力样本
}
```

—— 解码器**有状态，但状态只服务两件事**：尖峰抑制（`*_history`）和保持最后一个好四元数（`last_quat`）。这里**没有滤波器**——融合在芯片里做完了。把状态显式保留是为了让 `crate::io::FakeIo` 能在测试里驱动同一个解码器。

#### `DEFAULT_MOUNT` 与 `new`（第 75–93 行）

```rust
/// The board is mounted so that trunk = `[+raw_z, +raw_y, −raw_x]`, a +90° rotation
/// about Y.
//  板子安装方向使得 躯干 = [+raw_z, +raw_y, −raw_x]，即绕 Y 轴 +90°
pub const DEFAULT_MOUNT: [f64; 4] = [
    std::f64::consts::FRAC_1_SQRT_2,  // w = 1/√2
    0.0,
    std::f64::consts::FRAC_1_SQRT_2,  // y = 1/√2
    0.0,
];
```

—— 物理安装决定了 raw 坐标系与躯干坐标系差一个绕 Y 轴 +90° 的旋转。陀螺用 `mount` 正变换到躯干系；四元数因为是"传感器系→世界"，要乘 `mount_inv`（共轭）才变成"躯干系→世界"。

#### `ready`（第 95–101 行）

```rust
pub fn ready(&self) -> bool {
    self.quat_samples >= 25   // 约 0.25 s @ 100 Hz
}
```

—— 直到芯片**确实产出过 25 个融合四元数**之前，姿态都是默认值而非测量值。slice 2 的跌倒检测在此之前**不得运行**。若从第一个块就 ready，机器人会在最初四分之一秒里被按默认姿态评判。

#### `decode`（第 103–156 行）

```rust
pub fn decode(&mut self, raw: &[u8; IMU_BLOCK_LEN]) -> ImuData {
    // 1) 陀螺：i16 LE × GYRO_RAD_PER_LSB，再按 mount 转到躯干系
    let gyro_sensor = [
        i16::from_le_bytes([raw[0], raw[1]]) as f64 * GYRO_RAD_PER_LSB,
        i16::from_le_bytes([raw[2], raw[3]]) as f64 * GYRO_RAD_PER_LSB,
        i16::from_le_bytes([raw[4], raw[5]]) as f64 * GYRO_RAD_PER_LSB,
    ];
    let gyro = rotate(self.mount, gyro_sensor);

    // 2) 四元数：全零字节 = SFLP 表还没写，保持 last_quat
    let packed = [
        u16::from_le_bytes([raw[6], raw[7]]),
        u16::from_le_bytes([raw[8], raw[9]]),
        u16::from_le_bytes([raw[10], raw[11]]),
    ];
    if packed != [0, 0, 0] {
        let (x, y, z) = (half(packed[0]), half(packed[1]), half(packed[2]));
        let norm_sq = x * x + y * y + z * z;
        // 芯片没产出过的四元数通不过这关；≤1.02 容忍满量程半精度舍入
        if x.is_finite() && y.is_finite() && z.is_finite() && norm_sq <= 1.02 {
            let w = (1.0 - norm_sq).max(0.0).sqrt();
            let mount_inv = [self.mount[0], -self.mount[1],
                             -self.mount[2], -self.mount[3]];
            let q = mul([w, x, y, z], mount_inv);
            let norm = (q[0]*q[0] + q[1]*q[1] + q[2]*q[2] + q[3]*q[3]).sqrt();
            if norm > 0.5 {
                self.last_quat = [q[0]/norm, q[1]/norm, q[2]/norm, q[3]/norm];
                self.quat_samples = self.quat_samples.saturating_add(1);
            }
        }
    }

    // 3) 重力 = 用 last_quat 反投影世界系 [0,0,-1]，再归一化
    let gravity = normalise(rotate_inverse(self.last_quat, [0.0, 0.0, -1.0]));

    // 4) 三轴分别做三样本中值滤波，抑制单尖峰
    let out = ImuData {
        gyro: median3_each(&self.gyro_history, gyro),
        gravity: median3_each(&self.gravity_history, gravity),
        quat: self.last_quat,
    };
    self.gyro_history = [self.gyro_history[1], gyro];
    self.gravity_history = [self.gravity_history[1], gravity];
    out
}
```

几个关键设计：

- **全零四元数字节不刷新 `last_quat`、也不计入 `quat_samples`**。板子刚上电或初始化失败时 SFLP 表是空的，会持续吐全零；此时保持最后一个好值，而不是弹回 identity（那会谎称机器人直立）。
- **`norm_sq <= 1.02` 而非 `<= 1.0`**：容忍满量程处的半精度舍入误差。`w` 用 `(1−norm_sq).max(0.0).sqrt()` 兜底，避免数值微负开方出 NaN。
- **`norm > 0.5` 才接受**：拒绝退化到近零的四元数更新，避免把坏样本推进 `last_quat`。
- **重力先归一化再中值**（与运行时一致）：注释明确指出后果——三个单位向量按分量取中值后**不一定仍是单位长度**，瞬态期间策略会看到略短的向量；稳态是精确的。是否要"中值后再归一化"留待 slice 2 接观测时再定，因为那会改变当前正在走路的行为。

### `median3_each`（第 159–168 行）

```rust
fn median3_each(history: &[[f64; 3]; 2], now: [f64; 3]) -> [f64; 3] {
    // 单样本尖峰抑制：掉块或坏块表现为一个野值；三样本中值丢掉它，
    // 又不像平均那样带滞后
    let m = |a: f64, b: f64, c: f64| a.max(b).min(c).max(a.min(b));
    [ m(history[0][0], history[1][0], now[0]),
      m(history[0][1], history[1][1], now[1]),
      m(history[0][2], history[1][2], now[2]) ]
}
```

—— 对 gyro 与 gravity 的**每个分量独立**取三样本中值（前两历史 + 当前）。解析式 `a.max(b).min(c).max(a.min(b))` 是三数中值的纯表达式写法，无分支。

### `half`（第 170–181 行）

```rust
/// IEEE 754 binary16 → f64. The LSM6DSV16X ships quaternion components in this format.
//  手写 IEEE 754 binary16 → f64。LSM6DSV16X 的四元数分量就是这种格式
fn half(bits: u16) -> f64 {
    let sign = if bits & 0x8000 != 0 { -1.0 } else { 1.0 };
    let exp = ((bits >> 10) & 0x1F) as i32;
    let frac = (bits & 0x3FF) as f64;
    match exp {
        0 => sign * frac * 2.0_f64.powi(-24),                 // 非规格化数
        0x1F if frac == 0.0 => sign * f64::INFINITY,          // ±∞
        0x1F => f64::NAN,                                      // NaN
        _ => sign * (1.0 + frac / 1024.0) * 2.0_f64.powi(exp - 15), // 规格化
    }
}
```

—— 为什么要手写而不用库？crate 不引入额外依赖，且 binary16 解码逻辑很短。**指数偏置错一位，地平线就会悄悄倾斜**，因此有专门测试钉住 0、1、−1、0.5、1/3 等已知值。

### 四元数与旋转辅助函数（第 183–237 行）

```rust
fn mul(a: [f64; 4], b: [f64; 4]) -> [f64; 4]   // Hamilton 积，scalar-first
fn rotate(q: [f64; 4], v: [f64; 3]) -> [f64; 3]         // q·v·q⁻¹
fn rotate_inverse(q: [f64; 4], v: [f64; 3]) -> [f64; 3]  // q⁻¹·v·q：世界向量转到本体系
fn normalise(v: [f64; 3]) -> [f64; 3] {
    let mag = /* ... */;
    if mag > 0.1 { v/mag } else { [0.0, 0.0, -1.0] }  // 退化时回退"直立"，而非除零
}
```

—— `rotate_inverse` 把世界系下的重力方向 `[0,0,−1]` 投影到躯干系，得到策略直接观测的 `gravity`。`normalise` 的 `mag > 0.1` 阈值防止近零除零，退化时回退直立向量而非 NaN。

## 重要常量

| 常量 | 值 | 含义 |
|---|---|---|
| `IMU_BLOCK_LEN` | `12` | 每 tick 消费的 v2 板块字节数（与 `bus.rs` 的 `READ_LEN` 相等） |
| `GYRO_RAD_PER_LSB` | `0.0175·π/180` | ±500 dps 量程、17.5 mdps/LSB，换算成 rad/s |
| `DEFAULT_MOUNT` | `[1/√2, 0, 1/√2, 0]` | 传感器→躯干安装旋转，绕 Y 轴 +90°，躯干 = `[+raw_z, +raw_y, −raw_x]` |
| `ready()` 阈值 | `quat_samples >= 25` | 约 0.25 s @ 100 Hz，与 `bus.rs` 的 `STALE_RUN_WARN=25` 同档 |

## 测试要点（第 243–355 行）

### `half_precision_decodes_known_values`

钉住手写 half 解码的关键用例：0（0x0000）、1（0x3C00）、−1（0xBC00）、0.5（0x3800）、1/3 附近（0x3555≈0.333）。指数偏置错了会静默倾斜地平线。

### `all_zero_quaternion_bytes_hold_the_last_good_value`

全零块 = "SFLP 还没启动"。验证：解码全零块不把 `last_quat` 重置为 identity，也不计入 `quat_samples`；先喂一个真实小旋转（x=0.125），再喂全零块，输出必须**保持**刚才那个四元数而不是弹回 identity。这是跌倒检测安全语义的核心。

### `not_ready_until_the_chip_has_produced_output`

连喂 24 个活块后 `ready()` 仍为 false，第 25 个才 true。门控 slice 2 的跌倒检测不能提前开启。

### `gravity_is_a_unit_vector_in_steady_state`

四个不同朝向各连喂三个相同块（让中值稳定），验证稳态下重力向量模长与 1 的偏差 < 1e-9。策略直接观测它，且训练时喂的是归一化输入。

### `gravity_stays_close_to_unit_through_a_transient`

相邻 tick 之间切换朝向：三个不同单位向量按分量中值会把结果压短。钉住这个瞬态边界（>0.5 且 ≤1.0+ε），免得未来有人"修"出一个训练时没见过的长度。

### `gyro_counts_are_signed`

把陀螺 x 轴写成 `−1000`，验证输出是负转速而不是巨大正转速。无符号误读会把每个负角速度变成大正数，看起来像机器人在狂转。

## 与其他模块的关系

- **被 `crate::bus` 使用**：`DynamixelIo` 在 `read()` 中把槽 0 的 12 字节交给 `SflpDecoder::decode`，并靠 `ready()` 实现 `RobotIo::imu_ready`。
- **被 `crate::lib` 重新导出**：`ImuData` 是 crate 公共 API 的一部分，`Sensors.imu` 的类型。
- **依赖 `crate::model::NUM_JOINTS`**：仅用于编译期断言 `NUM_JOINTS > 0`——IMU 块是定长的，与总线上挂多少舵机无关。
- **被 `crate::io` 的 `FakeIo` 驱动**：状态显式保留是为了测试替身能复现真实解码器行为。
- **被 `crate::obs` 与 `crate::fall` 消费**：`gravity` 进观测向量并作为跌倒检测阈值依据；`quat` 提供完整姿态。
- **数值口径对齐硬件**：`GYRO_RAD_PER_LSB`、`DEFAULT_MOUNT`、`IMU_BLOCK_LEN` 都对应 LSM6DSV16X v2 板的实际接线与量程。
#（注：内容由AI生成）
