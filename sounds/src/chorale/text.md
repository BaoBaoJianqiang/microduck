# chorale/text.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\chorale\text.rs`

## 核心设计决策

文本乐谱格式：可以在编辑器中写、在 review 中 diff 的合唱。乐谱起初是 Rust 中的 `Gesture` 字面量，对发货的一首作品还行，但对创作歌曲不对：每次改动都是重新编译，写音乐的人不会碰 `vec![Gesture::Chord { .. }]`。这是同一套手势的行导向文本文件——每行一个手势，音符名而非 MIDI 编号，对乐谱标记一次后持续有效的内容（力度、元音）用运行状态。

`scores/wistful.duckscore` 是语法自身的文档，并作为 `Score::wistful` 嵌入，所以发货作品和工作示例是同一文件，不可能漂移。

**不是什么**：不是通用音乐格式。无小节、无调号、无拍号、无超出 `Gesture` 表达能力的每声部节奏独立——四声部以四种不同节奏移动的乐谱无法在此写出，那正是 MIDI 导入器的用途。

## 类型与函数分析

- `ParseError { line, text, problem }` —— 哪里出问题，1-based 行号 + 行文本 + 问题描述。
- `parse(source) -> Result<Score, ParseError>` —— 解析乐谱。运行状态：`level`（默认 mf）、`vowel`（默认 Ah）。

**语法**：
- `name: <名字>`、`bpm: <速度>`
- `dynamic <ppp|pp|p|mp|mf|f|ff>` —— 设置运行力度
- `vowel <ah|eh|ee|oh|oo|mm>` —— 设置运行元音
- `rest <beats>` —— 休止
- `chord <beats> <bass> <tenor> <alto> <soprano>` —— 齐唱和弦
- `build <beats> stagger <beats> [top] <4 声部>` —— 逐声部进入，可选 `top` 从高到低
- `solo <part> under <4 声部> [hum <vowel>] sing <note>[:beats][/vowel] ...` —— 一声部移动

`#` 注释到行尾，空行忽略。

- `solo(words, running_vowel, level, fail)` —— 解析 solo 手势。
- `beats_of(word)` —— 解析拍数。
- `voicing_of(words)` —— 四个音名（低到高），`-` 为静默。
- `part_of(word)` —— 声部名解析。
- `midi_of(word) -> Option<Option<u8>>` —— 科学音高名 → MIDI。`Ok(None)` 是静默标记 `-`。C4 = 中央 C = MIDI 60（记谱编辑器约定）。

## 单元测试描述

- `the_embedded_default_score_parses` —— 嵌入的默认乐谱 `wistful` 可解析，bmp=58，四声部，力度和元音变化。
- `note_names_follow_the_editors_octave_numbering` —— C4=60、A4=69，升降号两种拼写，等音一致，大小写不敏感。
- `a_mark_holds_until_it_is_changed` —— 运行状态：标记一次持续到再次标记。
- `a_new_vowel_on_the_same_note_is_a_new_note` —— 同音高上新元音是新音符（重新起音）；力度变化是渐强（不重新起音）。
- `a_solo_hums_underneath_by_default` —— solo 下伴奏默认哼（Mm），`hum` 可覆盖。
- `a_build_reads_its_optional_direction` —— build 的可选 `top` 方向正确。
- `errors_say_which_line_and_what` —— 错误指明行号和问题，引用行文本。
- `comments_and_blank_lines_are_ignored` —— 注释和空行被忽略。

## 关键摘要

text.rs 提供了一个可在编辑器中书写和 diff 的合唱乐谱格式。核心设计：运行状态（力度、元音）标记一次持续有效；错误必须指明行号和内容；音符名使用记谱编辑器的 C4=中央 C 约定。与 MIDI 前端互补——文本用于手写合唱，MIDI 用于记谱编辑器能产生的任何东西。
