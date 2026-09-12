# main.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\main.rs`

## 核心设计决策

`sounds` 二进制——渲染和试听机器人的语音库。`ensure-bank` 是发布版本 postinstall 钩子运行的命令：它将原型的 `generate_sounds.sh`（venv + numpy + ffmpeg）替换为一个二进制，就地、幂等地渲染每只机器人的音库。其他都是基准工具：在发货前听一个种子、手动重新生成音库、打印一个声音背后的人格。

## 子命令分析

- `Show [--seed]` —— 打印种子背后的人格特征。
- `Render <tag> <out> [--seed] [--variant]` —— 将一个声音渲染为 wav 文件。
- `RenderAll <out_dir> [--seed]` —— 将所有 tag×variant 渲染到目录（robotd 播放的布局）。
- `Play <tag> [--seed] [--variant] [--device]` —— 合成一个声音并通过 aplay 播放。默认 ALSA 设备 `plughw:aic3104`（机器人 codec）。
- `Theremin [--out] [--seed] [--variant] [--device]` —— 试听实时特雷门琴声音：脚本化的手部扫过流式合成器。特雷门琴的音高是手的距离，工作台上没有手，所以播放手势本身——接近、保持、晃动、后退——以 ToF 实际交付的帧率驱动 `Stream`。
- `Chorale [--voices] [--seeds] [--score] [--out] [--bpm] [--transpose] [--room] [--rolloff] [--device]` —— 笔记本上的鸭子合唱：几只鸭子分四声部唱一首作品。离线渲染合奏并混音。每只鸭子保留自己的音色；*音符*是绝对的。
- `EnsureBank <dir> [--force] [--seed]` —— 确保此机器人的语音库存在且最新——不存在则渲染。幂等：标记记录种子和音库版本，匹配的音库不动。

## 函数分析

- `resolve_seed(seed)` —— 给定或从硬件派生种子。
- `show(p)` —— 打印所有人格特征。
- `play_pcm(buf, device)` —— 通过 `aplay` 播放，机器人设备失败则回退默认设备。
- `theremin_sweep(p, variant)` —— 脚本化手部手势：6 段（接近、保持、晃动、后退、手离开），每 15 Hz 边界更新参数，模拟真实 ToF 帧率。
- `load_score(path)` —— 按扩展名选择前端：`.mid`/`.midi` 走 MIDI 导入器，其他走文本解析器。报告决定和丢弃内容。

## EnsureBank 设计要点

1. 标记格式 `{seed}:{BANK_VERSION}`，写入 `<dir>/.seed`。
2. 标记匹配则跳过（幂等，每次安装运行）。
3. **暂存渲染再原子替换**：渲染到 `<dir>.new`，写标记，删除旧目录，`rename` 替换。中途断电不会留下标记声称完整的半音库。

## 关键摘要

main.rs 是 sounds crate 的 CLI 入口，提供 7 个子命令。核心是 `ensure-bank`（幂等、原子的音库生成，发布钩子使用）和 `chorale`（离线多鸭合奏渲染用于评判编曲）。`theremin` 子命令通过脚本化手势以真实 ToF 帧率驱动流式合成器，是在没有机器人时听到声音变化的唯一方式。
