# `bus.rs` 解读

## 概述

`bus.rs`（496 行）是 `duck-control` crate 的**真实硬件 IO 层**——`RobotIo` trait 的 Dynamixel 总线实现。它把 `rustypot` 提供的字节级总线操作封装成控制循环可以直接使用的 `Sensors` / `JointTargets` 类型。

核心设计意图是**用最少的总线事务完成每 tick 一次的读写**：

1. 每个 tick 一次**组合 `sync_read`**，覆盖 IMU 板与全部 15 个舵机（IMU 排在 ID 列表首位，使其先应答，缩短总线占用）；紧接一次 `sync_write` 下发目标位置。
2. 电池电压与外壳温度不适合塞进每 tick 的块——它们位于块外寄存器，因此 `slow_sensors()` 是独立事务，约每秒调用一次。
3. 数字（换算系数、启动时断言的 EEPROM 寄存器）来自 `microduck_runtime` 在真机上反复调出来的经验值，而非手册默认值——本文件只负责照抄并测试。

文件顶部文档注释明确：本层面向 `rustypot`，但**数值**与启动时断言的寄存器来自 `microduck_runtime`（见 `crate::model`）。

## 关键结构体 / 函数

### `DynamixelIo`（第 89–98 行）

```rust
pub struct DynamixelIo {
    controller: Xl330Controller,              // rustypot 的 XL-330 控制器 —— rustypot's XL-330 controller
    /// IMU first, then the servos in `JOINT_IDS` order — the order blocks come back in.
    //  IMU 在前，随后按 JOINT_IDS 顺序排列舵机 —— 即块返回的顺序
    ids: Vec<u8>,
    imu: SflpDecoder,                           // IMU 块解码器 —— IMU block decoder
    stale_imu: StaleImuTracker,                 // 检测"应答但不刷新"的 IMU 板 —— detects an answering-but-not-refreshing IMU board
}
```

—— `ids` 的顺序就是 `sync_read_raw_data` 返回块的顺序：**槽 0 永远是 IMU 板**，槽 1..16 是 15 个舵机。这个顺序在 `read()` 中被硬编码依赖。

### `DynamixelIo::open`（第 100–124 行）

```rust
pub fn open(port: &str) -> Result<Self> {
    let serial = serialport::new(port, BAUD_RATE)
        .timeout(READ_TIMEOUT)                 // 30 ms 超时，而非串口驱动默认值
        .open()
        .map_err(|e| IoError::Port { /* ... */ })?;
    let controller = Xl330Controller::new()
        .with_protocol_v2()
        .with_serial_port(serial);
    let mut ids = Vec::with_capacity(NUM_JOINTS + 1);
    ids.push(IMU_DXL_ID);
    ids.extend_from_slice(&JOINT_IDS);
    // ...
}
```

—— 启动时即把超时设为 `READ_TIMEOUT`（30 ms），而不是用串口驱动的默认超时。理由：默认超时会让一个丢失的设备卡住整个控制循环；30 ms 足够健康的 16 设备读完成，缺失设备只造成一次有界的停顿。

### `check_registers`（第 126–169 行）

```rust
pub fn check_registers(&mut self) -> Result<usize> {
    let mut fixed = 0;
    for &id in &JOINT_IDS {
        for &(name, want) in EXPECTED_REGISTERS {
            // rustypot returns a Vec even for a single-id read. An empty one means the
            // servo did not answer, which must not be read as "register is fine".
            // rustypot 即使单 ID 读也返回 Vec。空 Vec 表示舵机未应答，绝不能
            // 被解读为"寄存器没问题"
            let raw = match name { /* read_return_delay_time / baud_rate /
                                    pwm_slope / shutdown */ }
                .map_err(|e| IoError::Bus(format!("read {name} on {id}: {e}")))?;
            let got = *raw.first().ok_or(IoError::ShortRead { /* ... */ })?;
            if got == want { continue; }
            tracing::warn!(id, register = name, got, want, "correcting motor register");
            // 写回期望值并计数
        }
    }
    Ok(fixed)
}
```

—— 启动时**断言并纠正**控制循环依赖的 EEPROM 寄存器。设计理由：一个被恢复出厂或新换上的舵机自带 `return_delay_time = 250`，仅这一项就会吃掉跨总线 40% 的 tick 预算。启动时每寄存器多读一次，就消灭了一整类"为什么这台机器人这么慢"的问题。

**踩坑点**：rustypot 单 ID 读也返回 `Vec`，空 Vec 是"没应答"而非"读到了零"。代码用 `raw.first().ok_or(...)` 把这种情况升级为 `ShortRead`，而不是悄悄认为寄存器正确。

### `present_positions`（第 171–188 行）

```rust
pub fn present_positions(&mut self) -> Result<[f64; NUM_JOINTS]> {
    let values = self.controller.sync_read_present_position(&JOINT_IDS)
        .map_err(|e| IoError::Bus(format!("read present positions: {e}")))?;
    // 长度校验后拷贝
}
```

—— 只读位置，比 `RobotIo::read` 更轻。启动时调用一次，用于** adopting 机器人当前已经在的姿势**（`interpolate_to` 从这里取起点）。

### `set_torque`（第 190–202 行）

```rust
pub fn set_torque(&mut self, on: bool) -> Result<()> {
    for &id in &JOINT_IDS {
        self.controller.write_torque_enable(id, on)
            .map_err(|e| IoError::Bus(format!("torque {on} on {id}: {e}")))?;
    }
    Ok(())
}
```

—— 每个关节一次事务，**绝不能每 tick 调用**。控制循环只在有人对一台 limp 的机器人启用策略时调用一次。关键不变量见 `RobotIo::set_torque` 的注释：进程启动时**什么都不碰**扭矩——`robotd` 重启必须让站立的机器人继续站立。

### `interpolate_to`（第 204–229 行）

```rust
pub fn interpolate_to(
    &mut self,
    target: &[f64; NUM_JOINTS],
    duration: Duration,
    step: Duration,
) -> Result<()> {
    let start = self.present_positions()?;
    let steps = (duration.as_secs_f64() / step.as_secs_f64())
        .ceil().max(1.0) as u32;
    for i in 1..=steps {
        let t = i as f64 / steps as f64;
        let mut next = [0.0; NUM_JOINTS];
        for j in 0..NUM_JOINTS {
            next[j] = start[j] + (target[j] - start[j]) * t;  // 线性插值
        }
        self.write(&JointTargets::new(next))?;
        std::thread::sleep(step);                              // 阻塞睡眠
    }
    Ok(())
}
```

—— 从当前位置线性斜坡到目标位置。**阻塞，且刻意为之**：这段时间总不应有别的东西在总线上说话。只被显式的 `init` 调用——控制循环绝不能自己动机器人，否则一次更新重启就是一次跌倒风险。

### `RobotIo::read`（第 232–291 行）

```rust
fn read(&mut self) -> Result<Sensors> {
    let blocks = self.controller
        .sync_read_raw_data(&self.ids, READ_ADDR, READ_LEN)
        .map_err(|e| IoError::Bus(format!("combined imu+motor sync_read: {e}")))?;
    if blocks.len() != self.ids.len() {
        return Err(IoError::ShortRead { /* ... */ });
    }
    let mut sensors = Sensors::default();

    // Slot 0 is the IMU board. —— 槽 0 是 IMU 板
    if blocks[0].len() == IMU_BLOCK_LEN {
        let mut raw = [0u8; IMU_BLOCK_LEN];
        raw.copy_from_slice(&blocks[0]);
        let run = self.stale_imu.observe(&raw);
        // 仅在连续 25 次陈旧时首次告警；之后每 500 次再报一次，避免 50 Hz
        // 的相同警告把日志冲掉
        if run == STALE_RUN_WARN || (run > STALE_RUN_WARN && run.is_multiple_of(500)) {
            tracing::warn!(consecutive = run, total = self.stale_imu.stale.total,
                "imu board has returned the same sample {run} reads running");
        }
        sensors.imu = self.imu.decode(&raw);
    } else {
        return Err(IoError::ShortRead { /* ... */ });
    }

    for (joint, block) in blocks[1..].iter().enumerate() {
        // [0..2] present_pwm, unused · [2..4] current · [4..8] velocity · [8..12] position
        sensors.currents_ma[joint] =
            (i16::from_le_bytes([block[2], block[3]]) as f64).abs();
        let velocity = i32::from_le_bytes([block[4], block[5], block[6], block[7]]);
        sensors.velocities[joint] = velocity as f64 * RAD_PER_SEC_PER_COUNT;
        let position = i32::from_le_bytes([block[8], block[9], block[10], block[11]]);
        sensors.positions[joint] = (2.0 * PI * position as f64 / 4096.0) - PI;
    }
    Ok(sensors)
}
```

—— 一次事务读完 IMU + 15 个舵机。舵机块的字段布局：

| 偏移 | 长度 | 含义 | 处理 |
|---|---|---|---|
| 0..2 | 2 | present_pwm | 丢弃（未使用） |
| 2..4 | 2 | present_current | `i16` LE 取绝对值 → mA |
| 4..8 | 4 | present_velocity | `i32` LE × `RAD_PER_SEC_PER_COUNT` |
| 8..12 | 4 | present_position | `i32` LE → `2π·raw/4096 − π` 弧度 |

位置换算 `2π·raw/4096 − π`：XL-330 的 4096 个计数对应一圈，范围映射到 `[−π, +π)`。该换算必须与 `sync_write_goal_position` 出站时 rustypot 自己的 `AnglePosition` 约定一致，否则控制循环下发的角度和它以为读回来的角度不是同一个值（有测试守护）。

**告警节流**：陈旧 IMU 板每个 tick 都吐同一块，50 Hz 的相同 warning 会把日志（journal）冲掉，因此首次达到 `STALE_RUN_WARN` 报一次，之后每 500 次才再报。

### `RobotIo::write`（第 293–297 行）

```rust
fn write(&mut self, targets: &JointTargets) -> Result<()> {
    self.controller
        .sync_write_goal_position(&JOINT_IDS, &targets.positions)
        .map_err(|e| IoError::Bus(format!("sync_write goal positions: {e}")))
}
```

—— 一次 `sync_write` 把 15 个目标位置全部下发，不碰 IMU ID。

### `RobotIo::set_gain`（第 304–325 行）

```rust
fn set_gain(&mut self, kp: u16) -> Result<()> {
    // I and D are written too, at zero — the prototype's `--ki`/`--kd` defaults...
    // I 和 D 也一并写零 —— 原型机启动时给每个电机写的默认值
    // RAM 寄存器，每次上电都会恢复出厂值；出厂 D 增益不为零，
    // 留在那里会阻尼舵机内部 PID，使机器人在相同 kP 下明显偏软。
    // 这不是谁做的调参选择，因此钉死在这里，不暴露成旋钮
    const KI: u16 = 0;
    const KD: u16 = 0;
    for &id in &JOINT_IDS {
        self.controller.write_position_p_gain(id, kp)?;
        self.controller.write_position_i_gain(id, KI)?;
        self.controller.write_position_d_gain(id, KD)?;
    }
    Ok(())
}
```

—— 关键踩坑点：即使调用方只关心 kP，也必须把 I、D 一并写零。因为这些是 **RAM 寄存器**，每次上电恢复出厂值；而出厂 D 增益非零，会阻尼舵机内部 PID，导致"同一个 kP 下机器人比原型机软"。这不是有意的调参，所以钉死成常量。

### `RobotIo::slow_sensors`（第 339–383 行）

```rust
fn slow_sensors(&mut self) -> Result<SlowSensors> {
    let blocks = self.controller
        .sync_read_raw_data(&JOINT_IDS, SLOW_READ_ADDR, SLOW_READ_LEN)?;
    // [0..2] present_input_voltage · [2] present_temperature
    let counts = u16::from_le_bytes([block[0], block[1]]);
    let v = counts as f64 * VOLTS_PER_COUNT;   // 0.1 V/count
    if v > 0.0 { volts.push(v); }              // 零值过滤：乱答的设备不能被平均进电池电压
    temps_c[joint] = block[2] as f64;
    // 电压取平均（15 个舵机同一块电池）；温度逐关节保留（不平均）
}
```

—— 电压平均、温度不平均的设计理由：15 个舵机共用一组电池，单次测量就是同一个测量、只是更吵，所以平均降噪；而单个关节过载发烫才是值得看的情况，15 个温度的平均值恰好会隐藏那个即将触发热保护的舵机。

**全有或全无语义**：rustypot 的 `sync_read` 等所有 ID，任何一个不应答就整个事务失败——静默的舵机不会产生短答案，而是整笔失败。调用方应保留上一次采样，而不是把一次丢失当成新闻。电压的 `v > 0` 过滤防止乱答为 0 的设备被平均成"电池半电"。

### `StaleImuTracker`（第 61–87 行）

```rust
#[derive(Debug, Default)]
struct StaleImuTracker {
    /// `None` until the first block. A fixed initial value cannot work here: it would have
    /// to be all zeros, and an all-zero block is exactly what a board whose SFLP table is
    /// still empty sends — scoring a stale read against a predecessor that never existed.
    /// 首个块之前为 None。固定初值行不通：它只能是全零，而全零块恰好是
    /// SFLP 表还空着的板子发出来的 —— 会把一次陈旧读数记在一个从未存在的
    /// 前任头上
    last: Option<[u8; IMU_BLOCK_LEN]>,
    stale: ImuStale,
}

impl StaleImuTracker {
    fn observe(&mut self, block: &[u8; IMU_BLOCK_LEN]) -> u64 {
        if self.last.replace(*block) == Some(*block) {
            self.stale.total = self.stale.total.saturating_add(1);
            self.stale.run = self.stale.run.saturating_add(1);
        } else {
            self.stale.run = 0;
        }
        self.stale.run
    }
}
```

—— 检测"会应答但不刷新"的 IMU 板。从读路径里拆出来，是为了**没有串口也能测**：它描述的故障机器人上没有别的模块会报告，否则只能对着坏硬件验证。

`last` 用 `Option` 而非全零初值，是真实踩过的坑：全零初值会让每次开机第一个 tick 就把"SFLP 表还空"的板子报成一次陈旧读数，计数器被永久 +1 并渲染成警报。

## 重要常量

| 常量 | 值 | 含义 |
|---|---|---|
| `READ_ADDR` | `124` | 每 tick 读取的连续块起点：present_pwm/current/velocity/position |
| `READ_LEN` | `12` | 12 字节覆盖上述四个字段；恰好也是 IMU 板在同一地址吐出的长度 |
| `RAD_PER_SEC_PER_COUNT` | `0.229·(2π/60)` | 每个速度计数 = 0.229 rev/min，换算成 rad/s |
| `SLOW_READ_ADDR` | `144` | 慢传感器块起点：present_input_voltage（u16）+ present_temperature（u8） |
| `SLOW_READ_LEN` | `3` | 3 字节同时覆盖电压与温度，比分两次读省一次事务 |
| `VOLTS_PER_COUNT` | `0.1` | present_input_voltage 每个计数 0.1 V |
| `READ_TIMEOUT` | `30 ms` | 16 设备健康读远小于此；缺失设备只造成有界停顿 |
| `STALE_RUN_WARN` | `25` | 连续陈旧读数达到此值才告警；50 Hz 下约半秒，与 `SflpDecoder::ready` 的等待窗一致 |

`SLOW_READ_ADDR=144` 紧接 tick 块末尾之后 8 字节，中间隔了 `velocity_trajectory` / `position_trajectory`（12 字节，本层不要）。若把这些并进 `READ_LEN`，舵机每 tick 要答 22 字节，比分两笔事务花的总线时间更多——这是"为什么不合并"的实测权衡。

## 测试要点（第 394–495 行）

### `read_block_is_long_enough_for_every_field`

编译期断言 `READ_LEN == IMU_BLOCK_LEN`，且解析器触达的最高偏移（position 8..12）不越界。若 `READ_LEN` 与字段偏移不一致，关节会拿到彼此的值——表现成接线故障而非代码 bug。

### `position_conversion_round_trips_through_rustypot`

对 0/1024/2048/3072/4095 五个原始计数做"本文件弧度 → rustypot 计数"往返，必须严格相等。守护读回与写出角度约定一致。

### `velocity_scale_matches_the_datasheet_figure`

验证 `RAD_PER_SEC_PER_COUNT` 反推回 rpm 恰为 0.229。这个系数错了会给观测向量里每个关节速度乘一个常数，策略恰好能"勉强容忍"——结果就是走得很差。

### `the_first_block_is_never_stale`

首个块没有前任，不能算陈旧。守护 `last: Option` 设计——历史上全零初值曾导致每次开机第一个 tick 就永久 +1 警报。

### `fresh_blocks_count_for_nothing`

连续 10 个不同块后 total 与 run 都为 0。run 表示"此刻"，任何不重复的块必须清零。

### `a_hiccup_is_remembered_in_the_total_but_not_the_run`

两次相同块（run=1）后接一个新块：total=1 记住这次抖动，run=0 表示方向已恢复。total 回答"这事发生过几次"，run 回答"现在是不是冻住了"。

### `a_dead_board_runs_past_the_warning_threshold`

死板连续重复时 run 必须正好爬到 `STALE_RUN_WARN`，否则真正死掉的 IMU 永远不会被上报。

### `separate_episodes_add_up`

三段独立抖动（每段重复 2 次）后 total=6、run=2（最后一段仍在继续）。守护 total 跨情节累计。

## 与其他模块的关系

- **实现 `crate::io::RobotIo`**：是该 trait 的真实硬件后端；`FakeIo` 是它的无硬件替身。
- **依赖 `crate::imu`**：用 `IMU_BLOCK_LEN` 校验 IMU 块长度，用 `SflpDecoder` 解码槽 0 的 12 字节。
- **依赖 `crate::model`**：`BAUD_RATE`、`EXPECTED_REGISTERS`、`IMU_DXL_ID`、`JOINT_IDS`、`NUM_JOINTS`。
- **依赖 `crate::io` 的错误与数据类型**：`ImuStale`、`IoError`、`JointTargets`、`Result`、`RobotIo`、`Sensors`、`SlowSensors`。
- **被 `robotd` 调用**：`open` → `check_registers` → 控制循环 `read`/`write`；`set_gain` 由安全层在"变软"时调用；`slow_sensors` 约每秒一次。
- **数值口径来自 `microduck_runtime`**：换算系数与启动断言的寄存器清单都是真机调校结果，本文件只负责测试它们没被改坏。
#（注：内容由AI生成）
