# synth.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\synth.rs`

## 核心设计决策

DSP 原语——Python `synth.py`，48 kHz。原版渲染 22.05 kHz，安装程序用 ffmpeg 重采样所有文件，因为 Radxa 的 I²S 时钟树固定在 48k 系列（44.1k 系列速率播放偏 ~9%）。原生 48 kHz 渲染移除了重采样步骤和 ffmpeg 依赖。两个原语有在 22.05 kHz 调参的采样率相关常量——抖动平滑窗口和粉噪声极点——都按 `SR / 22050` 重新缩放，保持**时域**特征而非采样计数。

## 常量分析

- `SR: u32 = 48_000` —— 输出采样率。
- `TUNED_SR: f64 = 22_050.0` —— Python 配方调参时的速率，仅用于重新缩放两个基于采样计数的常量。

## 函数分析

- `t_axis(duration_s)` —— 一段时间的时间轴，每样本一个条目。
- `lerp(t, points)` —— 通过 (time_s, value) 点的分段线性曲线，numpy `interp`，范围外钳制到首/末值。
- `expdecay(t, attack_s, decay_s)` —— 快起音、指数衰减包络，峰值 ~1.0。
- `bell(t, attack_s, release_s)` —— 柔起音、平台、柔释放。
- `phase_from_freq(freq)` —— 将瞬时频率积分为相位（弧度）。
- `harmonic_osc(phase, weights)` —— 每个谐波 n 的 `sin(n·phase)·weight` 之和。
- `vibrato(t, rate_hz, depth_semitones, phase)` —— 慢速 LFO 的音高乘数。
- `jitter(t, depth_semitones, rng)` —— 随机音高抖动，平滑白噪声。窗口按 `64 * SR / 22050` 缩放。
- `moving_average_same(x, k)` —— numpy `convolve(x, ones(k)/k, mode="same")`，居中移动平均，零填充，滑动窗口 O(n) 实现。
- `pink_noise(n, rng)` —— 白噪声上的泄漏积分器，极点 `0.985^(22050/SR)` 重新缩放。
- `click(n, rng, length)` —— 短瞬态咔嗒声，`peck` 的起音。
- `normalise(x, peak_dbfs)` —— 缩放到 dBFS 峰值电平。
- `to_i16(x)` —— f32 → i16（钳制到 [-32768, 32767]）。

## 单元测试描述

- `lerp_matches_interp_semantics` —— lerp 钳制超出最后一点的值。
- `moving_average_matches_numpy_same_mode` —— 移动平均与 numpy `'same'` 模式一致（奇/偶窗口）。
- `normalise_hits_the_target_peak` —— 归一化达到目标峰值。
- `envelopes_stay_in_unit_range` —— 包络在 [0,1] 内。

## 关键摘要

synth.rs 是声音 DSP 的构建块：时间轴、包络、相位积分、谐波振荡器、颤音、抖动、粉噪声、咔嗒声、归一化。两个采样率相关常量（抖动窗口、粉噪声极点）从 22.05 kHz 重新缩放到 48 kHz，保持时域特征。所有数值语义都被 numpy 参考值钉死。
