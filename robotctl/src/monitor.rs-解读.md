# monitor.rs（实时控制循环监控器）解读与架构梳理

> 分析对象：`monitor.rs`（约 4220 行），`robotctl monitor`——控制循环的实时视图。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `robotctl monitor`——控制循环的实时监控。对同一条数据流做两种渲染，取决于 stdout 去向：

- **终端**：原地重绘帧——重要数字可以被*注视*而非从一千行滚动文本中重建。关节跟踪表是存在的理由：15 个测量角旁边 15 个指令角，10Hz 时纯文本不可读，15 个条形图一目了然。
- **其他**（管道/文件/--json）：每个 tick 一行，保持脚本可用。`robotctl monitor > log` 和 `| grep` 必须继续工作。

**关键设计**：
- **独立线程读流**：socket 读阻塞，终端事件不在 socket 上到达。UI 只能等机器人发帧才能注意按键 = 机器人一停就冻结。所以 socket 读在独立线程，通过 mpsc channel 传给 UI。
- **关节条形图**：BAR_FULL_SCALE=0.20 rad，BAR_HALF=12 格。0 有自己的列——"无误差"不能看起来像"偏左一点"。
- **固定布局高度**：HEADER_HEIGHT=10, PAD_HEIGHT=8。header 在限制出现时不增长——否则读者盯着某行时所有关节行下移。
- **ToF 面板**：8×8 矩阵。
- **idle 重绘**：250ms——卡住的流必须看起来卡住了（header 中的时间在涨）。

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | 两种渲染：终端（ratatui）vs 管道（逐行文本） | L1-11 | confirmed |
| F2 | socket 读在独立线程，mpsc channel 传 UI | L13-15, L20 | confirmed |
| F3 | BAR_FULL_SCALE=0.20 rad（显示刻度，非限制） | L44 | confirmed |
| F4 | BAR_HALF=12 格（0 有自己的列） | L48 | confirmed |
| F5 | TRACE_SAMPLES=600（loop-rate 历史） | L52 | confirmed |
| F6 | IDLE_REDRAW=250ms | L56 | confirmed |
| F7 | HEADER_HEIGHT=10（固定） | L62 | confirmed |
| F8 | PAD_HEIGHT=8（开 pad 时） | L73 | confirmed |
| F9 | SUBSCRIBE_ID=1（subscribe 请求 ID） | L66 | confirmed |
| F10 | 集成 duck（3D 视图）和 path_map（轨迹图） | L34 | confirmed |
| F11 | 非终端 → 逐行文本/JSON | L9-11 | confirmed |

---

## 三、架构

```
robotd / robot.state 流 (Unix socket)
  │
  ├─ 读线程: blocking read → mpsc::Sender
  │
  └─ UI 线程: recv timeout → ratatui draw loop
       ├─ joint bars (15 joints: measured vs commanded)
       ├─ loop-rate trace (sparkline)
       ├─ header: age, IMU, odometry, power
       ├─ 3D duck view (duck::DuckView)
       ├─ path map (path_map::PathMap)
       ├─ ToF 8x8 matrix panel
       └─ pad panel (axes + buttons)
```

---

## 四、关键设计决策

### 4.1 为什么独立线程读流

socket.read() 阻塞。如果同一线程既读 socket 又处理按键：
- 机器人不发帧时，按键事件排队——UI 冻结。
- 机器人停了，监控器也死了——最需要看的时候。

所以：读线程只管 socket，UI 线程只管按键和绘制，通过 channel 连接。

### 4.2 为什么 BAR_FULL_SCALE=0.20 rad 是显示刻度不是限制

注释说得很清楚：nothing refuses a joint for exceeding it。这是显示尺度——让条形图静止时在中间，走路时明显摆动。恒定饱和或不动的条形图一样没信息。

### 4.3 为什么布局高度固定

header 里出现一个限制就长高一格 → 所有关节行下移 → 读者正盯着那行。固定高度是"不挡路"的设计。

### 4.4 为什么非终端走逐行文本

`robotctl monitor > log` 和 `| grep` 必须继续工作。一个往日志文件写转义码的屏幕绘制 CLI 没人能脚本化。

---

## 五、结论

### confirmed
- C1：实时监控 TUI，关节条形图 + loop-rate trace + 3D 视图 + 路径图 + ToF。
- C2：独立线程读流，mpsc channel 传 UI。
- C3：非终端输出逐行文本/JSON。
- C4：固定布局高度防止跳行。
- C5：250ms idle 重绘显示卡住。

### inferred
- I1：关节数据从 robotd 的 state 订阅流获取。
- I2：按键交互包括旋转 3D 相机（[ / ]）、切换面板（pad/ToF）。
- I3：IMU 重力和里程计数据同流传输。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
