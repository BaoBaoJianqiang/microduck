# features.rs（pet-features log-mel 特征提取器）解读与架构梳理

> 分析对象：`features.rs`（51 行），`pet-features` 二进制——WAV 文件的 log-mel 特征提取器。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `pet-features`——把 WAV 文件的 log-mel 特征以 f32 LE 输出到 stdout。这是训练/推理一致性契约的**训练侧**：Python 训练脚本通过这个二进制提取特征，所以模型永远在机器人实际计算的特征上训练。

**设计要点**：
- **训练/推理 parity**：特征提取代码只有一份（在 `pet_detect` crate 的 `MelExtractor`）。训练用 `pet-features` 调用它，推理用 `pet-detect` 调用它——两边完全一致。
- **输出格式**：每个 block 输出 `[N_MELS, WINDOW_FRAMES]` 个 f32（little-endian），无文件头、无元数据——直接是 raw float bytes。
- **1 秒窗口，50% 重叠**：WINDOW_SAMPLES / 2 = stride。
- **任意采样率/声道**：自动重采样到 16kHz 单声道。
- **stderr 报告**：block 数量、每 block 字节数、hop——不污染 stdout 的 raw float 流。

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | 输出 f32 LE，shape [N_MELS, WINDOW_FRAMES] | L2-3 | confirmed |
| F2 | 1 秒窗口，默认 50% 重叠 | L3, L31 | confirmed |
| F3 | Python 训练脚本通过此二进制提取特征 | L5-6 | confirmed |
| F4 | 任意采样率/声道 → 重采样 16kHz mono | L21 | confirmed |
| F5 | stride 可选覆盖 | L25 | confirmed |
| F6 | 无文件头 raw float 输出 | L40-41 | confirmed |
| F7 | stderr 报告统计 | L45-48 | confirmed |
| F8 | 依赖 pet_detect::MelExtractor | L12 | confirmed |

---

## 三、数据流

```
WAV 文件（任意格式）
  → load_wav_mono_16k()  # 自动重采样
  → samples: Vec<f32> (16kHz mono)
  ↓
滑动窗口:
  start = 0, stride = WINDOW_SAMPLES / 2
  while start + WINDOW_SAMPLES <= len:
    mel = MelExtractor.log_mel(samples[start..start+WINDOW_SAMPLES])
    → [N_MELS, WINDOW_FRAMES] f32
    → flat_map → LE bytes
    → stdout
    start += stride
  ↓
stderr: "wrote N blocks of [M,F] f32 (B bytes each, hop=H samples)"
```

---

## 四、关键设计决策

### 4.1 为什么训练用此二进制而非 Python 特征提取

"the Python training script extracts features THROUGH this binary, so the model always trains on exactly what the robot computes."

如果 Python 训练用 torchaudio/librosa 提取特征，Rust 推理用自己的 MelExtractor——两边的浮点精度、窗函数、归一化都可能有细微差异。模型在"Python 特征"上训练，在"Rust 特征"上推理——精度损失。通过同一个二进制提取特征，parity 保证。

### 4.2 为什么 raw float 无文件头

stdout 是机器管道——Python 训练脚本直接 `np.frombuffer(raw_bytes, dtype=np.float32).reshape(-1, N_MELS, WINDOW_FRAMES)`。加文件头反而多此一举。

### 4.3 为什么统计走 stderr

stdout 必须是纯 raw float——任何文本输出都会被 Python 当成 float 解析失败。所以 block 计数走 stderr。

---

## 五、结论

### confirmed
- C1：WAV → log-mel 特征 f32 LE 输出到 stdout。
- C2：训练/推理 parity——Python 训练通过此二进制提取特征。
- C3：1 秒窗口 50% 重叠，自动重采样 16kHz mono。
- C4：raw float 无文件头，统计走 stderr。

### inferred
- I1：N_MELS 和 WINDOW_FRAMES 由 pet_detect crate 定义。
- I2：这是 offline 工具（训练时用），不在机器人上运行。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
