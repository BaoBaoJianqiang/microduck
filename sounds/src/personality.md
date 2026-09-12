# personality.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\personality.rs`

## 核心设计决策

`Personality` 从单个种子派生稳定的每只鸭子的声学特征。两个不同种子的机器人听起来明显不同；同一机器人跨运行一致。tag 内的变体重新抽取一个小子种子，避免鸭子听起来像卡住的录音。

特征集刻意广泛：register（八度偏移）、谐波倾斜、共振峰强调、滑音偏向、呱噪感——每一个本身都足以让两个种子感觉像不同的生物。

所有字段都是种子的稳定函数。`Copy`，因为配方按 tag 软化其副本（对应 Python 的 `dataclasses.replace`）。

## 类型分析

`Personality` 结构体（`Debug, Clone, Copy`），字段分四组：

**音高（pitch）**：
- `pitch_center_hz` —— 中心频率
- `register: f64`（-1..+1）—— 额外八度偏移
- `pitch_spread: f64`（0..1）—— 滑音戏剧性
- `glide_bias: f64`（-1..+1）—— 负=下降，正=上升

**音色（timbre）**：
- `brightness`（0..1）—— 谐波滚降，1=明亮/嗡嗡
- `tilt`（1.4..2.8）—— 谐波衰减指数，越高越暗
- `nasal`（0..1）—— 第 2/3 谐波强调
- `harmonic_skew`（-1..+1）—— 负=仅奇次（方波感），正=偶次倾向
- `formant_n: usize`（1..5）—— 共振峰提升的谐波序号
- `formant_gain`（0..1.5）—— 共振峰强度

**调制（modulation）**：
- `vibrato_rate_hz`、`vibrato_depth`（0..0.7 半音）
- `jitter_depth`（0..0.4 半音）—— 随机音高抖动
- `breath`（0..0.35）—— 噪声混合
- `quackiness`（0.2..1）—— 纯音 vs AM 嗡嗡混合
- `am_rate_hz`（18..55）、`am_depth`（0..0.7）
- `warble_hz`（7..18）、`warble_depth`（0..1.5 半音）

**节奏（timing）**：
- `attack_sharpness`（0..1）—— 0=柔和垫音，1=干脆
- `speed`（0.8..1.25）—— 全局节奏乘数

## 函数分析

- `from_seed(seed) -> Self` —— 用 `Rng::from_seed` 派生所有特征。register 为双峰分布（`[-1,0,0,1]` 选择 + 均匀扰动），整体种群偏低（鸭子/蟾蜍音域）。
- `variant_rng(tag, variant) -> Rng` —— 稳定的每 (种子, tag, variant) RNG，用于子随机化。使用 CRC-32 而非标准库 hasher（后者每进程加盐，会在每次音库重新生成时重抽变体）。
- `harmonics() -> Vec<f64>` —— 主振荡器的人格形谐波权重（7 个）。组合 tilt（整体滚降）、brightness（提升高频尾）、nasal（提升第 2/3 次）、harmonic_skew（奇偶偏好）、在一个选定谐波上的共振峰凸起。保证 f0 占主导（`weights[0] >= 0.7`）。

## 单元测试描述

- `a_seed_is_a_stable_identity` —— 同一种子永远是同一只鸭子，不同种子不同。
- `traits_stay_in_their_ranges` —— 500 个种子验证所有特征落在文档范围内。
- `variant_rng_is_stable_and_distinct` —— 变体 RNG 跨调用稳定、跨变体/跨 tag 不同。

## 关键摘要

personality.rs 是声音身份的核心：单一种子 → 20+ 声学特征的确定性映射。所有特征范围都被测试钉死。CRC-32（而非 std hasher）保证变体 RNG 跨进程稳定，因为鸭子的声音是派生的，RNG 本身就是声音。
