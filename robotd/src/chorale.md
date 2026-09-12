# chorale.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\chorale.rs`

## 核心设计决策

### 1. 无人负责，直到有人负责

被请求合唱的鸭子先**监听**：发出空闲 beacon 表示愿意，同时观察。当两只愿意的鸭子互相看见，**id 较低者指挥**——确定性选举，无消息可丢。听到已携带曲目的 beacon 不争论，直接加入。

### 2. 指挥拥有座位表

座位取决于加入顺序，鸭子自己从听到的子集排座会彼此不一致（都唱中音）。故指挥维护 roster 并广播，所有人对其重放 `seat_all`。不在 roster 中的鸭子继续监听，下一拍被加入——加入免费，无需协商。

### 3. 时间基准是指挥的拍计数器，非起始时间

无可约定时钟。`Conductor`（持拍者）与 `Follower`（其余）回答同一问题：「现在是第几拍」。音频侧据此渲染，stall 后恢复在正确位置而非落后一小节。

### 4. 身份来自 beacon，绝不来自无线电地址

BLE privacy 开启时地址每数秒旋转。曾用地址做 conductor 标识，导致 follower 采纳一次后拒绝后续所有 beat。现用 beacon 的 `(register, id)`。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `PEER_STALE` | 3 s | 听到的 beacon 多久后失效（走出范围） |
| `SETTLE` | 1.5 s | 听多久后若无人则独自开始（实际不会独唱，仅是两鸭互相确认的时间） |
| `PIECE_WISTFUL` / `PIECE_DUCK_STRUT` / `PIECE_OUTER_WILDS` | 1/2/3 | 曲目 id（OUTER_WILDS 测试用，发布前移除） |
| `IDLE_HEARTBEAT` | 1.5 s | 空闲 beacon 多久变一次（不变则不重注册广告，两只鸭找彼此需数十秒） |

## 类型

- **`Peer`**：最近听到的鸭子（beacon + 本地到达时间）。
- **`State`**：`Off` / `Listening{since}` / `Conducting{conductor, roster}` / `Following{follower, conductor, roster, last_beat}`。
- **`Tick`**：本 tick 对循环的指示——`advertise`（beacon 变化时才发）、`singing`（声部+拍位）、`joining`、`voices`（实际在唱的数量，保留离场者座位）。
- **`Chorale`**：本鸭身份（register 量化 + 16 位 id）、当前曲目、状态、peers、`forced_piece`、`env_piece`。

## 方法

- `new(pitch_center_hz, seed, env_piece)`：id 取 seed 的 16 位混合（曾用 8 位，四鸭房间碰撞）。env_piece 过滤为已知曲目。
- `set_active` / `heard` / `tick`：状态机驱动。
- `conduct` / `follow`：指挥推进拍计数、维护 roster；跟随者用相位锁估计位置。
- `my_part`：从 roster 查本鸭声部。
- `score()`：当前曲目供音频侧渲染。
- `head_expression(beats, reach) -> [4]`：唱歌时的头部表情——由**共享拍位**计算，使整群同步摇摆，无需协调。幅度刻意小（头载 ToF 且策略平衡此处质量）。

## 单元测试（节选）

- `one_duck_alone_does_not_sing`：独处不唱（合唱非独唱）。
- `the_lower_id_conducts_and_the_other_follows`：低 id 指挥，无选举消息。
- `a_conductor_restarting_with_a_new_piece_takes_its_followers_with_it`：指挥重启换曲，跟随者必须同步换（回归测试）。
- `a_beat_counter_going_backwards_is_a_new_performance`：拍计数倒退视为新演出，重置相位锁。
- `a_forced_piece_wins_the_coin_toss`：`DUCK_CHORALE_PIECE` 固定指挥选曲；未知 id 回退到随机而非卡死。
- `a_performance_ends_and_the_ducks_go_back_to_listening`：曲目结束回到监听。
- `a_duck_sharing_a_register_is_still_a_different_duck`：id 16 位防碰撞。
- `the_conductor_is_followed_across_its_rotating_addresses`：跨地址旋转仍跟随（核心回归测试）。
- `an_idle_beacon_has_a_heartbeat`：空闲 beacon 必须定期变化。
- `an_unknown_piece_is_declined_not_guessed`：混合版本群中老鸭子保持安静。
- `the_head_sways_in_phase_and_stays_small`：头部表情同拍同步且幅度受限。

## 关键摘要

`Chorale` 是多鸭合唱的行为层（`btd` 管无线电、本模块管思考）。核心机制：低 id 确定性指挥、指挥独占 roster、拍计数器即时间基准、身份来自 beacon 而非旋转的无线电地址。曲目结束回到监听，未知曲目拒绝不猜。头部表情由共享拍位驱动，整群免费同步摇摆。
