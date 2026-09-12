# theremin.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\theremin.rs`

## 核心设计决策

### 1. 三个速率在此交汇

- 深度帧：`tofd` 通过 socket 以 15 Hz 推送（本 daemon 不拥有该 socket）。
- 控制循环：50 Hz，且是唯一允许动嘴的地方。
- 音频：48 kHz 写入线程。

本模块是前两者的交汇点：一个 reader 线程停在深度 socket 上，把最新帧放入槽；`Theremin::tick`（由控制循环调用，永不阻塞）把槽里的内容变成音符、嘴型与状态行。

### 2. 一个手势，三个输出

手的 closeness（0=可弹奏带远端，1=近端，来自 `kinematics::hand`）同时驱动**音高、音量、嘴张开度**。不是同一事物的三个调参，而是字面上一个数——嘴张开曲线与音高曲线不一致会读成「在声音上播放的嘴动画」而非动物发声。

### 3. 显式模式，不耍聪明

第一版自动 arm（捕捉前方作为背景以区分手与墙）。在工作台上可行，在鸭子身上同一手势时而 arm 时而拒绝，因为**哪些 zone 携带可用状态逐帧变化**，背景只和平均它的帧一样稳定。现改为显式开启，开启时最近回波即手。

### 4. 速率失配是淡出，非门控

帧停止到达（`tofd` 重启、传感器掉线）不应让音符永远响，也不应硬切。超过 `FRAME_STALE` 把 level 置零，其余保持——乐器静音，帧回来时恢复。短掉帧由 `hand::Tracker` 桥接，根本到不了这里。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `FRAME_STALE` | 500 ms | 深度帧仍可播放的时长 |
| `RECONNECT` | 2 s | reader 重连深度 socket 的间隔 |
| `SUBSCRIBE_ID` | 1 | 订阅请求 id |

## 类型

- **`Frame`**：原始距离与状态字节（不解释——哪些状态有效是 `hand::Config` 的决定，在此解释会埋没它）+ 到达时间。
- **`Note`**：一 tick 产物——`mouth`（嘴张开 0..1，乐器举起时总存在）、`state`（`ThereminState`，`note_hz` 留给调用者填，因映射需要 voice 的 register）、`closeness`。
- **`Theremin`**：`latest`（`ArcSwapOption<Frame>`）、`tracker`、`active`。

## 方法

- `spawn(socket, config)`：启动 reader 线程（无论是否有人要 instrument 都启动，延迟连接会让首次拾琴多等一帧）。
- `new(config)`：**不启动线程**——供测试用。reader 指向无人应答的 socket 会在 connect 失败时清空槽，在并行测试中会抹掉刚推入的帧。
- `has_frames`：最近帧是否新鲜（`robot.theremin` 的拒绝依据）。
- `set_active`：拾琴/放下；放下时 `tracker.reset()`，避免上次的残留音在重新拿起时响起。
- `tick`：不激活返回 `None`；帧过期返回「闭嘴 + no depth frames」状态；否则经 `tracker.track` 得到 hand，输出 Note。

## 辅助函数

- `describe(status, config)`：把状态字节直方图化为一行，标注哪些 code 被本 build 相信（`*` 标记）。这是能在一分钟内发现第一版 bug 的诊断——直接报告传感器对每个 zone 说了什么。
- `read_frames`：永久停在 `tofd` 深度流上，断线清空槽。
- `stream_frames`：一次连接（subscribe→帧），跳过无法解析的行。

## 单元测试

- `a_theremin_that_is_down_says_nothing`：未拾琴不输出任何东西（连静音都不输出）。
- `it_plays_on_the_first_frame_after_being_picked_up`：拾琴后第一帧即响，无 arming。
- `a_hand_past_thirty_centimetres_plays`：40cm 手带 consistency-failure 状态（4/13）必须响——曾把此功能打回重做的回归。
- `the_mouth_opens_with_the_note`：越近嘴张越大，且与音高是同一个数。
- `the_state_reports_what_the_sensor_actually_said`：状态行报告各 code 计数及相信标记。
- `a_stale_frame_is_silence_and_the_instrument_stays_up`：传感器停摆→闭嘴，乐器仍在手；帧回来恢复。
- `a_flickering_sensor_does_not_chop_the_note`：可用/不可用帧交替不切音（hold 桥接）。
- `putting_it_down_forgets_the_held_note`：放下后再拿起，旧音不残留。

## 关键摘要

`Theremin` 把 15Hz ToF 深度帧桥接到 50Hz 控制循环：reader 线程停在 socket 上存最新帧，`tick` 永不阻塞。closeness 同时驱动音高、音量、嘴型（一个数，三输出）。显式模式（不自动 arm）、速率失配淡出（非门控）、状态行直接报告传感器原始状态分布，是重写时确定的三个核心设计。
