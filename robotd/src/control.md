# control.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\control.rs`

## 核心设计决策

### 1. 纯计算层，不持有 IO 句柄

本模块位于 `RobotIo::read` 与 `safety.apply` 之间，只做传感器+命令→关节目标的计算。**构造上不可能命令电机**，只能提议目标——安全层持有唯一的写句柄。

### 2. 替换原型 `control_step`，保留两个微妙行为

来自 `microduck_runtime` 的优先级链与数值默认值，其中两个易被「顺手修掉」的点被显式保留：

- **踢球窗口以站立调参运行**：kick 的观测携带全零命令，站立转换恰在此触发，故 kick 用 `standing_action_scale` 与软化后的站立增益。
- **sitstand 的「起身」也用站立增益**（命令全零），而「坐下」不用（姿势标志使 twist 幅度为 1）。

### 3. 故意的一点分歧：每 tick 重算 scale 与 gain

原型在转换时保存/恢复 `action_scale`，sit→stand 周期后可能留下陈旧值直到下次行走。这里 scale/gain 每 tick 从活动状态重算，无遗留。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `HEAD_JOINTS` | `5..9` | 头低通覆盖的关节：neck_pitch, head_pitch, head_yaw, head_roll |
| `GROUND_PICK_END_PHASE` | `0.7` | ground pick 在此处交回（原型截断点，1.0 会在回程重播伸手动作） |
| `RISE_SECS` | `1.0` | sitstand 起身持续秒数 |
| `ROULADE_CHAIN_WINDOW` | `0.15` | 滚翻结束后多久内的请求算「按住连滚」（约 7 tick） |

## 类型

- **`Tuning`**：行走调参（`action_scale=0.9`、`standing_action_scale=1.0`、`standing_gain_ratio=0.8`、`gain=200`、`head_lowpass=Some(0.5)`、`legs_lowpass=Some(0.7)`）。
- **`SkillTuning`**：脚本技能调参（ground_pick 周期 4s、kick 0.5s、roulade 1.0s 等）。
- **`Step`**：一 tick 的决策结果——`targets`、`label`（网络线标签）、`gain`、`busy`（脚本动作进行中，不可重启）。
- **`Sit`**：坐↔站循环状态（`Up`/`Sitting`/`Rising{remaining}`）。
- **`Controller`**：持有 `Policy`、调参、`last_action`（原始未缩放，因策略观测自己的输出）、`previous`（低通用）、各技能窗口状态。

## 核心方法

### `step(sensors, command, body_active, dt, scale_mult) -> Step`

每 tick 一次，顺序：

1. **窗口过期**（先于推理）：kick/roulade/rising 到期。
2. **选网络**（优先级链）：`roulade > kick > ground_pick > sit/rise > stand > walk`。
   - roulade/kick：命令全零（被选中即触发）。
   - ground_pick：twist 槽携带相位编码 `(cos, sin, 0)`。
   - sit：`twist=[1,0,0]`（姿势标志）；rise：`twist=[0,0,0]`。
   - 站立/行走：按 `command.twist_magnitude()` 与阈值选择；`body_active` 强制站立。
3. **构建观测** → `policy.infer`。
4. **重算 scale 与 gain**（每 tick）：站立调参在 `Stand` 网络，或 kick/sitstand 且命令幅度落入站立阈值时生效。
5. **targets = home_pose + scale × offsets**，应用 head/legs 低通。
6. **推进窗口**（推理之后，原型在电机写之后推进相位）。

### 技能控制

- `start_ground_pick`：无门控（原型行为，可抢占 kick 尾段）。
- `start_kick`：任何脚本动作运行中拒绝。
- `request_roulade`：运行中刷新 chain 窗口；`Ok(true)` 启动、`Ok(false)` 刷新。
- `sit_toggle`：起身中拒绝。
- `begin_shutdown_sit` / `begin_boot_rise`。

## 单元测试

- `the_defaults_match_the_prototype`：钉死所有默认值与原型一致（过滤器 alpha 必须匹配训练值 0.5/0.7）。
- `standing_softens_the_gain`：站立增益 = 200×0.8 = 160。
- `the_ground_pick_cutoff_is_the_prototypes`：`GROUND_PICK_END_PHASE==0.7`、`RISE_SECS==1.0`。

## 关键摘要

`Controller` 是策略调度器：按原型优先级链在 7 个网络间切换，每 tick 从活动状态重算 scale/gain（消除原型保存/恢复的陈旧值 bug），保留 kick/sitstand-rise 以站立调参运行的微妙行为。`last_action` 保留原始未缩放输出（策略观测自身输出），低通覆盖 head（0.5）与 legs（0.7）。
