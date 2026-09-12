# features.rs 文件解析

## 文件位置

`d:\microduck\pet-detect\src\bin\features.rs`

## 核心设计决策

`pet-features` 是**训练侧的特征提取二进制**，把 WAV 的 log-mel 特征以 f32 LE 原始字节写到 stdout。它存在的唯一理由是：**Python 训练脚本通过它提取特征，从而模型永远训练在机器人实际计算的东西上**——消除训练/推理的 mel 实现漂移风险。

用法（由 `training/train.py` 调用）：

```bash
pet-features <wav> [--stride <samples>]
```

## 输出格式

- 每个 1 秒窗口输出 `N_MELS * WINDOW_FRAMES`（4000）个 f32，LE 字节
- 形状 `[N_MELS, WINDOW_FRAMES]`，行主序
- 窗口默认 50% 重叠（`stride = WINDOW_SAMPLES/2`）
- stderr 打印块数与字节数统计

## 主流程

1. `load_wav_mono_16k` 加载并重采样到 16 kHz 单声道
2. `stride` 默认 `WINDOW_SAMPLES/2`
3. 滑动窗口：`start` 从 0 递增 `stride`，直到 `start + WINDOW_SAMPLES > samples.len()`
4. `MelExtractor::log_mel` 提取特征
5. 转为 LE 字节写 stdout
6. stderr 打印 `wrote N blocks of [40,100] f32 (16000 bytes each, hop=160 samples)`

## 关键摘要

`features.rs` 是训练/推理一致性契约的关键：训练脚本不自己实现 mel，而是调用这个二进制复用 `lib.rs` 的 `MelExtractor`，保证模型训练时看到的特征与机器人运行时逐字节相同。
