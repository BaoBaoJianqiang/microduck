# chorale/midi.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\chorale\midi.rs`

## 核心设计决策

标准 MIDI 文件输入，合唱输出——使记谱编辑器成为乐谱编辑器。这是真正解锁为鸭子写音乐的前端。文本格式适合手写合唱，但无法表达四声部以四种不同节奏移动。MIDI 文件可以，每个记谱编辑器和 DAW 都导出，且整个公共领域四声部曲目库（尤其是巴赫合唱曲）已有 MIDI。

**无依赖**。标准 MIDI 文件是长度前缀分块格式 + 变长 delta 时间，乐谱需要的子集是 note-on、note-off、tempo、track name——几百行代码，零供应链。

### 音轨如何变成声部

按**平均音高**而非音轨顺序——最低的一组音符唱低音。音轨顺序是显然规则但错两次：记谱编辑器按顶部谱表优先写（soprano, alto, tenor, bass，与 `Voicing` 列出的相反），DAW 文件可能有 tempo 轨、空轨或任意顺序。按音高排序无论谁产生的文件都正确，且与 `cast` 分配鸭子用同一规则。写了 "Soprano" 的音轨*名*优先相信（人写下的）。

单个复音音轨（钢琴谱表）按每个和弦内的音高排名拆分。

### 故意丢弃的

力度变成音符的力度（免费且正确）。其他一切——控制器、弯音、音色变化、第一个 tempo 之后的任何东西——都跳过：鸭子只有一个振荡器和一张嘴，节奏图必须被四台机器人经网络一致遵守，这是无法保证的承诺。有 tempo 变化的文件按第一个 tempo 唱，并报告该事实。

## 类型与函数分析

- `MidiError`（NotMidi/Truncated/SmpteTiming/NoNotes）—— 解析错误。
- `Import { score, tracks, casting, dropped }` —— 文件中除音符外的内容：音轨名、每声部如何决定、被跳过的东西。
- `Raw { group, start_tick, end_tick, midi, velocity }` —— 文件描述的原始音符，分配声部前。
- `parse(bytes) -> Result<Import, MidiError>` —— 读取标准 MIDI 文件。
- `push(...)` —— 记录一个原始音符，按 (track, channel) 分组。
- `assign(raws, names, ticks_per_beat)` —— 将原始音符组转为四个声部。
- `split_by_rank(raws)` —— 单个复音组按和弦内音高排名拆分为声部。
- `part_named(name) -> Option<Part>` —— 人可能在谱表上打的声部名。
- `Reader` —— 文件游标，每次读取都边界检查（解析下载的文件字节，截断应是错误不是 panic）。
  - `varint()` —— MIDI 变长量：每字节 7 位，高位表示"还有"。

## 单元测试描述

- `a_score_exported_top_staff_first_still_puts_the_bass_on_the_bass` —— MuseScore 导出的顶部谱表优先文件，低音仍在低音声部。
- `a_named_track_is_believed_over_its_pitches` —— 命名音轨优先于其音高。
- `a_single_polyphonic_track_is_split_into_voices` —— 单个复音音轨按音高排名拆分。
- `timing_and_velocity_survive_the_trip` —— tick→拍通过文件自身 division，力度→力度。
- `a_tempo_change_is_reported_rather_than_hidden` —— tempo 变化被丢弃并报告，按第一个 tempo 唱。
- `bad_files_are_errors_and_never_panics` —— 坏文件是错误绝不 panic（截断文件的每个前缀）。
- `running_status_is_understood` —— Running status（note-on 的状态字节由前一事件隐含）正确解析。
- `more_than_four_voices_says_what_it_left_out` —— 超过四声部报告丢弃了什么。

## 关键摘要

midi.rs 是一个零依赖的标准 MIDI 文件解析器，使记谱编辑器成为鸭子合唱的乐谱编辑器。核心设计：按平均音高（非音轨顺序）分配声部，与 `cast` 用同一规则；力度变力度，其他全部丢弃并报告；所有读取边界检查防止 panic。`scores/duck_strut.mid` 是此工作流的真相源（在 MuseScore 中编辑并提交导出）。
