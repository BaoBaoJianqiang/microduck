# imu.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\imu.rs`

## 核心设计决策

`imu.rs` 解码 `imu_to_dxl` v2 板（LSM6DSV16X）的数据。该板位于 Dynamixel 总线上，其 12 字节块与舵机在同一个 `sync_read` 中获取，因此无需轮询独立传感器，也无需在主机上运行融合——芯片的 SFLP 模块直接输出游戏旋转四元数并自估计陀螺偏置。

控制循环仅消费前 12 字节（完整诊断块为 20 字节，还含原始加速度计、采样计数器、状态标志），以便读取能与舵机在一个事务内完成。

## 数据块布局（地址 124）

| 字节 | 内容 |
|---|---|
| 0..6 | 陀螺 x/y/z，`i16` LE 原始计数，±500 dps |
| 6..12 | SFLP 四元数 x/y/z，IEEE 半精度；`w = √(1 − x² − y² − z²)` |

## 类型与常量

- `IMU_BLOCK_LEN = 12` — 每 tick 消费的块字节数
- `ImuData`：`gyro: [f64; 3]`（角速度）、`gravity: [f64; 3]`（投影重力，正立为 `[0,0,-1]`）、`quat: [f64; 4]`（姿态，标量在前 `[w,x,y,z]`）
- `GYRO_RAD_PER_LSB` — ±500 dps 下 17.5 mdps/LSB，转 rad/s

## `SflpDecoder`

有状态解码器，状态仅用于尖峰抑制和保存上一个有效四元数，**不包含滤波器**。

### 关键方法
- `new(mount)` — 安装旋转为 `[+raw_z, +raw_y, −raw_x]`（绕 Y 轴 +90°）
- `ready() -> bool` — 芯片是否已产出融合输出（约 25 个样本，0.25s@100Hz）。在此之前姿态是默认值而非测量值，slice 2 的跌倒检测必须等其就绪
- `decode(raw) -> ImuData` — 解码块：
  - 陀螺：LE i16 × `GYRO_RAD_PER_LSB`，再经安装旋转
  - 四元数：全零块表示 SFLP 尚未写入表（板刚上电或初始化失败），保持上一个有效值而非跳回单位四元数（跳回单位会把姿态未知的机器人报告为正立，对跌倒检测是最坏的谎言）
  - 半精度解码（`half`），`norm_sq ≤ 1.02` 允许满量程半精度舍入
  - 重力 = 归一化 `rotate_inverse(last_quat, [0,0,-1])`，在中位滤波**之前**归一化（与运行时一致）
  - 陀螺和重力各做 3 点逐分量中位滤波（`median3_each`），丢弃单个野值而不带均值的滞后

### 辅助函数
- `half(bits: u16) -> f64` — IEEE 754 binary16 解码
- `mul(a, b)` — Hamilton 乘积
- `rotate(q, v)` / `rotate_inverse(q, v)` — 四元数旋转
- `normalise(v)` — 单位化，近零时回退 `[0,0,-1]`

## 单元测试描述

- `half_precision_decodes_known_values`：半精度解码 0、1、-1、0.5、0.333 等已知值。
- `all_zero_quaternion_bytes_hold_the_last_good_value`：全零块保持上一个有效四元数，不重置为单位；且不计入活样本。
- `not_ready_until_the_chip_has_produced_output`：25 个样本前 `ready()` 为 false。
- `gravity_is_a_unit_vector_in_steady_state`：稳态下重力为单位向量。
- `gravity_stays_close_to_unit_through_a_transient`：瞬态中重力模长不崩溃（>0.5 且 ≤1.0）。
- `gyro_counts_are_signed`：陀螺计数有符号，-1000 计数应得到负角速度。

## 关键摘要

`imu.rs` 是对 `imu_to_dxl` v2 板 12 字节块的解码层。关键设计：不做主机端融合（用芯片 SFLP 输出）、全零块保持上一有效值而非跳回正立、3 点中位滤波去尖峰、`ready()` 门控确保姿态收敛前不被用作测量。所有数值与布局与 `microduck_runtime` 对齐。
