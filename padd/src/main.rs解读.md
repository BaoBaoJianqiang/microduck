# `main.rs` 解读 — padd

## 概述

`padd` 是一个 gamepad 意图客户端（intent client）——从手柄读取摇杆和按钮，转为意图，通过 `robotd` 的 socket 发送。

### 为什么是独立进程而非 robotd 中的线程

独立进程的意义：**意图 API 每天被开发人员使用**，不会像只有手机 app 用的 API 那样默默腐烂。代价是一次 socket hop——几十微秒，相对于 20ms 的 tick 可以忽略。

### 映射是原型的

肌肉记忆从 `microduck_runtime` 继承：

| 按钮 | 功能 |
|------|------|
| Start | 切换策略 |
| Y (North) | 头部模式——摇杆摆头 |
| B (East) | 身体姿态模式——摇杆倾斜/蹲下 |
| A (South) | 地面拾取 |
| LB / RB | 左/右踢 |
| DPad-Down | 坐下 ↔ 站立 |
| RT / LT | 嘴（max 获胜）· RT 鸭叫 · LT wheee |
| Select, 2s | 坐下然后关机 |

头部和身体姿态模式在激活时都将速度归零——头部模式下机器人继续走是糟糕的惊喜。

平滑在 `robotd` 中（`[control] cmd_alpha` / `head_alpha`），不在此处：这个进程发送原始目标，每个客户端得到相同手感。

---

## 启动流程

```rust
fn main() -> std::process::ExitCode {
    let args = Args::parse();
    tracing_subscriber::fmt()...init();
    duck_ipc_proto::log_startup_identity!("padd");
```

### 启动顺序的设计

1. **解析参数**
2. **初始化 tracing**
3. **记录启动身份**：在任何可能失败的操作之前，特别是在 gamepad 子系统之前——padd 曾经是唯一其 journal 无法说出哪个构建在运行的守护进程
4. **初始化 gilrs**：失败则退出（系统会重试）
5. **启动 tap**（在 robotd socket 之前）：tap 是唯一能告诉为什么手柄看起来死掉的东西，它自己的失败被记录并跳过
6. **连接 robotd socket**：失败则退出
7. **查询机器人模式**（walk/roller）：决定摇杆映射

---

## 关键常量

```rust
const IDLE_POLL: Duration = Duration::from_millis(500);
```

无手柄时的轮询间隔。**故意比控制 tick 长**：padd 从启动时就在每个机器人上运行，大多数时候没有手柄连接——以控制速率空转是每 20ms 一次唤醒，永远，什么都不做。500ms 在有人开手柄时不可察觉，也不是后台负载。

```rust
const SHUTDOWN_HOLD: Duration = Duration::from_secs(2);
```

Select 按住此时长→坐下然后关机。

```rust
const MODE_HOLD: Duration = Duration::from_secs(3);
```

DPad-Up 按住此时长→切换驱动模式（walk ↔ roller）。比关机保持长，原型的数字。DPad-Up 是任何人在驾驶时可能倚靠的方向；模式切换让机器人回家并重载策略，所以必须是没人会意外执行的保持。

```rust
const BODY_MAX_Z_UP: f64 = 0.010;
const BODY_MAX_Z_DOWN: f64 = 0.025;
const BODY_MAX_ANGLE: f64 = 0.2618;
```

身体姿态摇杆范围（来自训练环境）：z 不对称（站立高度上方空间小，蹲下下方空间大），角度上限约 15°。

```rust
const ROLLER_PUSH: f64 = 0.6;
const ROLLER_BRAKE: f64 = 0.5;
const ROLLER_YAW: f64 = 0.3;
```

roller 模式摇杆塑形：推和刹不对称，没有横移，转向独立限制在 0.3 rad/s。

---

## 主循环

### 事件排空与边沿检测

```rust
while let Some(event) = gilrs.next_event() {
    if let gilrs::EventType::ButtonPressed(button, _) = event.event {
        match button {
            Button::Start => toggle_enable = true,
            Button::North => toggle_head = true,
            Button::East => toggle_body = true,
            Button::South => ground_pick = true,
            Button::West => roulade = true,
            Button::LeftTrigger => kick_left = true,
            Button::RightTrigger => kick_right = true,
            Button::DPadDown => sit_toggle = true,
            _ => {}
        }
    }
}
```

**排空队列**以便下面的轴轮询看到当前状态，并捕获按钮*边沿*——按住 Start 必须只切换一次，而非每秒五十次。

### 无手柄时

```rust
let Some((_, pad)) = gilrs.gamepads().next() else {
    // 不发送任何东西：robotd 的 deadman 自己停止机器人
    if driving {
        tracing::warn!("pad gone — sending nothing; robotd's deadman holds the robot");
        driving = false;
    }
    if let Some(tap) = tap.as_ref() { tap.idle(); }
    std::thread::sleep(IDLE_POLL);
    continue;
};
```

**不发明零命令**——那会把断开的手柄伪装成某人有意停止。转换时记一次 warn（`RUST_LOG=warn` 下存活）。"手柄消失"是 journal 中机器人中途停止响应时最有用的一行。

### 模式切换

```rust
if toggle_head {
    mode = if mode == Mode::Head { Mode::Drive } else { Mode::Head };
}
if toggle_body {
    let leaving = mode == Mode::BodyPose;
    mode = if leaving { Mode::Drive } else { Mode::BodyPose };
    if leaving {
        // B 按钮退出立即将身体弹回标称位置
        notify(&mut stream, &proto::Call::RobotPose(proto::PoseParams {
            active: false, ..Default::default()
        }))?;
    }
}
```

### Start 切换——机器人拥有状态

```rust
let call = proto::Call::RobotEnable(proto::EnableParams { on: false, toggle: true });
```

> The robot owns the toggle. A local on/off belief here drifts from the robot's the moment anything else moves it — robot.relax, the shutdown sequence, either side restarting — and a stale belief turns Start into a button that does nothing every other press.

机器人拥有切换状态。本地 on/off 信念会在任何其他东西移动它时漂移——robot.relax、关机序列、任一侧重启——陈旧信念让 Start 变成每隔一次按下就什么都不做的按钮。`toggle` 翻转机器人自己的状态。

### 一次性技能

```rust
for (fired, skill) in [
    (ground_pick, proto::Skill::GroundPick),
    (kick_left, proto::Skill::KickLeft),
    (kick_right, proto::Skill::KickRight),
    (sit_toggle, proto::Skill::SitToggle),
    (roulade, proto::Skill::Roulade),
] {
    if fired {
        request(&mut stream, &mut next_id, &proto::Call::RobotDo(proto::DoParams { skill }))?;
    }
}
```

**有回答的**，因为"拒绝，以及为什么"是真实结果——可能没有踢腿策略，或空中有另一个动作。

### X 按钮按住：连续翻滚

```rust
if pad.is_pressed(Button::West) && !roulade {
    notify(&mut stream, &proto::Call::RobotDo(proto::DoParams {
        skill: proto::Skill::Roulade,
    }))?;
}
```

机器人在当前翻滚结束附近收到请求时连锁下一个翻滚，所以"按住"被拼写为"每 tick 重发"——作为通知，因为每秒 50 个有回答的请求会花时间等待回复。

### 关机保持

Select 按住 2 秒→坐下然后关机。每保持发送一次——机器人从那里拥有序列，第二个请求反正是 no-op。

### 模式切换（DPad-Up 3 秒）

```rust
let target = if roller { "walk" } else { "roller" };
// ...
match request(&mut stream, &mut next_id, &proto::Call::RobotSetMode(proto::SetModeParams { mode: target.to_owned() })) {
    Ok(Some(answer)) => match answer.result_as::<proto::IntentResult>() {
        Ok(result) if result.accepted => {
            roller = target == "roller";
        }
        Ok(result) => tracing::warn!(... "the robot refused the mode switch"),
        ...
    }
}
```

**目标被命名而非切换**——跨越从其他地方来的切换的请求要求一个模式而非"另一个"。摇杆塑形跟随机器人，仅在它同意时：被拒绝的切换在这里改变映射会让手柄用 roller 曲线驾驶行走机器人。

---

## 摇杆映射

### 死区

```rust
let deadzone = |v: f32| {
    let v = v as f64;
    if v.abs() < args.deadzone { 0.0 } else { v }
};
```

模拟摇杆很少精确停在零，没有死区机器人会爬行。默认 0.1，原型的值。

### 行走模式

```rust
Mode::Drive => proto::Call::RobotMove(proto::MoveParams {
    vx: left_y * if left_y >= 0.0 { args.max_linear } else { args.max_linear_backward },
    vy: -left_x * args.max_linear,  // vy 正方向向左；摇杆左在 gilrs 归一化后读负
    vyaw: -right_x * args.max_angular,
}),
```

### Roller 模式

```rust
Mode::Drive if roller => proto::Call::RobotMove(proto::MoveParams {
    vx: left_y * if left_y >= 0.0 { ROLLER_PUSH } else { ROLLER_BRAKE },
    vy: 0.0,
    vyaw: -right_x * ROLLER_YAW,
}),
```

### 头部模式

```rust
Mode::Head => {
    // 先发零速度——头部模式下机器人继续走是糟糕惊喜
    notify(&mut stream, &proto::Call::RobotMove(proto::MoveParams::default()))?;
    proto::Call::RobotHead(proto::HeadParams {
        neck_pitch: right_y * args.max_head,
        head_pitch: -left_y * args.max_head,
        head_yaw: -left_x * args.max_head,
        head_roll: right_x * args.max_head,
    })
}
```

头部命令喂策略的观测而非直接喂舵机，所以 2.5 弧度的 generous 范围——网络自己决定头实际走多远。

### 身体姿态模式

```rust
Mode::BodyPose => {
    notify(&mut stream, &proto::Call::RobotMove(proto::MoveParams::default()))?;
    proto::Call::RobotPose(proto::PoseParams {
        z: left_y * if left_y >= 0.0 { BODY_MAX_Z_UP } else { BODY_MAX_Z_DOWN },
        pitch: right_y * BODY_MAX_ANGLE,
        roll: right_x * BODY_MAX_ANGLE,
        active: true,
    })
}
```

---

## 声音边缘检测

```rust
const SOUND_THRESHOLD: f64 = 0.3;
if prev_rt < SOUND_THRESHOLD && rt >= SOUND_THRESHOLD {
    // RT 上升沿 → 鸭叫
}
if lt >= SOUND_THRESHOLD {
    // LT 按住 → wheee 开始
} else if prev_lt >= SOUND_THRESHOLD {
    // LT 释放 → wheee 停止
}
```

机器人切断仍在播放的声音，所以快速脉冲快速鸭叫。wheee 跟随 LT：按下开始，然后每 tick 发送保持通知——机器人将保持视为衰减的电平，所以 padd 在 ride 中途死亡会留下一个落地的 ride。

---

## IPC 协议

### notify（连续意图）

```rust
fn notify(stream: &mut UnixStream, call: &proto::Call) -> std::io::Result<()> {
    let mut line = serde_json::to_vec(&proto::Request::notify(call))?;
    line.push(b'\n');
    stream.write_all(&line)?;
    stream.flush()
}
```

无 id、无回复、无等待。用于每 tick 发送的移动/头部/姿态/嘴命令。

### request（离散意图）

```rust
fn request(stream: &mut UnixStream, next_id: &mut u64, call: &proto::Call)
    -> std::io::Result<Option<proto::Response>>
```

有 id、读取回复。用于一次性技能（踢、拾取、坐下、关机、模式切换）。**有回答的**，因为"拒绝，以及为什么"是真实结果——客户端忽略它会让操作者疑惑为什么什么都没发生。

---

## 平台差异

### Linux 上的 tap

```rust
#[cfg(target_os = "linux")]
mod tap;
```

### 非 Linux 上的空 tap

```rust
#[cfg(not(target_os = "linux"))]
mod tap {
    pub struct Tap;
    impl Tap {
        pub fn serve(_socket: &Path) -> std::io::Result<Self> {
            Err(std::io::Error::other("the raw pad tap reads evdev, which only Linux has"))
        }
        pub fn watch(&self, _pad: &gilrs::Gamepad) {}
        pub fn idle(&self) {}
    }
}
```

Mac 上 padd 仍驱动手柄——这是 crate 文档中的工作台设置，因为调试设施失去它是糟糕的交易。它不服务 tap，`robotctl monitor` 找不到 socket 并如实说明。

---

## 与其他文件的关系

- **`tap.rs`**：原始输入 tap，读取 evdev 原始流供 `robotctl monitor` 使用
- **`duck_ipc_proto`**：IPC 协议类型（Call、Request、Response、各种 Params）
- **`gilrs`**：手柄库，提供事件、轴、按钮读取
- **`robotd`**：意图 API 服务端，padd 是其客户端
- **`configd`**：配对在那里（不在 padd 中），因为配对需要 root 和 BlueZ

---

## 关键踩坑点总结

1. **Start 切换：机器人拥有状态，本地不存信念**：本地 on/off 信念会在 robot.relax/关机/重启时漂移，让 Start 变成每隔一次按下就什么都不做的按钮。用 `toggle: true` 翻转机器人自己的状态。

2. **无手柄时不发零命令**：发零命令会把断开的手柄伪装成有意停止。让 robotd 的 deadman 自己处理。

3. **头部/身体姿态模式先发零速度**：头部模式下机器人继续走是糟糕惊喜。deadman 最终会捕获，但 explicit 更好。

4. **模式切换只在机器人同意后改本地映射**：被拒绝的切换如果改了本地映射，会让手柄用 roller 曲线驾驶行走机器人。

5. **X 按钮按住用 notify 而非 request**：50Hz 有回答的请求会花时间等回复。press 时已经得到了真正的回答，hold 时用通知重发。

6. **RT/LT 声音用上升沿/电平检测**：RT 上升沿触发鸭叫，LT 电平控制 wheee。释放 LT 发送 `hold: Some(false)` 立即停止。

7. **空手柄轮询 500ms 而非 20ms**：padd 从启动就运行，大多数时候没有手柄。以控制速率空转是每 20ms 一次无意义唤醒。

8. **gilrs 的 LeftTrigger/RightTrigger 是肩键（LB/RB）**：模拟扳机是 LeftTrigger2/RightTrigger2。这是命名陷阱。

9. **vy 正方向向左，摇杆左读负**：符号方向需要注意。

10. **tap 失败不致命**：tap 是调试设施，创建失败时记录 warn 继续驾驶——不能让调试设施阻止机器人驾驶。
#（注：内容由AI生成）
