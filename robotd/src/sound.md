# sound.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\sound.rs`

## 核心设计决策

### 1. PCM 独占单客户端，两个性质由此推出

- **一个播放子进程，新声音杀死旧的**：让连续点击 chirp 干净利落。
- **wheee ride 流进单个 `aplay`**：start→loop（按住时重复）→end，由节奏化线程写入使 pipe 队列不超过 ~250ms，否则释放会滞后那么久。

### 2. ride 的两个出口不是同一回事

- `Released`（客户端说松开）：**截断**——kill 子进程，writer 因 broken pipe 退出。
- `Decayed`（按住过期）：**落地**——让 writer 退出循环，写入 end 段到仍打开的 pipe。只有后者播放 `wheee_end_*`。

### 3. 特雷门琴合成，非播放

其他声音是 bank 预渲染的 wav（形状已知）。特雷门琴音高是手的距离——每秒 15 个新值，无法预知，故无文件可选。writer 线程从 `sounds::Stream` 实时拉块，比 ride 跟得更紧（30ms lead vs 250ms），因为乐器的全部质量在这个数字上。

### 4. 播放即 spawn，不阻塞 50Hz tick

唯一阻塞的是关机前的 goodbye peck（那时已无 tick 可错过），且有上限（1500ms）。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `BLOCKING_PLAY_MAX` | 1500 ms | goodbye peck 的阻塞上限（防卡死 PCM 拖慢关机） |
| `SYNTH_BLOCK` | `SR/100`（10ms） | 特雷门琴/合唱一次渲染的音频块 |
| `SYNTH_LEAD_S` | 0.03 | 合成 writer 领先播放的秒数 |
| `CHORALE_LEVEL` | 0.55 | 合唱单只鸭子的音量（四只声学求和，必须留余量） |
| `SPEAKER_ROLLOFF_HZ` | 300 Hz | 鸭扬声器下限（实测），用谐波承载低音 |

## 类型

- **`Ride`**（枚举）：PCM 所有权状态——`Off`/`Riding`/`Landing`/`Theremin`/`Singing`。**不是 bool 也不是两个**：每个状态都需区分。
- **`Live`**：writer 线程的实时参数（`hz`/`level`/`open` 原子 + `playing` 标志）。
- **`Sound`**：播放器，持有 bank 路径、device、子进程、`wheee_held`、`ride`、`live`、`singing`、`personality`。

## 方法

- `play(tag, blocking)`：从 bank 的 `tag/` 目录随机选 wav，spawn `aplay`。ride 持有 PCM 时丢弃（非阻塞）；blocking goodbye peck 例外。
- `theremin_start` / `theremin_set` / `theremin_stop` / `theremin_settle`：拾琴、设参数、放下、回收。放下是淡出非硬切。
- `sing_start` / `sing_at` / `sing_stop`：合唱。按 `(piece, part)` 幂等，换 part 才重启。
- `wheee(hold)`：按 `WheeeHold` 三态驱动 ride。
- `start_wheee`：start→loop→end 流进一个 `aplay`，保持 ~250ms 领先。
- `degraded_wheee`：bank 无 wheee 三段时降级为一次性播放并**锁存 ride**（否则按住会每秒 50 次 fork/exec aplay）。
- `voice`：从 bank 的 `.seed` 标记读 Personality（而非硬件 id，使 bank 与 theremin 声音一致）。

## 辅助

- `block_sigpipe`：writer 线程屏蔽 SIGPIPE（`robotd` 启动时恢复了 SIGPIPE 默认行为以使 stdout 管道正常，否则写入死 aplay 会杀死整个 daemon）。
- `wait_bounded`：限时候子进程（goodbye peck）。
- `read_wav_pcm`：最小 RIFF 解析（采样率 + S16LE mono 载荷）。

## 单元测试

- `read_wav_pcm_reads_what_sounds_writes`：与 `sounds` crate 的 wav 写入对拍。
- `a_missing_bank_is_silent_not_fatal`：无 bank 降级为静默。
- `a_bank_without_wheee_triads_latches_instead_of_respawning`：无三段 wheee 时锁存，不反复 spawn。
- `a_decayed_hold_lands_and_a_release_cuts`：两个出口的声音不同，且都回到 `Off`。

## 关键摘要

`Sound` 管理独占单客户端 PCM：一次性音效 spawn `aplay` 播放 bank wav；wheee ride 以 start/loop/end 三段流进一个 `aplay`，有截断（Released）与落地（Decayed）两个不同出口；特雷门琴与合唱由 writer 线程从 `sounds::Stream` 实时合成（30ms 低延迟领先）。所有播放不阻塞 tick，唯独关机 peck 阻塞且有上限。
