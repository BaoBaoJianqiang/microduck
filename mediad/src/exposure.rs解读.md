# `exposure.rs` 解读

## 概述

`exposure.rs` 是 mediad（Microduck 机器人的媒体守护进程）中的**软件自动曝光（Software Auto-Exposure, AE）闭环模块**。它在一个独立的系统线程里以 500ms 为周期采样 GStreamer 管线 tee 分支上的原始 UYVY 帧，计算平均亮度（luma），与目标亮度比较后，用阻尼乘性步长调节摄像头的三个控制量（曝光行数、模拟增益、ISP 数字增益），通过 `v4l2-ctl` 写入传感器。

**核心设计意图**：Rockchip 的 `rkaiq_3A_server` 虽然在 stream-start 时做过一次曝光收敛，但之后就不再响应，也不会随场景变化持续调节。机器人从窗边走进走廊后画面仍停留在窗边的曝光值；当 3A 引擎错过 stream-start 事件时，连那一次收敛都不发生，画面就一直停留在 mediad 启动时钉住的值。本模块用一个运行在 mediad 进程内的闭环循环，把"曝光"从"两个进程的启动顺序"中彻底解耦出来。

模块从 `microduck_runtime` 的 `camera.rs` 原型移植而来，在原型里跑了数月。与原型相比有两处改进：

- **luma 直接取自 tee 的 raw 分支**（原本就为 duck 检测器和 `get_frame` surface 存在），帧格式是 UYVY，luma 隔一个字节就是，求均值只需一次抽样子走，无需解码。
- **不再第二次打开摄像头做测量**。原型第一版用并行的 `v4l2-ctl` 采样 ISP self path，在驱动层面与采集竞争，间歇性杀死管线；读已有的帧不会。

---

## 关键常量

| 常量 | 值 | 含义 |
|---|---|---|
| `INTERVAL` | 500ms | 采样周期。"Twice a second: fast enough to follow a robot walking from a window into a corridor, slow enough that the damped step never rings." —— 每秒两次：跟得上机器人从窗边走进走廊，又慢到阻尼步长不会振荡。 |
| `REASSERT_TICKS` | 20（即 10 秒） | 静默 tick 计数上限，到点就把当前值再写一遍。 |
| `REPORT_TICKS` | 20（即 10 秒） | 周期性输出"loop 活着"的 info 日志的间隔。 |
| `TARGET_Y` | 90.0 | 平均 luma 设定点，8-bit，**经过 ISP gamma 曲线之后**的值。 |
| `DEADBAND` | 0.12 | 相对误差死区。没有死区时，传感器噪声本身就会让曝光持续抖动，画面看起来在"呼吸"。 |
| `SOFT_LINES` | 600.0（≈11.4ms） | 软快门上限。先把快门推到这里，再用增益。 |
| `HARD_LINES` | 1200.0（≈22.9ms） | 硬快门上限。真实天花板：驱动对超过帧长的曝光会**拉伸帧时间**而非截断，请求 3500 行帧率会从 29.96fps 掉到 15.1fps。模式总行数 1766。 |
| `MAX_ANALOGUE` | 11.0（倍数） | 传感器模拟增益上限。寄存器值 = 倍数 × 256。 |
| `MAX_DIGITAL` | 16.0（倍数） | ISP 数字增益上限。远低于传感器能接受的值，因为超过后画面只是变亮，不再变清楚。 |

### 为什么 `REASSERT_TICKS` 是 20

```rust
// Because this loop is not the only thing that writes the sensor.
// -- 因为写传感器的不止这个循环。
// v4l2src applies --exposure/--analogue-gain through extra-controls whenever the device is opened,
// including a re-open nobody here initiated
// -- v4l2src 每次打开设备（包括我们没发起的重开）都会通过 extra-controls 套用 --exposure/--analogue-gain
```
跳过写操作是因为"我们已经写过那个值"的假设，恰恰是让相机变黑、日志却说已收敛的元凶。所以这个假设要在一个慢心跳上重新校验：每次调用 43ms CPU，间隔 10 秒只占不到半核的 0.5%，而每 tick 都写要占十分之一个核。

### 为什么 `REPORT_TICKS` 用 info 而非 debug

```rust
// At info, not debug, because the question "is auto-exposure alive?" should not need a drop-in.
// -- 用 info 而不是 debug，因为"自动曝光还活着吗？"这个问题不该需要临时改日志级别才能回答。
```
已收敛的循环什么都不写，从传感器角度和"测光冻结帧"或"循环根本没启动"无法区分；区别在于测得的 luma，只有这行日志携带。10 秒一条 = 每分钟 6 行。

### 为什么三档亮度预算要按"噪声顺序"花

```rust
// The two shutter caps are the whole reason this steers three controls rather than one.
// -- 两个快门上限就是为什么这里要同时驾驭三个控制量而非一个。
// Brightness is spent in noise order: shutter up to the soft cap first (cheapest and cleanest...),
// then sensor analogue gain (clean amplification), then shutter up to the hard cap,
// and ISP digital gain — the noisiest — only when there is nothing else left.
// -- 亮度按噪声顺序花：先快门到软上限（最便宜最干净，11ms 短到走路的机器人不会糊），
//    再传感器模拟增益（干净放大），再快门到硬上限，最后才是最吵的 ISP 数字增益。
```

---

## 关键结构体

### `Controls`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Controls {
    pub exposure: u32,        // 曝光，单位 sensor lines
    pub analogue_gain: u32,   // 模拟增益，256 = 1x
    pub gain: u32,            // ISP 数字增益，256 = 1x
}
```

loop 写给传感器的值，用传感器自己的单位。`PartialEq, Eq` 派生是为了 `step()` 末尾与 `written` 比较，避免把相同的值再写一遍。

### `Ae`

```rust
pub struct Ae {
    exposure: f64,
    analogue: f64,
    digital: f64,
    written: Option<Controls>,  // 上次交付的值，保证相同值不会写两次
}
```

loop 的状态：一个亮度预算，拆分到三个控制量上。内部用 `f64` 保持分数，输出时才转成寄存器整数。

#### `Ae::starting_at(exposure_lines, analogue_gain_reg)`

```rust
// Starting from what mediad already wrote to the sensor, so the first step is relative to the
// picture on screen rather than to a number this module invented.
// -- 从 mediad 已经写给传感器的值起步，这样第一步是相对屏幕上的画面，而不是相对本模块凭空造的数字。
```
曝光行数 clamp 在 `[4.0, HARD_LINES]`，模拟增益 clamp 在 `[1.0, MAX_ANALOGUE]`，数字增益初始 1.0。

#### `Ae::step(mean_luma) -> Option<Controls>`

核心算法。要点：

1. **ratio = (TARGET_Y / mean_luma).clamp(0.25, 4.0)**：亮度比，限制在 4 倍范围内，防止单步过激。
2. **死区检查**：`(1.0 - ratio).abs() < DEADBAND` 直接返回 `None`。
3. **阻尼乘性步长**：
   ```rust
   let budget = (self.exposure * self.analogue * self.digital * ratio.powf(0.6))
       .clamp(4.0, HARD_LINES * MAX_ANALOGUE * MAX_DIGITAL);
   ```
   ```rust
   // The step is multiplicative on the product of the three controls, because that product is
   // what luma is proportional to; the split back into three is where the noise ordering lives.
   // ratio^0.6 rather than ratio damps it: the controls are linear in light and the measured
   // luma is not, so a full correction against a gamma-compressed measurement overshoots and
   // the loop hunts.
   // -- 步长对三个控制量的乘积做乘性调整，因为 luma 与该乘积成正比；拆回三个量时才是噪声顺序所在。
   //    用 ratio^0.6 而非 ratio 来阻尼：控制量对光线性，但测得的 luma 不是，
   //    所以对 gamma 压缩后的测量做完整修正会过冲，loop 会振荡。
   ```
4. **预算拆分**（按噪声顺序）：
   - 先把 `exposure` 拉到 `min(budget, SOFT_LINES)`；
   - 模拟增益 = `(budget / exposure).clamp(1, MAX_ANALOGUE)`；
   - 若还有余量，`exposure` 再拉到硬上限 `clamp(exposure, HARD_LINES)`；
   - 数字增益 = `(budget / (exposure * analogue)).clamp(1, MAX_DIGITAL)`。
5. **与上次写入比较**：相同则返回 `None`。这不是优化——
   ```rust
   // A room darker than the sensor can reach ... leaves the ratio permanently outside the deadband,
   // because the setpoint is unreachable rather than merely far away. Without this the loop writes
   // the same three numbers twice a second for as long as the robot is in that room, and each write
   // is a v4l2-ctl process: measured on the board at 43 ms of CPU a call...
   // -- 比传感器能力还暗的房间…… ratio 永远在死区外，因为设定点不可达而非只是遥远。
   //    没有这个判断，机器人在那个房间里每秒两次写相同的三个数，每次写都是一个 v4l2-ctl 进程，
   //    板上实测每次 43ms CPU……
   ```

#### `Ae::current() -> Controls`

周期性 re-assert 心跳用：返回当前值，并把它记为"已写"，所以心跳会重启而非之后每 tick 都触发。

### `Stop(Arc<AtomicBool>)`

```rust
/// Set when the daemon is going away, so the thread does not outlive the pipeline it steers.
/// -- 守护进程退出时置位，让线程不会活得比它所驾驭的管线还久。
```
用 `Relaxed` 序——只是一个布尔停止信号，不需要跨线程的内存序保证。

---

## `mean_luma(frame: &Frame) -> Option<f64>`

UYVY 把 luma 放在每个奇数字节。从 index=1 开始，每 16 字节（即每 8 像素）取一个：

```rust
let mut index = 1; // U Y V Y: the first luma byte —— U Y V Y：第一个 luma 字节
while index < frame.data.len() {
    sum += frame.data[index] as u64;
    count += 1;
    index += 16; // every 8th pixel —— 每第 8 个像素
}
```
1280×720 帧抽 11k 个样本，均值精度足够三位。

格式不匹配时返回 `None` 而不是瞎猜：
```rust
// None for any other format rather than a wrong answer — a mean computed over the wrong byte
// layout would still look like a plausible brightness, and the loop would chase it.
// -- 其他格式返回 None 而不是给出错误答案——在错误字节布局上算出的均值看起来仍然像合理的亮度，
//    loop 会去追它。
```

---

## `spawn(device, frames, exposure_lines, analogue_gain_reg) -> Stop`

启动 loop。

- `device` 是**采集节点**而非 sensor subdev，因为 rkisp 通过它代理 sensor 控制——这也是 `v4l2src` 通过 `extra-controls` 写入初始曝光的同一条路径。只开一个节点，而且是我们已知能开的那个。
- 线程名 `"auto-exposure"`，便于 `ps`/日志定位。
- 启动日志打印 `target_luma`、shutter 软/硬上限、模拟增益上限。

### 主循环要点

```rust
while !mine.stopped() {
    std::thread::sleep(INTERVAL);
    ...
    let metered = frames.inspect(mean_luma);
    ...
}
```

关键状态变量：

- `proven`：是否已经成功读过回写一次。用于把"第一次失败"打成 error、后续失败降级为 debug。
- `waiting_logged`：没帧可测时只打一次 debug。
- `format_logged`：帧格式不对时只打一次 error（之后 loop 永远测不到亮度）。
- `quiet`：距上次写操作的 tick 数，到 `REASSERT_TICKS` 就 re-assert。
- `since_report`：距上次 info 报告的 tick 数。

### 读回验证（read-back）

```rust
// Every way the write path can fail is silent, and all of them leave the sensor at the value
// mediad pinned at startup — so a read-back that finds the pin proves nothing if the pin is also
// what we asked for. The check would pass on the strength of somebody else's write.
// -- 写路径任何一种失败都是静默的，所有失败都让传感器停在 mediad 启动时钉住的值——
//    所以如果读回发现就是钉住的值，而我们请求的也正是那个值，这次读回什么都没证明。
//    这个检查会靠着别人的写通过。
```
`worth_reading_back(proven, asked, pinned)` 只在"还没被验证过 **且** 请求值 ≠ 启动钉住值"时才读回。第一次 step 落在 600 行（正是 mediad 钉的值）时报成功，是真实发生过的 bug。

读回成功后打 info `auto-exposure is driving the sensor`；若请求值与落地值不一致，打 error——"write reported success and the sensor did not take it"。

### 写失败的降级

```rust
// At debug, not an error twice a second for as long as the daemon runs:
// the first one already said what is wrong.
// -- 用 debug，而不是守护进程跑多久就每秒两次 error：第一次已经说明问题了。
```
第一次失败打 error，并在日志里直接给出排查命令 `v4l2-ctl -d {device} --list-ctrls`。

---

## `write(device, controls)` 与 `set(device, controls)`

用 `v4l2-ctl` 而非直接 ioctl：

```rust
// v4l2-ctl rather than the ioctl, for the prototype's reason: the struct layout of
// VIDIOC_S_EXT_CTRLS is three nested types we would have to pin by hand, and getting one offset
// wrong is a write that succeeds and changes nothing. The tool is in v4l-utils, which the
// capture path already needs for media-ctl.
// -- 用 v4l2-ctl 而非 ioctl，原因和原型一样：VIDIOC_S_EXT_CTRLS 的结构体布局是三层嵌套类型，
//    我们得手工对齐，一个偏移错了就是"写成功了但什么都没变"。这个工具在 v4l-utils 里，
//    采集路径本来就要用它跑 media-ctl。
```

**两次 `set`，不是一次**：

```rust
// Two calls, not one. exposure and analogue_gain belong to the sensor and gain to the ISP,
// and v4l2-ctl fails a whole --set-ctrl if any name in it is unknown on the node. One call would
// mean a board that spells digital gain differently loses its shutter as well, which is the
// entire picture rather than the last stop of brightness.
// -- 两次调用，不是一次。exposure 和 analogue_gain 属于 sensor，gain 属于 ISP，
//    而 v4l2-ctl 只要其中一个名字在节点上不认识就整个 --set-ctrl 失败。
//    一次调用意味着把数字增益拼法不同的板子会连带失去快门，那是整张画面而非亮度的最后一档。
```
且 `gain == 256`（1x）时跳过写数字增益——那是 no-op 写，跳过它能让没有数字增益的节点根本不会报错。

`set()` 把 stderr 包装进 `io::Error::other`，便于上层用 `%e` 直接打印。

---

## 测试要点

`mod tests` 共 9 个测试，覆盖了所有踩坑点：

1. **`luma_comes_from_the_luma_bytes`** — 构造 UYVY 帧，奇数字节=64、偶数字节=200，验证 `mean_luma` 只取 luma 字节得到 64.0。防止走错字节布局。
2. **`a_format_we_cannot_read_is_not_guessed_at`** — 把 format 改成 `"NV12"`，断言返回 `None`，不瞎猜。
3. **`a_picture_at_the_setpoint_is_left_alone`** — 在设定点和死区两侧都返回 `None`。
4. **`a_dark_picture_gets_more_light_and_a_bright_one_less`** — 暗帧 20.0 → 亮度乘积上升；亮帧 220.0 → 亮度乘积下降。验证方向。
5. **`light_is_spent_on_the_shutter_before_the_gains`** — 从最暗状态起步，第一步快门增长到软上限内，两个增益都不动（256=1x）。验证噪声顺序。
6. **`a_room_darker_than_the_sensor_can_reach_stops_being_written_to`** — 真实在机器人上见过的 bug：钉在所有天花板后 ratio 永远在死区外，loop 每秒两次写相同的值。跑 200 次后写次数 < 10，且确实停在硬上限；再给一个亮场景，下一次 step 必须重新写出来（不能卡在"未变化"）。
7. **`the_re_assert_writes_the_same_values_and_restarts_the_heartbeat`** — `current()` 返回当前值，且把它记为已写，下一个 tick 保持静默。
8. **`the_shutter_never_asks_for_a_longer_frame_than_the_sensor_has`** — 全黑场景跑到收敛，快门 ≤ 硬上限、模拟增益 ≤ 上限、数字增益 ≤ 上限。防止驱动拉伸帧时间掉帧率。
9. **`the_sensor_is_only_believed_about_a_value_it_was_not_already_at`** — `worth_reading_back(false, 600, 600)` 为假；`worth_reading_back(false, 601, 600)` 为真；`worth_reading_back(true, ...)` 永远为假。
10. **`the_loop_converges_rather_than_ringing`** — 用一个粗略传感器模型（luma 与控制量乘积成正比，再过 gamma ^(1/0.45)）模拟：画面起步暗 8 倍，60 步内收敛到死区内且 < 30 步。验证阻尼参数 `0.6` 不会振荡。

---

## 与其他模块的关系

- **`crate::pipeline`**：依赖 `CAPTURE_FORMAT`、`Frame`、`Frames`（tee 上的 raw 帧 tap）。loop 不打开摄像头，只读管线已经在跑的帧。
- **`crate::main`**：在管线启动后调用 `exposure::spawn(camera.device, frames, camera.exposure, camera.analogue_gain)`，传入的是 `v4l2src` 已经通过 `extra-controls` 写到传感器的初始值——所以 loop 从屏幕上已经有的画面起步。`--no-auto-exposure` 时不启动。
- **`rkaiq_3A_server`（外部进程）**：本模块存在的原因。它做白平衡、颜色矩阵、gamma、降噪，但其 AE 只在 stream-start 收敛一次。本模块接管"持续调节"这一半。
- **`v4l-utils`（`v4l2-ctl`）**：实际写入/读取传感器的工具，进程外调用。
- **duck 检测器**：与 AE 共享 tee 的 raw 分支，但互不干扰——检测器只读帧，AE 只读帧 + 写传感器。

---

## 设计哲学小结

整段代码的注释反复在讲同一件事：**在一个被其他进程、驱动、启动顺序、错过事件反复抢写的传感器上，闭环 loop 的敌人不是算法本身，而是"沉默地停在某个值上还自以为活着"**。所以：

- 周期性 re-assert 假设可能已被推翻；
- 周期性 info 报告证明 luma 在变；
- 第一次写成功后做一次 read-back，且只在请求值 ≠ 启动钉住值时做；
- 格式不匹配时 loud error；
- 失败一次之后降级为 debug，避免淹没日志；
- 死区 + 相同值去重，防止在不可达设定点上空转烧 CPU；
- 阻尼指数 0.6 补偿 gamma 曲线，防止过冲振荡。
#（注：内容由AI生成）
