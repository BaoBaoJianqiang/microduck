# stream.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\stream.rs`

## 核心设计决策

鸭子的声音，持续打开：由实时参数驱动的逐块合成器。crate 中其他所有声音都是*配方*——从预先写下的频率曲线离线渲染整个发声。这对嘎嘎声合适，但对机器人必须在外部某物移动时*歌唱*的场合不对。ToF 特雷门琴的音高是手的距离，仅 15 次/秒且无法预知，所以频率曲线不能预先写下：必须在到达时积分。

**它是同一只鸭子**。不是音库的重采样也不是第二个声音：谐波权重来自 `Personality::harmonics`，颤音、抖动、呼吸和呱-AM 是那个人格的，`Stream::wheee` 应用与 joy-ride 配方相同的软化。一只鸭子的特雷门琴听起来像那只鸭子的 wheee，持续多久取决于手停留多久。

### 与离线路径的不同及原因

- **归一化是静态的**。配方将完成的缓冲区归一化到峰值；流没有完成的缓冲区，逐块归一化会随每块泵电平。增益从谐波权重导出——最坏情况同相和——`tanh` 软削波捕获 AM 和呼吸叠加的部分。
- **参数 slew**。深度每 ~67 ms 到达，嘴舵机更慢；在帧边界步进频率会产生可听的阶梯。每个参数以音频速率的单极点滤波器滑向目标。
- **抖动和呼吸滤波器是递归的**。离线版本卷积整个缓冲区（居中移动平均、归一化泄漏积分器）；此处两者都变成具有相同时间常数的单极点滤波器。特征相同，采样值不同。

## 常量分析

- `PITCH_TAU_S = 0.045` —— 音高时间常数，最慢（听者听作滑音）。
- `LEVEL_TAU_S = 0.030` —— 电平时间常数，足够快感觉键控，足够慢手离开帧是淡入淡出而非咔嗒。
- `OPEN_TAU_S = 0.060` —— 音色时间常数，慢于嘴舵机可移动速度，保持两者同步。
- `JITTER_TAU_S = 64.0 / 22_050.0` —— 抖动滤波器时间常数。
- `FORMANT_RANGE = 3.0` —— 元音可移动共振峰的谐波距离。
- `OPEN_TILT_LIFT = 0.8` —— 大张嘴提升谐波倾斜的量。
- `PEAK = 0.62` —— 静态增益的满刻度下余量。
- `ROLLOFF_DB_PER_OCTAVE = 12.0` —— 小扬声器低于有用范围的衰减斜率。
- `MAX_BASS_LIFT = 2.0` —— 扬声器 rolloff 放弃低频后的最大再分配增益。
- `REFRESH_SAMPLES = 256` —— 谐波权重重算频率（~5 ms）。

## 类型与函数分析

- `Stream` —— 可由慢速到达的参数以采样精度驱动的声音。`set(hz, level, open)` 设置目标，`block(out)` 渲染样本。
- `Stream::new(p, seed_tag, variant)` —— 这只鸭子的普通声音。
- `Stream::wheee(p, variant)` —— joy-ride 声音：颤音和呱-AM 减半，加兴奋 wobble。特雷门琴使用此构造器。
- `Stream::choral(p, variant)` —— 合奏声音：破坏调音的调制被驯服（颤音×0.30、呱噪×0.35、抖动×0.30、呼吸×0.15），但保留音色身份。
- `set_speaker_rolloff(hz)` —— 告诉声音扬声器实际能重现什么。**通过让基频更轻使低音更响**：硬币大小驱动器在几百赫兹以下几乎不产生，所以把权重从驱动器无法产生的谐波移到能产生的谐波上。音高存活因为音高不在基频中——残差音高。
- `set_formant_shift(harmonics)` —— 设置正在唱的元音：共振峰相对人格自身 `formant_n` 提升哪个谐波。
- `set_level(level)` —— 静音而不移动音高（音符结束而非倒下）。
- `set(hz, level, open)` —— 设置流滑向的目标。
- `is_silent()` —— 电平已淡至静音且无请求。
- `block(out)` —— 渲染下一个 `out.len()` 样本。块大小是调用者选择，不改变输出。
- `refresh_weights()` —— 当前 `open` 的谐波权重 + 保持和在范围内的增益。
- `range_hz(p) -> (lo, hi)` —— 鸭子的可演奏范围。
- `hz_at(p, position)` —— range 中 0..1 位置的频率，参数几何（半音线性）。

## 单元测试描述

- `block_size_does_not_change_the_signal` —— 块边界不在信号中（设计的核心性质）。
- `a_long_ride_stays_finite_and_bounded` —— 长时间骑行保持有限且 [−1,1]。
- `keying_the_level_fades_instead_of_clicking` —— 电平 0 是淡入淡出而非咔嗒。
- `an_open_mouth_is_brighter` —— 张嘴实际改变音色（高次谐波能量增加）。
- `a_speaker_rolloff_moves_the_bass_into_the_harmonics` —— 扬声器 rolloff 将低音能量移入谐波，同时保持音高间距。
- `the_breath_is_a_whisper_and_not_a_hiss` —— 呼吸是耳语不是嘶嘶声（流式版曾比离线版响 5 倍）。
- `the_bass_lift_cap_actually_caps` —— 低音提升上限真正生效。
- `a_note_below_everything_is_not_amplified_into_distortion` —— 驱动器完全无法产生的音符保持安静。
- `the_pitch_map_is_linear_in_semitones` —— 音高映射在半音上线性。
- `the_range_follows_the_personality` —— 范围跟随人格。

## 关键摘要

stream.rs 实现了实时流式合成器，与离线配方是**同一只鸭子**的声音。核心设计：参数 slew 消除 15 Hz ToF 输入的阶梯感、静态增益 + tanh 软削波替代逐块归一化、扬声器 rolloff 通过残差音高原理把低音能量移入可重现谐波。块大小不影响信号是整个设计的基石性质，被测试严密守护。
