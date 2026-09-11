# `worker.rs` 解读 — pet_detect

## 概述

麦克风作为后台 worker：`arecord` 子进程 → 宠物分类器 + 环境声哨兵。

音频源是 `arecord` 子进程而非进程内 ALSA 绑定——与独立 `pet-detect` 二进制文档化的相同模式，不引入新的原生依赖。采集设备是单客户端的，所以所有分析麦克风的东西共享这一个流。

从原型的 `pet_worker.rs` 移植；`println!` 诊断变成了 `tracing`。

---

## SoundEvent 枚举

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SoundEvent {
    Noise,
    Voice,
}
```

环境声事件，来自宠物分类器消费的同一个流。纯 RMS 包络启发式——无 ML：

- **`Noise`**：尖锐瞬态（拍手、砰、门）——≤ ~0.38 s 的大声
- **`Voice`**：持续发声（语音、对鸭子叫）——最多 ~3 s 的大声。更长的是连续噪声（吸尘器、音乐），不发事件；自适应地板吸收它们

---

## SoundSentry

```rust
const SENTRY_FRAME: u32 = 512;  // 32 ms at 16 kHz
```

### 结构体

```rust
struct SoundSentry {
    floor: f32,              // 自适应环境地板
    frame_acc: f32,          // 当前帧能量累加
    frame_n: u32,            // 当前帧采样数
    in_event: bool,          // 是否在事件中
    event_frames: u32,       // 当前事件持续帧数
    quiet_frames: u32,       // 连续安静帧数
    cooldown_frames: u32,    // 冷却剩余帧数
    petting_hold_frames: u32,// 宠物保持剩余帧数
    event_peak: f32,         // 当前事件峰值 RMS
    floor_log_frames: u32,   // 距上次地板日志的帧数
}
```

### 自适应地板

```rust
// 环境地板只从非事件帧适应（τ ≈ 6 s），所以持续噪声（步态舵机、音乐）抬高门槛而非刷屏事件
self.floor = 0.995 * self.floor + 0.005 * rms;
```

τ ≈ 1/(0.005 × 31) ≈ 6.5 秒。地板只在非事件帧更新，所以持续噪声抬高门槛而非触发事件。

### 阈值

```rust
let on_thresh = (self.floor * 6.0).max(0.002);
let off_thresh = (self.floor * 3.0).max(0.0012);
```

绝对最小值存在只是为了近零地板的 6× 不在静音中触发——比率阈值做实际工作。样本是 ±1 归一化的；低采集增益让真实拍手远在绝对猜测之下。

### 事件分类

```rust
if self.event_frames > 94 {
    // > ~3 s: 连续噪声 — 吸收到地板，不发事件
    self.in_event = false;
    self.floor = self.floor.max(rms * 0.5);
    self.cooldown_frames = 31;
} else if self.quiet_frames >= 3 {
    self.in_event = false;
    self.cooldown_frames = 31; // ≥ 1 s between events
    if self.petting_hold_frames == 0 {
        let dur = self.event_frames - self.quiet_frames;
        // ≤ ~0.38 s = 瞬态（拍手 + 混响尾）；更长 = 发声
        let ev = if dur <= 12 { SoundEvent::Noise } else { SoundEvent::Voice };
        out.push(ev);
    }
}
```

| 持续时间 | 分类 |
|----------|------|
| ≤ 12 帧（~0.38 s） | Noise（拍手、砰） |
| 13–94 帧（~0.4–3 s） | Voice（语音、叫） |
| > 94 帧（~3 s） | 连续噪声，吸收到地板，不发事件 |

### 宠物保持期

```rust
if petting {
    self.petting_hold_frames = 31; // ~1 s hangover after petting
} else if self.petting_hold_frames > 0 {
    self.petting_hold_frames -= 1;
}
```

宠物声在这个麦克风上很大（它实际上是头部抓挠的接触式麦克风），所以分类器报告宠物时事件被抑制（+1 s 残留）。

---

## PetConfig

```rust
pub struct PetConfig {
    pub alsa_device: String,      // ALSA 采集设备
    pub model_path: PathBuf,     // ONNX 模型路径
    pub enter_threshold: f32,    // 宠物开始阈值
    pub exit_threshold: f32,      // 宠物结束阈值
}
```

默认值：
- `alsa_device`: `"plughw:aic3104,0"`（TLV320AIC3x codec）
- `model_path`: `"/opt/robot/daemon/current/models/pet_detect.onnx"`

---

## PetHandle

```rust
pub struct PetHandle {
    rx: Receiver<PettingEvent>,
    rx_sound: Receiver<SoundEvent>,
    shutdown: Arc<AtomicBool>,
    join: Option<JoinHandle<()>>,
}
```

### spawn

```rust
pub fn spawn(config: PetConfig) -> Result<Self> {
    // 在主线程构建 detector，不在线程中——缺失模型或运行时是调用者看到的错误，
    // 而非 worker 第一次呼吸就安静死亡
    let detector = std::panic::catch_unwind(|| {
        PettingDetector::new(&config.model_path, ...)
    }).unwrap_or_else(|_| Err(anyhow!("the ONNX runtime is not loadable")))?;
    // ... spawn worker 线程
}
```

**关键设计**：detector 在主线程构建，不在 worker 线程中。理由：
1. 缺失模型或运行时是调用者看到的错误，而非 worker 安静死亡
2. `ort` 在它认为不可恢复的失败上 panic——用 `catch_unwind` 捕获，缺失 libonnxruntime 读作"无麦克风 worker"，而非死守护进程

### 非阻塞接收

```rust
pub fn try_recv_event(&self) -> Option<PettingEvent> { self.rx.try_recv().ok() }
pub fn try_recv_sound(&self) -> Option<SoundEvent> { self.rx_sound.try_recv().ok() }
```

每个控制 tick 调用一次。非阻塞。

### shutdown

```rust
pub fn shutdown(mut self) {
    self.shutdown.store(true, Ordering::Release);
    if let Some(j) = self.join.take() {
        let _ = j.join();
    }
}
```

设置原子标志，等待线程退出。

---

## 重启退避

```rust
const RESTART_BACKOFF_MIN: Duration = Duration::from_millis(250);
const RESTART_BACKOFF_MAX: Duration = Duration::from_secs(30);
const RESTART_HEALTHY: Duration = Duration::from_secs(5);
const RESTART_QUIET_AFTER: u32 = 5;
```

### 为什么需要退避

> A board where `arecord` exists but the codec does not — `configure_audio` fails soft at every step, so a failed DKMS build leaves exactly that — makes `arecord` exit immediately on every spawn. Without a backoff on *that* path (the original only slept when the `arecord` binary itself was missing) the worker fork/execs as fast as the CPU allows for the life of the daemon, with a `warn!` per iteration into the journal.

板子上 `arecord` 存在但 codec 不存在——`configure_audio` 每步软失败，失败的 DKMS 构建正好留下这个状态——`arecord` 每次 spawn 立即退出。没有这条路径上的退避（原始版本只在 `arecord` 二进制本身缺失时 sleep），worker 会以 CPU 允许的最快速度 fork/exec，每次迭代一条 `warn!` 进 journal。

### 退避算法

```rust
failures = failures.saturating_add(1);
let backoff = RESTART_BACKOFF_MIN
    .saturating_mul(1u32 << (failures - 1).min(8))
    .min(RESTART_BACKOFF_MAX);
```

从 250 ms 开始翻倍，上限 30 s：
- 第 1 次：250 ms
- 第 2 次：500 ms
- 第 3 次：1 s
- ...
- 第 9 次及以后：30 s（上限）

### 健康重置

```rust
if started.elapsed() >= RESTART_HEALTHY {
    failures = 0;
    continue;
}
```

运行了 5 秒以上的捕获不是这个退避要处理的失败，延迟重置。

### 安静化

```rust
if failures == RESTART_QUIET_AFTER {
    tracing::warn!("pet worker: the mic will not stay up — retrying quietly from here");
}
```

连续 5 次立即失败后，每次重启的日志降为 `debug`。退避此时已到上限，警告已说过——worker 继续尝试（codec 回来应该被听到）但不再叙述。

### 分片 sleep

```rust
fn sleep_unless_shutdown(total: Duration, shutdown: &Arc<AtomicBool>) -> bool {
    const SLICE: Duration = Duration::from_millis(100);
    // 每 100 ms 检查一次 shutdown
}
```

> Sliced, because `shutdown()` joins this thread: a 30 s sleep would be 30 s of robotd not exiting.

分片 sleep，因为 `shutdown()` join 这个线程：30 s sleep 会让 robotd 30 s 不退出。

---

## pump 函数

```rust
fn pump(
    child: &mut Child,
    detector: &mut PettingDetector,
    sentry: &mut SoundSentry,
    tx: &Sender<PettingEvent>,
    tx_sound: &Sender<SoundEvent>,
    shutdown: &Arc<AtomicBool>,
) -> Result<()> {
    let mut buf = [0u8; 4096];
    // 循环读取 arecord stdout
    // 转 i16 LE → f32
    // detector.push_samples() → PettingEvent
    // sentry.push() → SoundEvent
}
```

### 为什么 sentry 从 detector 读 petting 状态

> Read off the detector rather than tracked here: a Start..End session spans many seconds, and `arecord` can flap inside one. The detector survives a restart (it is owned by `worker_loop`), so a local copy would come back `false` mid-session — and no second `Start` would ever re-arm it, leaving the sentry emitting head-scratch noise as `Voice` events until the probability finally fell below the exit threshold.

从 detector 读 petting 状态而非在此追踪：一个 Start..End 会话跨多秒，`arecord` 可能在其中抖动。detector 在重启中存活（它由 `worker_loop` 拥有），所以本地副本会在会话中途回到 `false`——没有第二个 `Start` 会重新武装它，让 sentry 把头部抓挠噪声作为 `Voice` 事件发出，直到概率最终降到退出阈值以下。

---

## spawn_arecord

```rust
fn spawn_arecord(device: &str) -> Result<Child> {
    Ok(Command::new("arecord")
        .args(["-D", device, "-f", "S16_LE", "-r", "16000", "-c", "1", "-t", "raw"])
        .stdout(Stdio::piped())
        .stderr(Stdio::null())
        .spawn()?)
}
```

参数：
- `-D plughw:aic3104,0`：ALSA 设备
- `-f S16_LE`：16 位小端整数
- `-r 16000`：16 kHz 采样率
- `-c 1`：单声道
- `-t raw`：原始输出（无 WAV 头）

stderr 丢弃——arecord 的诊断对调试无用，且会污染 journal。

---

## 与其他文件的关系

- **`lib.rs`**：PettingDetector、PettingDetectorConfig、PettingEvent、i16_to_f32
- **`robotd`**：宿主进程，消费 PetHandle
- **`arecord`**：外部 ALSA 录音工具
- **`plughw:aic3104,0`**：TLV320AIC3x codec 设备
- **`models/pet_detect.onnx`**：模型文件

---

## 关键踩坑点总结

1. **detector 在主线程构建而非 worker 线程**：缺失模型是调用者看到的错误，而非 worker 安静死亡。`ort` panic 用 `catch_unwind` 捕获。

2. **arecord 子进程而非进程内 ALSA**：不引入新的原生依赖。采集设备是单客户端的，所有分析共享这一个流。

3. **重启退避从 250 ms 翻倍到 30 s**：板子上 arecord 存在但 codec 不存在时，arecord 每次 spawn 立即退出。没有退避会以 CPU 允许的最快速度 fork/exec。

4. **健康重置**：运行 5 秒以上的捕获重置失败计数——正常重启不应该累积退避。

5. **安静化**：连续 5 次立即失败后日志降为 debug，避免 journal 刷屏。

6. **分片 sleep**：30 s sleep 分片为 100 ms，确保 shutdown 能及时响应——否则 robotd 退出要等 30 s。

7. **sentry 从 detector 读 petting 状态**：不在 pump 中本地追踪——arecord 抖动时 detector 存活但本地副本会重置，导致头部抓挠噪声被误报为 Voice。

8. **自适应地板只在非事件帧更新**：持续噪声（步态舵机、音乐）抬高门槛而非刷屏事件。τ ≈ 6.5 s。

9. **Noise vs Voice 按持续时间分类**：≤0.38 s 是瞬态（拍手），0.4–3 s 是发声（语音），>3 s 是连续噪声（不发事件）。

10. **petting 期间抑制 sentry 事件**：头部抓挠在接触式麦克风上很大，+1 s 残留。
#（注：内容由AI生成）
