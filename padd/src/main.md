# main.rs 文件解析

**文件位置**：`d:\microduck\padd\src\main.rs`

## 核心设计决策

`padd` 把手柄变成意图客户端——**对机器人没有特权访问**。它读手柄、把摇杆和按键转成意图，通过 `robotd` 的 socket 像任何其他客户端一样发送。

**独立进程而非 robotd 内线程的意义**：意图 API 是 app、SDK、远程客户端都会走的路，这里每天被开发机器人的人锻炼，不会像只有手机 app 用的 API 那样静默腐烂。代价是一次 socket 跳跃（几十微秒对 20ms tick）。

**平滑在 robotd 侧**（`[control] cmd_alpha` / `head_alpha`），不在此处——本进程发原始目标，所有客户端获得一致手感。

## 按键映射（原型的，肌肉记忆沿用）

```
Start        切换策略
Y (North)    头部模式——摇杆摆头
B (East)     身体姿态模式——摇杆让站立机器人倾斜/下蹲
A (South)    地面拾取
LB / RB      左/右踢
DPad-Down    坐↔站
RT / LT      嘴（任一扳机，取最大）· RT 叫 · LT 骑 wheee
Select, 2s   坐下然后关机
DPad-Up, 3s  行走↔滚轮模式切换
```

头部与身体姿态模式激活时都将速度归零——机器人因你开始摆头而继续走是糟糕的意外。

## 滚轮模式

启动时询问 `robot.mode`。滚轮机器人用原型滚轮预设：非对称前进/刹车（0.6/0.5）、无横移、±0.3 rad/s 转向；A 触发蹲伏（在 ground-pick 槽）。

## 常量

- `IDLE_POLL = 500ms`：无手柄时的轮询间隔。刻意长于控制 tick——本进程从开机常驻，大部分时间无手柄，以控制率自旋发现"无手柄"是每 20ms 一次无谓唤醒。
- `SHUTDOWN_HOLD = 2s`：Select 按住这么久触发关机。
- `MODE_HOLD = 3s`：DPad-Up 按住这么久切换模式。比关机长，因模式切换把机器人带回家并重载策略，必须是无人会误操作的长按。
- `BODY_MAX_Z_UP = 0.010`、`BODY_MAX_Z_DOWN = 0.025`、`BODY_MAX_ANGLE = 0.2618`（~15°）：身体姿态摇杆范围，z 非对称（站立高度向上余量小，向下蹲伏余量大）。
- `ROLLER_PUSH = 0.6`、`ROLLER_BRAKE = 0.5`、`ROLLER_YAW = 0.3`：原型滚轮模式摇杆塑形。

## 类型

### `Args`（CLI）
- `socket`：robotd socket，默认 `/run/robotd.sock`。
- `hz = 50`：发意图频率。精确匹配控制率无意义（循环每 tick 读一次最新值），但≥控制率可保额外延迟<1 tick。
- `deadzone = 0.1`：死区，模拟摇杆很少精确归零，否则机器人会蠕走。
- `max_linear = 0.3`、`max_linear_backward = 0.3`：前/后满偏速度（原型分别限）。
- `max_angular = 1.5`：满偏转向率 rad/s。
- `max_head = 2.5`：满偏头部行程 rad（头命令喂策略观测而非直接舵机，网络自己决定头实际走多远）。
- `tap_socket`：原始输入 tap socket，默认 `proto::socket::PAD`。

### `Mode`
`Drive` / `Head` / `BodyPose`。头部与身体姿态是模态的——两个摇杆无法表达 9 个自由度。

## 函数

### `main`
1. 解析参数、初始化 tracing、`log_startup_identity!`。
2. 初始化 `Gilrs`，失败退出（systemd 重试）。
3. 启动 raw input tap（失败只 warn，继续驾驶——调试设施不能阻止机器人）。
4. 连接 robotd socket，失败退出。
5. 询问 `robot.mode` 一次，确定是否滚轮。
6. 主循环（按 `hz` 节拍）：
   - 排空 gilrs 事件队列，捕获**按键边沿**（Start 按住只切换一次而非 50 次/秒）。
   - 无手柄：什么都不发（robotd 的 deadman 自己停机器人）；发零命令会把断开的手柄伪装成主动停止。每转换记录一次 warn。
   - 有手柄：模式切换（Y/B）、策略切换（Start——用 `toggle:true` 让机器人自己翻转状态，不本地持信念）、一次性技能（地面拾取/踢/坐/翻滚——有应答，因"拒绝并说明原因"是真实结果）、Select 长按关机、DPad-Up 长按模式切换（目标显式命名而非 toggle，因跨切换请求可能到时模式已变）。
   - 嘴：RT 与 LT 取最大。RT 上升沿叫（chirp），LT 按住骑 wheee（hold=true 每 tick 通知，释放 hold=false）。
   - 按当前模式构建意图：Drive（walk/roller 不同塑形）、Head（先停速度再发头命令，符号与原型一致）、BodyPose（先停速度再发身体姿态，退出时 snap 回 nominal）。
   - 发 `notify`（连续意图无 id 无应答）。

### `notify(stream, call)`
发连续意图：无 id、无回复、不等。

### `request(stream, next_id, call)`
发离散意图并读应答。有应答因为"拒绝并说明原因"是真实结果（无策略的技能、无音库的声音），忽略会让操作者疑惑为什么没反应。

## 关键摘要

`padd` 是意图 API 的日常"压力测试"——无特权、走 socket、与 app/SDK 同一路径。核心工程取舍：(1) 本地不持策略状态信念，让机器人拥有 toggle；(2) 无手柄时发空而非零，避免伪装断开；(3) 模式切换目标显式命名；(4) 平滑在 robotd 侧统一。
