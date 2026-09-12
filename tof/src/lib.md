# lib.rs 文件解析

## 文件位置

`d:\microduck\tof\src\lib.rs`

## 核心设计决策

1. **头部 ToF 传感器的帧形状与区带语义定义层。** 本模块只定义数据类型：8×8 距离矩阵 + ST 的逐区状态。它不做重投影——把 zone 转换到机器人坐标系需要头部正运动学，本守护进程不具备也不伪造。需要几何的消费者（建图、避障）会把 `tof.frame` 与关节状态在运动学到达时合并；只需要"看"传感器的消费者（`robotctl monitor`）则完全不需要。

2. **状态字节是真相，不是噪声。** ST 的 status 是"什么都没有"与"我读不出来"之间的区分。如果把它们折叠（例如把非有效状态都当作无回波），地图会失去最关键的信息：空旷空间是有意义的信息，不可用的测量不是。

3. **解析时的防御性。** `distance_mm` 与 `status` 都是 `Vec`，跨版本的对端可能发送更短的帧。`zone()` 用 `.get()` + `unwrap_or` 处理越界与缺失状态，绝不 panic；缺失的状态按 `STATUS_NO_TARGET` 解读。

## 常量

| 常量 | 值 | 含义 |
|---|---|---|
| `ROWS` | 8 | 行数，被 `start` 配置且为线格式承载 |
| `COLS` | 8 | 列数 |
| `ZONES` | 64 | `ROWS * COLS` |
| `STATUS_VALID` | `[5, 9]` | ST 文档认为可用的状态码：5=有效，9=有效但大脉冲（约 50% 置信度） |
| `STATUS_NO_TARGET` | 255 | "测了但没有东西在量程内" |

## 类型

### `Frame`

```rust
pub struct Frame {
    pub rows: u8,
    pub cols: u8,
    pub distance_mm: Vec<i16>,
    pub status: Vec<u8>,
}
```

行优先的并行数组，长度均为 `ZONES`。一个距离只在状态说有效时才有意义。

### `Zone`

```rust
pub enum Zone {
    Range(f32),     // 状态 5 或 9，米
    NoTarget,       // 状态 255：空旷空间
    Unusable(u8),   // 其他状态：测量失败，保留原始码供查 ST 表
}
```

## 函数

### `Frame::zone(index) -> Zone`

按状态解读距离：

- 状态在 `STATUS_VALID` 且 `distance > 0` → `Range(distance / 1000.0)`（米）
- 状态有效但距离非正（收敛失败偶发）→ `Unusable(status)`
- 状态 255 → `NoTarget`
- 其他 → `Unusable(status)`

### `Frame::zones() -> impl Iterator<Item = Zone>`

按行优先遍历所有区带。

### `Frame::valid_count() -> usize`

统计 `Range` 区带数——"传感器到底看到东西了没有"的单一数字。

## 单元测试

- `status_decides_what_a_distance_means` — 验证三类区带可区分，特别是"无目标"与"不可用"在仅距离视图下看起来相同但对地图意义相反；以及"有效状态+负距离"必须判定为不可用。
- `valid_count_counts_ranges_only` — 仅 `Range` 计入计数。
- `a_ragged_frame_reads_as_no_target` — 长度不匹配的帧不 panic，缺失状态/越界按 `NoTarget` 解读。

## 关键摘要

lib.rs 是 ToF 数据类型与语义层，核心贡献是把 ST 状态字节提升为一等公民：`Range`/`NoTarget`/`Unusable` 三态区分对建图至关重要。帧用 `Vec` 承载并在解读时全路径防御，保证跨版本的短帧不会让消费者崩溃。
