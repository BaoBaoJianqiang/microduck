# lib.rs 文件解析

## 文件位置

`d:\microduck\pet-detect\src\lib.rs`

## 核心设计决策

`pet-detect` 的库层是一个**纯音频特征提取 + 流式推理分类器**，回答"机器人头部是否被挠"这一个问题。

核心设计要点：

1. **特征路径是训练契约**。mel 滤波器组、窗长、跳数、N_FFT、归一化方式全部来自 `apirrone/microduck_pet_detect` 原样移植；任何数字变动都意味着需要重新训练。为此专门提供 `pet-features` 二进制，让训练脚本通过它提取特征，确保训练侧与推理侧走同一段 Rust 代码。

2. **流式检测器带滞回**。`PettingDetector` 持续推送采样，以 `enter_threshold`/`exit_threshold` 双阈值切换 `Petting` 状态，输出 `Start`/`End` 事件。退出阈值（0.85）高于直觉值，因为即使环境噪声也维持在 p≈0.7，真正停止挠头才会跌破 0.85。

3. **单线程推理**。20 KB 模型用默认的"每核一个 intra-op worker"只会在线程同步上浪费 CPU，因此显式 `with_intra_threads(1)` + `with_inter_threads(1)`。

4. **mel 滤波器组存为稀疏结构** `Vec<Vec<(bin, weight)>>`，只保留非零权重，避免每帧遍历 257 个 FFT bin。

## 常量分析

- `SAMPLE_RATE = 16_000`：16 kHz 单声道采样率
- `N_FFT = 512`：FFT 点数
- `HOP = 160`：帧移（10 ms）
- `WIN = 400`：窗长（25 ms，汉宁窗）
- `N_MELS = 40`：40 个 mel 频带
- `WINDOW_FRAMES = 100`：1 秒窗口对应 100 帧
- `WINDOW_SAMPLES = 16_240`：窗口样本数 = (100-1)×160 + 400
- `FMIN = 0.0` / `FMAX = 8_000.0`：mel 频率范围
- `LOG_EPS = 1e-6`：对数谱底，避免 log(0)

## 类型与函数分析

### `MelExtractor`

log-mel 特征提取器。`new()` 预规划 FFT、生成汉宁窗、构建 mel 滤波器组。`log_mel(samples)` 对 1 秒窗口逐帧：加窗 → 零填充到 N_FFT → FFT → 功率谱 → mel 滤波 → 取对数，输出 `[N_MELS * WINDOW_FRAMES]` 行主序展平向量。

### `hz_to_mel` / `mel_to_hz`

标准 mel 标度互转公式（2595·log10(1+f/700)）。

### `build_mel_filterbank`

构建三角 mel 滤波器组。在 mel 标度上等距取 `n_mels+2` 个点，每个频带由 (lo, ctr, hi) 形成三角窗，只保留权重 >0 的 (bin, weight) 对。

### `load_wav_mono_16k`

加载 WAV：支持 int/float 采样格式、立体声下混为单声道、非 16 kHz 用线性插值重采样到 16 kHz。

### `resample_linear`

线性插值重采样。按输出索引反查输入位置，取相邻两样本加权。

### `PettingEvent`

`{ Start, End }`，状态迁移事件。

### `PettingDetector`

流式检测器。内部维护环形缓冲 `ring`、`samples_until_infer`（下次推理所需累积样本数）、`stride`（推理窗口间隔）。`push_samples` 将新采样追加到 ring，每当累积够一个窗口就提取 mel、跑 ONNX 推理、按滞回阈值判定状态迁移。推理后 `samples_until_infer += stride`。环形缓冲上限为 4×窗口，超出则丢弃旧数据并同步修正 `samples_until_infer`。

### `PettingDetectorConfig`

- `stride`：推理间隔（默认 `WINDOW_SAMPLES/4` ≈ 250 ms）
- `enter_threshold = 0.95`：进入阈值
- `exit_threshold = 0.85`：退出阈值

### `i16_to_f32`

`arecord -f S16_LE` 产出的 i16 LE 样本转为 [-1, 1] 的 f32。

## 单元测试

- `the_feature_contract_is_pinned`：钉死 `WINDOW_SAMPLES=16240`、`N_MELS*WINDOW_FRAMES=4000`、40 个滤波器非空——特征布局变动即测试失败，强制重新训练。
- `log_mel_reacts_to_signal`：静音应输出全底（`log(LOG_EPS)`）；440 Hz 满幅正弦应有频带 >0，验证特征提取对信号有响应。

## 关键摘要

`lib.rs` 实现了"头部挠痒检测"的全部计算核心：40 频带 log-mel 特征提取（严格复现训练契约）+ ONNX 流式二分类 + 双阈值滞回状态机。所有数值常量被测试钉死，任何布局变更都会触发失败以提醒重新训练。模型仅 20 KB，单线程推理即可。
