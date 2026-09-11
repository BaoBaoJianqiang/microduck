# `lib.rs` 解读 — pet_detect

## 概述

宠物检测：一个微型音频分类器，听板载麦克风上的头部抓挠声。

- **特征**：40 波段 log-mel 频谱图，1 秒窗口 @ 16 kHz 单声道
- **模型**：约 20 KB CNN（Conv → BN → ReLU → MaxPool → Conv → BN → ReLU → GAP → Linear），vendored 在 `models/pet_detect.onnx`
- **输出**：[`PettingEvent::Start`] / [`PettingEvent::End`]，带滞后

从 `apirrone/microduck_pet_detect` 逐数字移植：mel 布局是训练契约，`pet-features` 二进制存在正是为了让训练和推理共享这个文件。arecord worker 和环境声哨兵在 [`worker`] 中。

---

## 关键常量

```rust
pub const SAMPLE_RATE: usize = 16_000;
pub const N_FFT: usize = 512;
pub const HOP: usize = 160;      // 10 ms
pub const WIN: usize = 400;     // 25 ms
pub const N_MELS: usize = 40;
pub const WINDOW_FRAMES: usize = 100;  // 1.0 s
pub const WINDOW_SAMPLES: usize = (WINDOW_FRAMES - 1) * HOP + WIN;  // 16,240
pub const FMIN: f32 = 0.0;
pub const FMAX: f32 = 8_000.0;
pub const LOG_EPS: f32 = 1e-6;
```

### 特征参数解读

| 参数 | 值 | 说明 |
|------|-----|------|
| 采样率 | 16 kHz | 语音频段，足够抓挠声 |
| FFT 大小 | 512 | 32 ms 频率分辨率 |
| Hop | 160 | 10 ms 帧移 |
| 窗长 | 400 | 25 ms Hann 窗 |
| Mel 波段 | 40 | 标准配置 |
| 窗口帧数 | 100 | 1 秒音频 |
| 窗口采样数 | 16,240 | (100-1)×160 + 400 |
| 频率范围 | 0–8 kHz | 16 kHz 奈奎斯特 |

**为什么这些数字是训练契约**：mel 布局直接影响模型输入。改变任何一个参数（如 N_MELS 从 40 改为 32），模型输入形状就变了，必须重新训练。测试 `the_feature_contract_is_pinned` 固定了这些数字。

---

## MelExtractor

```rust
pub struct MelExtractor {
    fft: Arc<dyn Fft<f32>>,
    hann: Vec<f32>,
    mel_filters: Vec<Vec<(usize, f32)>>,
}
```

log-mel 提取器——模型训练时使用的确切特征路径。

### 设计：稀疏 mel 滤波器

```rust
/// Sparse: for each mel band, (fft_bin, weight).
mel_filters: Vec<Vec<(usize, f32)>>,
```

每个 mel 波段存储 `(fft_bin, weight)` 的稀疏列表，而非稠密矩阵。三角滤波器的大部分 bin 权重为零，稀疏表示节省内存和计算。

### log_mel 处理流程

对每个帧：加窗 + 零填充到 N_FFT → FFT → 功率谱（前 N_FFT/2+1 个 bin）→ 稀疏 mel 滤波 + log → 输出 `[N_MELS * WINDOW_FRAMES]` 行优先。

### Hann 窗

标准 Hann 窗（0.5 - 0.5·cos(2πn/(N-1))），25 ms 窗长。

### mel 滤波器组构建

标准 HTK mel 滤波器组：三角滤波器在 mel 尺度均匀分布。只保留权重 > 0 的 bin（稀疏）。

---

## load_wav_mono_16k

加载 WAV 文件，整数采样归一化到 [-1, 1]，立体声取平均下混为单声道，如果采样率不是 16 kHz 则线性重采样。用于测试和离线特征提取。

---

## PettingDetector

```rust
pub struct PettingDetector {
    session: Session,
    extractor: MelExtractor,
    ring: Vec<f32>,
    samples_until_infer: usize,
    stride: usize,
    is_petting: bool,
    enter_threshold: f32,
    exit_threshold: f32,
}
```

### 流式检测模型

推 16 kHz 单声道 f32 采样，在状态转换时接收 Start/End 事件。

### 滞后（hysteresis）

- 进入阈值：0.95（高）
- 退出阈值：0.85（低）

**为什么退出阈值比直觉低**：即使环境噪声在小训练集上也有 p ≈ 0.7，所以降到 0.85 以下干净地意味着抓挠确实停止了。

### 单线程 ONNX Runtime

```rust
let session = Session::builder()?
    .with_optimization_level(GraphOptimizationLevel::Level3)?
    .with_intra_threads(1)?
    .with_inter_threads(1)?
    .commit_from_file(model_path)?;
```

**故意单线程**：默认的每核一个 intra-op worker 会为 20 KB 模型 spawn 线程，在同步开销上烧 CPU。

### 推理循环

```rust
pub fn push_samples(&mut self, samples: &[f32]) -> Result<(Vec<PettingEvent>, Option<f32>)> {
    self.ring.extend_from_slice(samples);
    while self.ring.len() >= self.samples_until_infer {
        // 1. 取窗口
        // 2. log-mel 特征
        // 3. ONNX 推理
        // 4. 滞后判断
        // 5. 推进 stride
    }
    // 环形缓冲区上限
}
```

### 默认 stride

```rust
stride: WINDOW_SAMPLES / 4,  // ≈ 250 ms
```

每 250 ms 推理一次。更小 = 更低延迟但更多 CPU。

### 环形缓冲区管理

缓冲区上限为 2× 窗口大小，内存有界。

---

## i16_to_f32

i16 LE 采样（`arecord -f S16_LE` 的输出）转 f32 [-1, 1]。

---

## 测试

### the_feature_contract_is_pinned

固定特征契约：`WINDOW_SAMPLES = 16_240`、`N_MELS * WINDOW_FRAMES = 4_000`。这些数字变了意味着需要重新训练。

### log_mel_reacts_to_signal

静音必须产生全地板 log-mel；全幅正弦波必须不产生全地板。验证特征提取器对信号有反应。

---

## 与其他文件的关系

- **`worker.rs`**：后台 worker，调用 `PettingDetector::push_samples()`
- **`models/pet_detect.onnx`**：20 KB CNN 模型
- **`apirrone/microduck_pet_detect`**：原型来源
- **`pet-features` 二进制**：共享特征提取代码，用于训练数据提取
- **`robotd`**：宿主进程，消费 PettingEvent

---

## 关键踩坑点总结

1. **mel 布局是训练契约**：改变任何特征参数（N_MELS、N_FFT、HOP、WIN）意味着重新训练。测试固定了这些数字。

2. **ONNX Runtime 默认多线程是开销**：20 KB 模型不需要多线程，默认每核一个 worker 在同步开销上烧 CPU。必须显式设 `intra_threads=1`、`inter_threads=1`。

3. **滞后阈值非对称**：进入 0.95、退出 0.85。环境噪声在小训练集上也有 p ≈ 0.7，所以退出阈值不能太低。

4. **稀疏 mel 滤波器**：三角滤波器大部分 bin 权重为零，稀疏存储节省内存和计算。

5. **静音的 log-mel 不是负无穷**：用 `(s + LOG_EPS).ln()`，LOG_EPS=1e-6，避免 log(0)。

6. **stride 是延迟和 CPU 的权衡**：默认 ≈250 ms，更小 = 更低延迟但更多推理次数。
#（注：内容由AI生成）
