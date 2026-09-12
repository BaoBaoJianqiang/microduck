# sensor.rs 文件解析

## 文件位置

`d:\microduck\tof\src\sensor.rs`

## 核心设计决策

1. **安全 Rust 包装 vendored ULD。** 每个调用都是一次 FFI 进入 `vendor/*/shim.c`，其表面全是标量与两个 64 项数组。因此 `unsafe` 全部集中在本文件且形态一致——"C 往我开的 64 大小缓冲区写 64 项"——而非散布在手写的 `repr(C)` 结构体镜像中。

2. **两代传感器，同一接口。** VL53L5CX 与 VL53L8CX 是同封装、同寄存器映射、同 8×8 输出，仅固件与驱动前缀不同。`Generation` 在上传任何固件之前由 ID 读取决定，之后所有调用按代分派到对应 shim。上层无需知道装的是哪个——但下层全部上报它，因为"这只鸭子是哪颗传感器"是深度看起来不对时的第一问。

3. **每进程单实例，强制。** 每个 shim 把配置存在文件作用域静态变量中（见其头文件），第二个 `Sensor` 会静默共享并破坏第一个的状态。`Sensor::open` 通过 `TAKEN` 原子标志拒绝。

4. **非 Linux 平台 `Sensor` 不可驻留。** `vendor/platform.c` 通过 Linux `I2C_RDWR` ioctl 访问总线，其他平台无此接口。`build.rs` 不编译 C，`Sensor` 被定义为持有 `Infallible` 且 `open` 始终失败——这不是假传感器（`--fake` 已存在），存在只是为了 `cargo test --workspace` 能离板运行。编译器因不可驻留而排除所有其他方法体。

## 类型

### `Generation`

```rust
pub enum Generation {
    L8cx,                         // revision 0x0C
    L5cx,                         // revision 0x02（更旧，大多数现场鸭子）
    Unknown { device_id, revision_id },
}
```

- `from_ids(device_id, revision_id)` — 由 revision 字节映射
- `as_str()` — `"VL53L8CX"` / `"VL53L5CX"` / `"unknown"`
- `driver() -> Option<Driver>` — `Unknown` 无驱动（不猜固件）

### `Driver`（仅 Linux）

`L5` / `L8`，每个方法是对不同 shim 的同一调用。故意不用函数指针表，使每个 `unsafe` 块能自说明意图。

### `Sensor`（Linux）

```rust
pub struct Sensor {
    generation: Generation,
    driver: Driver,
    ranging: bool,
}
```

### `Sensor`（非 Linux）

```rust
pub struct Sensor(std::convert::Infallible); // 不可驻留
```

## 关键函数

### `Sensor::open(bus, address) -> Result<Self>`（Linux）

1. `TAKEN.swap(true)` 抢占单实例；失败即报错
2. `open_inner` 失败时通过 `inspect_err` 释放 `TAKEN`，否则一次失败的打开会让本进程永远拒绝重试
3. `open_inner`：
   - `tof_probe_id` 读取 device_id + revision_id（在加载任何固件之前）
   - 由 `Generation::from_ids` 选代；`driver()` 为 `None` 则报错（避免上传错误固件砖块化探针）
   - `driver.open` → `is_alive` 握手（失败则 close）→ `init` 上传固件（耗时数秒）
   - 返回 `Sensor { ranging: false }`

### `start(hz)` / `data_ready()` / `read_frame()`

- `start` 配置 8×8 + 频率，置 `ranging=true`
- `data_ready`：1=有，0=无，其他=总线错误（调用方当作"传感器消失"）
- `read_frame`：两个 `[u8/ZONES]` 栈数组接收距离与状态，转为 `Frame`

### `Drop for Sensor`

若 `ranging` 则 `stop`（错误忽略——描述符总会关闭），然后 `close`，最后 `TAKEN.store(false)`。

## FFI 声明

```rust
unsafe extern "C" {
    fn tof_probe_id(dev_path, addr, *device_id, *revision_id) -> i32;
    fn vl5_open / vl5_close / vl5_is_alive / vl5_init / vl5_start / vl5_stop / vl5_data_ready / vl5_get_frame(...);
    fn vl8_* ...;
}
```

每个返回 `i32`，0/1 表成功/就绪，非零为 ULD 状态码。

## 单元测试（`#[cfg(all(test, target_os = "linux"))]`）

- `revisions_map_to_generations_and_drivers` — revision 0x0C→L8→Driver::L8，0x02→L5→Driver::L5；未知 revision→`Unknown` 且 `driver()` 为 `None`（上传错误固件会砖块化）。
- `a_failed_open_releases_the_claim` — 对不存在总线的 `open` 失败后 `TAKEN` 必须释放，且可重试。

## 关键摘要

sensor.rs 是 vendored C 驱动的安全边界：FFI 表面被收敛为标量 + 固定数组，`unsafe` 集中在单一形态；两代传感器由运行时 ID 探测决定固件；`TAKEN` 原子强制单实例并在失败时释放；非 Linux 平台用 `Infallible` 不可驻留类型让编译器排除所有方法体，保证绝不发明假帧。
