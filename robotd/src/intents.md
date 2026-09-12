# intents.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\intents.rs`

## 核心设计决策

### 1. 意图槽用 `ArcSwap` 而非互斥锁

控制循环每 tick 读一次客户端意图，IPC 任务写。用 `ArcSwap` 让读者做一次原子 load、写者做一次原子 store，**任何一方都不能阻塞另一方**——循环永远不会卡在客户端上。

### 2. twist 与 head 分槽

单一合并槽更新其中一个字段需要 read-modify-write，两个客户端（手柄驱体、另一客户端驱头）会静默丢失彼此的更新。分槽使每个槽在实践中都是单写者，last-writer-wins 才有意义。

### 3. 每个槽都带时间戳

循环真正问的不是「值是什么」而是「它有多旧」——这是 deadman 看门狗读的东西。twist 时间戳在 `set_twist` 时刷新，`set_head` 不碰它。

### 4. 离散请求用「取一次」而非标志

- `power`（init/relax）、`theremin`、`chorale`、`mode_switch`、`shutdown`、`skills`、`sounds` 均为**边沿**（edge），循环每 tick `swap` 取走并清零。
- 理由：`set_torque` 是每关节一次总线事务，若为电平（level）会在每 tick 产生 16 次写。
- skills/sounds 用**位掩码**而非单个 last-writer 槽：同一 20ms tick 内两个不同按钮都应被看见。

### 5. twist 初始化为「最旧」

`Stamped { value: [0;3], at_us: 0 }`，使 deadman 在任何客户端连接前就认为 twist 已过期，机器人保持静止。若初始化为「新鲜」，机器人会短暂相信自己有一个活的驱动者。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `MODE_NONE` | `u8::MAX` | 无待切换模式（用 255 而非 0，因 0 是真实的 Walk 模式码） |
| `WHEEE_HOLD_FRESH` | 300 ms | wheee 长按保持新鲜的时限 |

## 类型

- **`Stamped<T>`**：值 + 自 epoch 起的微秒时间戳。
- **`PoseIntent`**：站立身体姿态 `[z, roll, pitch]` + `active`（false 时瞬缩回名义位，模拟原型 B 键退出）。
- **`SkillRequests`**：5 个一次性技能的位掩码（ground_pick / kick_left / kick_right / sit_toggle / roulade）。
- **`WheeeHold`**：三态枚举——`Held`（新鲜按住）、`Released`（客户端主动松开）、`Decayed`（客户端停止重通知或消失）。**两种「未按住」是不同声音**：Released 截断 ride，Decayed 让 ride 落地播放尾段。
- **`Snapshot`**：循环每 tick 读取的快照，含 `command`、`twist_age`、`enabled`、`pose`、`mouth`。
- **`Intents`**：全部意图槽的集合。

## 函数（节选）

- `set_twist` / `set_head`：写带时间戳的 twist/head。
- `stop`：将 twist 置零但**不**禁用策略（机器人应站立，而非瘫软）。
- `request_skill` / `take_skills`：位掩码取一次。
- `request_sound` / `take_sounds`：一次性音效用位掩码；`wheee` 例外，走 stamped level 槽。
- `wheee_hold`：将 wheee 槽读为三态。
- `request_mode_switch` / `take_mode_switch`：模式码（非 `Mode` 枚举，使本模块不必知道有哪些模式）。
- `request_init` / `request_relax` / `take_power_request`：`relax` 同时清 `enabled`。
- `request_chorale` / `take_chorale_request`：先写 `chorale_piece` 再写请求标志（标志 swap 是同步点）。
- `heard_chorale` / `take_chorale_heard`：**唯一用 Mutex<Vec> 的意图**，因为 beacon 是带到达时间的事件，last-writer-wins 会丢拍。
- `snapshot`：组装命令与 twist 年龄。

## 单元测试

- `a_stale_hold_reads_as_decayed_not_released`：固定三态语义——新鲜 `true`→`Held`，超时→`Decayed`，客户端 `false`→`Released`。
- `the_twist_starts_stale`：初始 twist 年龄已非零，`enabled=false`。
- `the_slots_are_independent`：写 head 不影响 twist 值与年龄。
- `writing_the_twist_refreshes_its_age`：写 twist 后年龄变小。
- `stop_zeroes_the_twist_and_leaves_the_policy_enabled`：`stop` 不归零 enabled。
- `the_body_command_stays_nominal`：body 块保持训练名义零值。

## 关键摘要

`Intents` 是 IPC 任务与 50Hz 控制循环之间的无锁交接层。核心设计：twist/head 分槽避免多客户端互踩；连续量（twist/head/mouth）走带时间戳的 `ArcSwap` 电平；离散请求（power/skills/sounds/mode/shutdown）走「取一次」边沿位掩码；chorale beacon 是唯一用 Mutex 队列的，因为事件不能丢。twist 初始化为最旧，使无人驱动时机器人自然静止。
