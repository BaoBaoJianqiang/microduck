# main.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\main.rs`

## 核心设计决策

### 1. 健康定义的重塑

`robot.health` 曾指「循环 tick 过一次」，使每次回退都在测试占位符。现指**循环是否满足截止期限**：以 60% 目标速率运行的循环活着、回答每个请求、但严重损坏。这是 daemon 形态的根本原因。

### 2. IPC 只读原子，永不调用循环

控制循环发布原子（ticks、achieved_hz、battery、fallen…），IPC 侧只读原子。**卡死的循环自我报告不健康，而非挂起调用者**——这正是 updater 需要答案的场景。

### 3. 控制循环独占线程与运行时

总线 read 是阻塞串口 IO，在共享运行时上会占据 worker。故循环有自己的 OS 线程 + current-thread runtime，使 IPC 工作永远不能被调度到 tick 之前。

### 4. tick 调度用 `MissedTickBehavior::Skip`

`Burst` 会堆积电机命令；`Delay` 把每次唤醒延迟累加到周期上（实测 43.1Hz @50Hz 目标，`missed=0`，看似硬件问题）。`Skip` 保持原调度、丢弃错过的 tick。

### 5. 启动不移动机器人

重启时 `adopt_startup_pose` 读取当前姿态并保持，servo 在进程死亡期间保持上一目标，站立的机器人继续站立。扭矩只在显式 `robot.init` 或 enable 时开启，**绝不因进程启动而开启**。

### 6. 策略加载失败可存活

循环继续以原速率运行、保持姿态，`health` 说明原因，updater 回退。拒绝启动会变成 `Restart=always` 下的 crashloop，到达健康门时为 `Unreachable`，归咎错误。

### 7. 掉线读用「滑行」而非停止

一次 Dynamixel 事务丢失是常态（~8 次/分钟）。`COAST_TICKS=3`（60ms）允许策略在最后好样本上滑行，覆盖丢失+重试+慢 tick。超过则静止。曾经失败读会停止策略→命令 hold 姿态→重置控制器→可见抽搐。

### 8. limp-fall：在落地前变软

站立策略擅长站起、不擅长跌倒。`FallPredictor` 在跌倒开始时（而非 `fallen` 判定后）触发：降增益到 `gain_limp`、命令关节到当前位置（跟随测量而非固定目标，避免电机顶地板）→ 等陀螺静止 → 线性 ramp 回站立姿态 → 交回站立策略。

## 关键常量

| 名称 | 值 | 含义 |
|---|---|---|
| `MODEL_API` | 1 | 模型看到的传感器/执行器契约版本 |
| `SOCKET_MODE` | `0o660` | socket 组决定谁可问 |
| `LOOP_SUMMARY_INTERVAL` | 300 s | 循环摘要日志间隔（每 tick 打会每天 430 万行挤掉关键日志） |
| `CHORALE_MOUTH_TAU_S` | 0.09 | 嘴跟随元音的时间常数 |
| `STATE_BUFFER` | 256 | 状态流缓冲（5 秒@50Hz，有损无背压） |
| `RATE_WINDOW` | 1 s | 速率测量窗口 |
| `HOME_RAMP` | 2 s | 到 home 姿态的 ramp |
| `BATTERY_EMA_ALPHA` | 0.1 | 电池电压平滑 |
| `SHUTDOWN_SIT` | 4 s | 关机坐下时长 |
| `COAST_TICKS` | 3 | 掉线滑行 tick 数 |
| `RESET_AFTER_PAUSE` | 200 ms | 驾驶中断多久后重置控制器 |
| `SEATED_BOOT_RAD` | 0.30 | 坐姿启动检测阈值 |

## 核心类型

- **`RobotState`**：循环发布的全部状态原子——ticks/missed/last_tick_us/achieved_hz/consecutive_errors/startup_bus_failures/battery_v/motor 温度/cpu_temp/imu 诊断/fallen/moving/homed，以及广播通道（state_tx/chorale_tx）、policy_error/policies（ArcSwap）、has_voice/theremin_ready/chorale_accepted/mode。
- **`LimpFall`**：`Idle`/`Limp{since, landing}`/`Posing{from, since}`。
- **`Landing`**：陀螺静止检测（去抖，一次安静样本不算落地）。
- **`Bringup`**：`Limp`/`Homing{from, since}`/`Ready`。
- **`Coast`**：掉线滑行器，保留最后好样本。
- **`PolicyNames`**：当前模式的策略文件名集合。

## 核心函数

### `health() -> HealthResult`

更新系统的输入。裁决顺序：`force_unhealthy` → 无 tick 且有 `startup_bus_failures`（degraded，未上电不回退）→ 无 tick（unhealthy）→ policy_error → consecutive_errors 超限 → stall（`last_tick_us` 过旧）→ achieved_hz 低于 `min_achieved_hz`。`healthy/degraded` 只反映发布可归咎的；其余描述随答案附上（机器人行为异常时只被问一次）。

### `safe_to_restart()`

行走中且未跌倒 → 不安全。站立或已倒下可中断。

### `control_loop()`

50Hz 主循环，每 tick：

1. `safety.read()`，`Coast::sample` 处理掉线。
2. `safety.observe`（仅新鲜样本），odometry 更新（仅 IMU 就绪后）。
3. `intents.snapshot()`，`safety.gate`（deadman）。
4. 处理 power 请求（init/relax，先于 enable 使 relax 胜）。
5. 技能请求（需 driving）。
6. 声音（wheee ride → 一次性音效 → 抚摸 coo）。
7. 模式切换（先回 home 再加载策略）。
8. 关机坐下（`robot.shutdown` 或空电池）。
9. **limp-fall 序列**（若开启）。
10. 命令平滑（twist/head/body EMA；limp-fall 时 twist 强行清零）。
11. enable 驱动 bring-up（Limp→Homing→Ready）。
12. 计算 `driving`（enabled && Ready && controller && 非 limp-fall && 有样本 && IMU 暖 && 未关机）。
13. 电压自适应 scale。
14. 选 targets/gain：limp-fall > driving.step > homing > hold。
15. 特雷门琴（嘴在 driving 时由 instrument 拥有）。
16. 合唱（嘴在合唱进行时由 vowel 拥有，slew 平滑）。
17. 嘴意图（仅 driving 且无 instrument/chorale 时）。
18. `safety.apply`。
19. 发布状态帧（仅当有订阅者）。
20. 更新 ticks/rate window/慢传感器（电池 EMA、电机温度、CPU 温度、IMU 诊断）。

### `serve()` / `handle()`

Unix socket JSON-RPC 服务器。每个连接一条线程，`handle` 管理请求行 + 状态流 + chorale beacon 流的多路 select。

## 单元测试

main.rs 主要靠 `tests/updater_gate.rs` 的真实进程测试覆盖。

## 关键摘要

`main.rs` 是 50Hz 控制守护进程。核心设计：健康=满足截止期限（非「tick 过」）；IPC 只读循环发布的原子（卡死自我报告）；循环独占线程+runtime；`Skip` 调度防漂移；启动不移动机器人（保持当前姿态、扭矩不自动开启）；策略加载失败可存活（回退而非 crashloop）；掉线读滑行 3 tick 不抽搐；limp-fall 在落地前降增益变软再 ramp 回站立。嘴的所有权按优先级：limp-fall > 特雷门琴 > 合唱 > 意图。
