# chorale/mod.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\chorale\mod.rs`

## 核心设计决策

几只鸭子同唱一件事：乐谱、谁唱什么、以及混音。这是鸭子合唱的*音乐*半部分，刻意与真实硬件上困难的部分（互相发现、时钟同步）分离。它在笔记本上离线渲染合奏，使编曲在任何包在两只鸭子之间发送之前就能被评判。

### 身份是音色，不是音高

每只鸭子的声音由其 SoC 序列号派生，变化最大的是 `pitch_center_hz`。让其偏移*音符*会破坏作品：四只鸭子各按自己的调音唱一个和弦就是四只互不同调的鸭子——拍频而非和声。所以音符是绝对的，来自一个共享参考（`A4_HZ=440.0`，平均律），每只鸭子保留其他一切：谐波权重、共振峰、鼻音、呼吸，以及被驯服的颤音。register 用于**分配角色**（最低的鸭子唱低音）和选择作品落在哪调。

### 完全同步的合唱是错误目标

四个声音同一样本起音、持同一频率听起来不像合唱团，像一个带厚音栓的管风琴。使合奏成为合奏的是成员*几乎*在一起、*几乎*同调：几音分的音高散布和几十毫秒的起音散布。两者都有意添加，由每只鸭子的种子派生。这也是真实鸭子时钟同步需要多紧的答案：目标是 ±20 ms 而非 ±1 ms。

## 模块结构

- `pub mod beat` —— 无共享时钟的多鸭同步
- `pub mod midi` —— MIDI 文件导入
- `pub mod text` —— 文本乐谱格式

## 类型分析

- `Part`（Bass/Tenor/Alto/Soprano）—— 四个声部，低到高。`ensemble(n)` 返回 n 只鸭子唱的声部集合，**嵌套**：`{S} ⊂ {B,S} ⊂ {B,A,S} ⊂ {B,T,A,S}`，使鸭子能加入已在唱的组而无需任何人换声部。
- `Note` —— 一个声部的音符：part、start_beat、beats、midi、level（0..1 响度）、vowel。
- `Vowel`（Ah/Eh/Ee/Oh/Oo/Mm）—— 唱的元音：嘴开度 + 共振峰位置。嘴开度是与嘴舵机相同的数字，使元音可见又可听。`Mm` 是闭哼，更安静。
- `Voicing = [Option<u8>; 4]` —— 带静默声部的和弦配置。
- `Gesture` —— 乐谱的写作手势：`Chord`（齐唱）、`Build`（逐声部进入）、`Solo`（一声部移动其他持和弦）、`Rest`。
- `Score` —— 四声部乐谱：按声部、起始排序的音符 + bpm。**音符而非手势**——文本和 MIDI 两个前端在此汇合。
- `Singer` —— 合奏中的一只鸭子：personality、part、detune_cents（±5 音分散布）、onset_offset_s（±15 ms 起音散布）。
- `Options` —— 渲染选项：transpose、room（混响湿混合）、peak_dbfs、speaker_rolloff_hz（默认 300 Hz，鸭子扬声器实测）。

## 函数分析

- `cast(personalities) -> Vec<Singer>` —— 分配合奏：最低的鸭子唱最低声部。仅由人格确定性决定，真实鸭子无需协商即可达成。
- `seat(existing, newcomer) -> Singer` —— 为已在唱的组分配一只新鸭子。**已在唱的人不换声部**。
- `seat_by_register(held, pitch_center_hz) -> Part` —— 按 register 的座位形式，无线电用这个。
- `seat_for(personality, part) -> Singer` —— 为已知 part 生成 detune/onset 散布。
- `seat_all(registers) -> Vec<Part>` —— 从 register 名册分配所有声部——**按声音而非名册顺序**。
- `midi_hz(midi) -> f64` —— MIDI 音符的平均律频率。
- `render(score, singers, options) -> Vec<f32>` —— 将合奏渲染为单声道缓冲区。每只歌手整体渲染后求和（模拟四台独立机器），增益 `1/sqrt(n)`（不相关声音求和接近 sqrt(n)），可选 Schroeder 混响。
- `sing(score, singer, shift, total, options) -> Vec<f32>` —— 一只鸭子的声部，如从其扬声器输出。
- `reverb(buffer, wet)` —— 小房间混响：四个并行梳状滤波器 → 两个串联全通滤波器（Schroeder 排列）。
- `Score::from_gestures` —— 将手势编译为乐谱。连音合并：同声部同音高同元音跨手势边界合并为一个音符（声部写作方式）。元音变化重新起音（新音节），休止符打断连音。
- `Score::from_notes` —— 从原始音符列表构建乐谱（MIDI 导入方式），两种方式共享的连音合并。
- `Score::wistful()` —— 默认作品，从嵌入的 `scores/wistful.duckscore` 解析。
- `Score::duck_strut()` —— 另一首作品，通过 MIDI 前端，`scores/duck_strut.mid` 是真相源（MuseScore 编辑工作流）。

## 关键摘要

chorale/mod.rs 是鸭子合唱的音乐核心。设计哲学：身份是音色不是音高（绝对调音 + 共享 A4），不完全同步是目标（±20 ms 散布有意添加），声部分配嵌套使鸭子可无协商加入。两个前端（text + MIDI）汇合到统一的 `Score`（音符，非手势），连音合并保留声部写作语义。Schroeder 混响 + 扬声器 rolloff 使预览接近硬件实际听感。
