# voices.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\voices.rs`

## 核心设计决策

每个 tag 的配方——Python `voices.py`，关键处逐行移植。配方用人格特征作画——音高中心、register、滑音偏向、谐波倾斜/共振峰、呱噪感、warble——所以**同一**配方在两个不同种子上给出两只明显不同的鸭子。

## 函数分析

- `attack(p, dur, snappy)` —— 起音时间，由 `attack_sharpness` 调制。`snappy=1` 用于打击乐配方。
- `voice(p, t, freq, rng, am_scale, breath_scale)` —— 共享核心：谐波振荡器 + 颤音 + 抖动 +（可选）AM 嗡嗡 + 呼吸。
- `alarm` —— 报警声，高于中心但保持在 honk 范围内；含随 brightness 缩放的 crackle。
- `greet_syllable` / `greet` —— 问候声，`glide_bias` 翻转轮廓（正=上升，负=下降）；40% 概率为双声"wak-wak"。
- `inquire` —— 询问声，总是上升（疑问句）。
- `peck` —— 啄食声，总是偏低；含随 `attack_sharpness` 缩放的 click。
- `chirp_syllable` / `chirp` —— 嘴触发声，变体循环四种不同形状（上升、下降、颤音、双声）。
- `coo` —— 咕咕声，远低于中心，呼吸更多、调制更慢、无嗡嗡。
- `wheee_segments(p, variant) -> (start, loop, end)` —— 滑行触发的 joy ride 的 (start, loop, end)。**渲染为一个连续主信号再切片**，使抖动/呼吸/wobble 跨 start→loop 切口无缝衔接。loop 文件尾部与 loop 起点前的样本交叉淡入淡出，使循环播放无咔嗒声。
- `wheee(p, variant)` —— 一次完整滑行：start + 两遍 loop + end。

## 单元测试描述

- `every_recipe_renders_sane_audio` —— 所有配方对一组种子和变体产生有限、归一化、非平凡的缓冲区（峰值在 [0.3, 1.0]）。
- `the_wheee_loop_wraps_without_a_click` —— wheee 循环无缝循环，loop[0] 与 loop[last] 的阶跃 < 0.2。

## 关键摘要

voices.rs 是 7 个 tag 的声音配方实现。每个配方都是人格特征的函数，所以同配方不同种子产生不同鸭子。`wheee_segments` 的连续主信号 + 交叉淡入淡出设计保证循环播放无缝衔接。所有配方输出都被测试验证为有限且在合理电平范围内。
