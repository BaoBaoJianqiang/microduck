# `io.rs` 解读

## 概述

`io.rs`（336 行）定义了控制循环与物理世界之间的**接缝（seam）**——`RobotIo` trait。核心设计哲学是：`read()` 和 `write()` 之间的一切都是对纯数据的纯计算，这使得整个控制循环可以在没有机器人硬件的情况下进行测试。

文件包含以下关键组件：
- `Sensors`：一次原子采样，关节数据与 IMU 数据**捆绑在一起**（由硬件拓扑决定）
- `JointTargets`：仅位置控制（alpha 机器人无速度模式关节）
- `SlowSensors`：电压与温度，每秒采样一次而非每 tick
- `ImuStale`：IMU 陈旧读数的双计数器（累计 + 当前连续）
- `RobotIo` trait：统一的 IO 接口
- `FakeIo`：测试用假 IO，位置回显，支持多种故障注入

## 关键结构体 / trait / 函数

### `Sensors`（第 14–36 行）

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Sensors {
    pub positions: [f64; NUM_JOINTS],   // 关节角度（弧度）
    pub velocities: [f64; NUM_JOINTS],  // 关节速度（rad/s）
    pub currents_ma: [f64; NUM_JOINTS], // 当前电流幅值（mA），丢弃符号
    pub imu: ImuData,                   // IMU 数据
}
```

—— 关节与 IMU 放在同一个结构体中，因为硬件就是这样做的：IMU 板位于 Dynamixel 总线上，与舵机在同一次事务中被读取。如果 trait 将它们分开，就会发明一个总线不存在的区别，并为了遵守这个区别而加倍总线流量。

`currents_ma` 丢弃了电流符号——方向可以从速度推断，且目前所有消费者只关心负载大小而非方向。

### `JointTargets`（第 38–48 行）

```rust
pub struct JointTargets {
    pub positions: [f64; NUM_JOINTS],
}
```

—— 仅位置控制。注释明确指出 "alpha has no velocity-mode joints"（alpha 没有速度模式关节），因此不需要速度目标字段。

### `IoError`（第 50–73 行）

```rust
pub enum IoError {
    Port { path: String, source: std::io::Error },  // 串口错误
    Bus(String),                                     // 总线事务失败
    ShortRead { what, expected, got },               // 短读：设备未应答
    Simulated,                                       // 模拟失败（FakeIo 用）
}
```

—— `ShortRead` 是关键变体：当 `sync_read` 返回的块数不对或块长度不对时，意味着某个设备没有应答。**报告而非掩盖**：静默的短读会在半个关节数组中留下陈旧值。

### `SlowSensors`（第 75–91 行）

```rust
pub struct SlowSensors {
    pub volts: f64,                  // 舵机平均供电电压（唯一的电池测量）
    pub temps_c: [f64; NUM_JOINTS],  // 每关节外壳温度（°C）
}
```

—— 每秒采样一次而非每 tick。理由：电池不会在 20ms 内放电，电机也不会在 20ms 内升温。这些寄存器位于地址 144–146，超出了 `read()` 读取的连续块范围，因此需要额外一次总线事务（约 1ms）。每秒一次可以忽略，但在 50Hz 下会占用 5% 的预算。

温度采用**每关节**而非平均值，因为有趣的场景是单个关节承载负载——深蹲时的膝盖比嘴热得多，15 个舵机的平均值会恰好隐藏即将触发热保护的那个舵机。

### `ImuStale`（第 93–111 行）

```rust
pub struct ImuStale {
    pub total: u64,  // 自启动以来的陈旧读数总数，累计不重置
    pub run: u64,    // 当前连续陈旧读数的长度，任何新鲜块将其归零
}
```

—— 两个数字回答两个不同的问题：
- **total**：整个运行期间板卡重复自身的频率。偶发命中是正常的，因为循环和板卡各有自己的时钟，一个 tick 落在板卡刷新周期内会合法地看到相同的字节两次。
- **run**：方向是否**此刻**被冻结。停止融合的板卡会在每个 tick 都重复，因此 run 会无限增长，而单独的 total 看起来和几次抖动一样。

两者一起报告，使得任何后端都不能只提供一个而不提供另一个——没有 total 来衡量的 run 正是该计数最初被误读为警报的原因。

### `RobotIo` trait（第 113–159 行）

```rust
pub trait RobotIo {
    fn read(&mut self) -> Result<Sensors>;
    fn write(&mut self, targets: &JointTargets) -> Result<()>;
    fn set_gain(&mut self, kp: u16) -> Result<()>;
    fn set_torque(&mut self, on: bool) -> Result<()>;
    fn slow_sensors(&mut self) -> Result<SlowSensors>;
    fn imu_stale(&self) -> ImuStale { ImuStale::default() }  // 默认"无异常"
    fn imu_ready(&self) -> bool { true }                      // 默认已就绪
}
```

#### `set_gain` 为什么在 trait 上？

它不仅仅是总线层的操作——它是让"变软（go limp）"有意义的东西。拒绝命令跌倒的机器人只会将其冻结在跌倒的姿势中；降低增益让它可以屈服。原型机做同样的事情：运行值 kP=200，跌倒时 kP=50。

#### `set_torque` 为什么在 trait 上？

因为启动机器人现在是控制循环在人类请求时做的事情，而所有到达电机的东西都经过 `safety::Safety`（它拥有唯一的写句柄）。

**关键不变量**：启动时**绝不**调用 `set_torque`。由更新重启的 `robotd` 必须让站立的机器人保持站立——扭矩在有人启用策略时才开启，绝不是因为进程开始了。

#### `imu_stale` 和 `imu_ready` 的默认实现

默认返回"无异常"和"已就绪"，这样 FakeIo 或未来的后端不必发明诊断数据。

### `FakeIo`（第 161–294 行）

```rust
pub struct FakeIo {
    sensors: Sensors,
    pub fail_next_read: bool,   // 下一次 read 失败后自动清除
    pub fail_reads: u32,        // 还需失败的读数次数（模拟舵机尚未上电）
    pub last_written: Option<JointTargets>,
    pub reads: usize,
    pub writes: usize,
    pub last_gain: Option<u16>, // 区分"变软"和"停止命令"
    pub imu_ready: bool,        // 模拟启动后前几秒方向滤波器未收敛
    pub slow: Option<SlowSensors>, // None 表示 slow_sensors 失败
    track_targets: bool,        // true 时 read 回显最后写入的位置
    pub torque: Option<bool>,   // None = "重启没有移动机器人"
    pub torque_writes: usize,   // 区分"启动一次"和"每 tick 都写"
}
```

—— 始终编译（非 `#[cfg(test)]`），因为它是测试套件运行的对象，也是 `cargo test` 不需要硬件、网络或 Docker 的原因。

#### 构造器与配置方法

- `new()`：默认 mid-pack（7.4V）、手温（32°C）、`track_targets=true`
- `at(positions)`：从已知位姿开始（通常是 `DEFAULT_POSITION`）
- `frozen()`：冻结报告位置，忽略写入——模拟 limp 或被手推动的机器人
- `failing_reads(n)`：前 n 次 read 失败，然后正常——模拟板卡比舵机电先启动
- `set_imu(imu)`：注入 IMU 数据

#### `read()` 实现

```rust
fn read(&mut self) -> Result<Sensors> {
    if self.fail_next_read { self.fail_next_read = false; return Err(IoError::Simulated); }
    if self.fail_reads > 0 { self.fail_reads -= 1; return Err(IoError::Simulated); }
    self.reads += 1;
    Ok(self.sensors)
}
```

—— `fail_next_read` 是一次性的（失败后自动清除），否则注入一次失败的测试永远无法恢复，循环的重试路径就无法测试。

#### `write()` 实现

当 `track_targets=true` 时，写入的目标位置会被复制到 `sensors.positions`，因此下一次 `read()` 会回显——行为像一个完美跟踪的舵机。

## 重要常量

本文件无独立常量，所有维度常量（`NUM_JOINTS` 等）来自 `crate::model`。

## 测试要点（第 296–335 行）

### `fake_io_tracks_what_was_written`

验证 FakeIo 的核心契约：写入什么就读回什么。如果 FakeIo 忽略写入，每个保持位姿的测试都会**空洞地通过**（vacuously pass）。

### `frozen_fake_io_ignores_writes`

验证 `frozen()` 模式：limp 或被手握住的机器人不跟随命令。安全层的工作需要能针对这种情况进行测试。

### `simulated_read_failure_clears_itself`

验证 `fail_next_read` 是一次性的：一次失败后下一次必须成功，否则循环的重试路径不可测试。

## 与其他模块的关系

- **被 `lib.rs` 重新导出**：`RobotIo`、`Sensors`、`FakeIo` 等是 crate 公共 API 的核心。
- **依赖 `imu::ImuData`**：`Sensors` 内嵌 IMU 数据。
- **依赖 `model::NUM_JOINTS`**：所有关节数组的维度。
- **被 `bus` 实现**：`bus::Bus` 是 `RobotIo` 的真实硬件实现。
- **被 `safety` 消费**：`Safety` 持有 `Box<dyn RobotIo>`，是唯一调用 `write()` 的地方。
- **被 `obs` 消费**：`Observation::build` 接收 `&Sensors` 构建观测向量。
- **被测试广泛使用**：`FakeIo` 是整个 crate 测试基础设施的基石。
#（注：内容由AI生成）
