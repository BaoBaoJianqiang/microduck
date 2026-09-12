# monitor.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\monitor.rs`

## 定位

`robotctl monitor` — 控制循环实况视图。同一流的两种渲染：
- **终端**：原地重绘一帧（关节跟踪是存在理由——15 个实测角对 15 个命令角，10Hz 文本不可读，条形图一目了然）
- **其他**（管道/文件/`--json`）：每 tick 一行（`monitor > log` 和 `| grep` 必须工作）

## 多线程架构

状态流在自有线程读取（socket read 阻塞，终端事件不在 socket 上，UI 只在机器人发帧时才注意按键=机器人停时无响应）。四个独立连接/线程：
1. `robot.state` 订阅流
2. `padd` 手柄 raw tap（`PAD_RETRY=2s` 重连——padd 每次更新重启）
3. `tofd` 深度流（同上，多数鸭子无 ToF）
4. `robot.health` 轮询（`HEALTH_POLL=2s`，电池/温度/总线不在状态流上）

pad/tof/health 丢失**不致命**：它们是第二/第三/第四 daemon，缺席是正常状态。

## 关键常量

- `BAR_FULL_SCALE=0.20 rad` — 偏差条满刻度（显示用非限值，用弧度不随按键换单位）
- `TRACE_SAMPLES=600` — 环路速率历史
- `HEALTH_STALE=6s` — 健康答案多旧算陈旧
- `CADENCE_WINDOW=100` — 手柄节奏窗口

## 单位切换

`Units::{Degrees, Radians}` — 显示辅助（度数直观），但 wire 始终弧度。`angle()` 保证度数精度不低于弧度（2 位度 vs 3 位弧度），切换不静默隐藏差异。

## PadView — 原始事件流

**全部从 raw reports 派生，不用 padd 的判断**（padd 50Hz 重发最后摇杆值，使已停的链路从下游看像有手在杆上）。

- `Silence` 四态：Arriving/Notable/PastTheDeadman/Idle——区分"静止"与"死链路"是测手柄链路的核心难点（曾把静止算成 75s 死链路）
- `cadence()` — 仅运动时的报告率（不用流逝时间，否则暂停时读成极慢）
- 统计：gaps、worst_ms、over_notable、over_deadman、quiet、after_drops、socket_dropped、clock_steps

## ToF 网格

8×8 矩阵，近→远暖→冷色阶（手在头前=红色）。`frame_zone()` 镜像 `tof::Zone` 规则（不链接 tof crate 驱动）。floor return 绿色（不是障碍物），TooClose 暗淡，Unusable 品红 `x`。`TOF_STALE=400ms`。

## 3D 视图 + 路径图

集成 `duck::DuckView`（3D 机器人）和 `path_map::PathMap`（俯视里程计轨迹，盲文 2×4 点）。

## 关键摘要

monitor.rs 是控制循环实况终端 UI：终端原地重绘/管道逐行两种模式；四独立线程（state/pad/tof/health），pad/tof/health 丢失不致命；关节偏差条用弧度满刻度；pad 链路区分静止与死链路四态；ToF 8×8 暖冷色阶+floor 绿色；集成 3D 视图与路径图。
