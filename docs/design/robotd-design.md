# `robotd` —— 控制循环

状态：草稿 · 日期：2026-08-20 · 负责人：pierre

实现 [`architecture.md`](architecture.md) §1 的 `robotd` 行，覆盖 [`roadmap.md`](../project/roadmap.md) M3。被吸收的原型是 [`apirrone/microduck_runtime`](https://github.com/apirrone/microduck_runtime)，全文称为*the runtime*。

**仅 alpha 变体，仅 Radxa，仅 v2 `imu_to_dxl` 板。** v1/v1.5/v1.6、其他四个 IMU、三个摄像头和 Pi 被丢弃，每个发货的 policy 都是 `alpha_*`。轮式配置作为一个参数开关存活——`policy.mode = "roller"`（§4.2）——因为它选择的是 policy 集和调优预设，而非硬件变体。

## 1. 它的形态

一个进程、一条串行总线、一个 50 Hz 循环。循环在一次事务中读总线上全部十六个设备，决定十五个关节目标，并写回。其他一切——客户端、健康、遥测——挂在循环上，永远不能阻塞它。

### 1.1 总线，以及谁拥有端口

十五个舵机和 `imu_to_dxl` 板共享一个 UART。没有第二条总线也没有第二个端口：

```text
                     robotd — control thread
                              │
                              │  duck_control::bus::DynamixelIo
                              │  serialport · TIOCEXCL
                              ▼
         /dev/ttyS2 · 1 Mbps · Dynamixel protocol v2
                              │
    ┌────────────┬────────────┴───────┬──────────────────┐
    │            │                    │                  │
  id 200       20–24                30–34             10–14
 imu_to_dxl   left leg        neck · head · mouth     right leg
  v2 board    5 servos             5 servos           5 servos
```

IMU 是 `id 200` 且在与舵机*相同*的 `sync_read` 中读取，因为硬件就是如此：v2 板坐在 Dynamixel 总线上，从舵机应答的同一寄存器块提供片上 SFLP 四元数。一块板、一条代码路径、无 IMU 抽象。它在 id 向量中列第一个，所以它在舵机突发之前应答。

**一次一个拥有者，且没有硬强制。** `serialport` 设 `TIOCEXCL`，把第二个*非特权* open 变成 `EBUSY`——但 `robotd.service` 以 root 运行，因为电机控制需要字符设备，而 root 不受那个标志阻止。所以排除是安排的而非强制的，每个其他主张者被刻意挡在端口外：

- **控制循环**在 daemon 运行期间拥有它。
- **`robotd init`** 自己打开端口，作为 root 它会在 daemon 运行时*成功*——两个写者在一条总线上交错数据包，读作硬件故障。所以它要 daemon 停止，这正是 `robot.init` 和 `robot.relax` 作为 IPC 方法存在的原因（§3.3）：daemon 从循环内部服务两者，所以其他东西根本不必打开总线。`init` 是 daemon 没运行的机器人的逃生口。
- **`serial-getty@ttyS2`** —— Armbian 默认在 UART2 上跑登录控制台，持有端口的 `agetty` 让每个舵机对其他一切不可见。`scripts/setup-board.sh` 屏蔽该 unit；`fuser -v /dev/ttyS2` 点名 `agetty` 是发现方式，且它仍是回答"谁拥有总线"的命令。
- **runtime**，在更粗粒度上：它驱动同一条总线，所以一块板子跑 runtime 或 `robotd`，从不同时跑，unit 用 `Conflicts=` 说明（§5.2）。

### 1.2 谁跟 `robotd` 说话

```text
   ┌──────────┐   robot.move / robot.head      ┌─────────────────────┐
   │  padd    │───(notifications, 50 Hz)──────►│                     │
   │ gamepad  │   robot.stop / robot.enable    │                     │
   └──────────┘───(requests, answered)────────►│                     │
                                               │   /run/robotd.sock  │
   ┌──────────┐   robot.subscribe              │   JSON-RPC 2.0      │
   │ robotctl │──────────────────────────────► │   NDJSON            │
   │ monitor  │◄──robot.state (decimated)──────│                     │
   └──────────┘                                │                     │
                                               │                     │
   ┌──────────┐   robot.health                 │                     │
   │ updaterd │──robot.safeToRestart──────────►│                     │
   │          │  robot.modelApi                └──────────┬──────────┘
   └──────────┘                                           │
        │                                                 │
        │ on_apply: systemctl restart robotd              │
        └─────────────────────────────────────────────────┘

   ┌ the two transports ────────────────────────┐
   │  mediad — WebRTC + JSON-RPC relay          │  phone, browser and LLM
   │  btd    — BLE, a subset of the same API    │  clients arrive through here
   └────────────────────────────────────────────┘
```

其中每个说同样两种词汇——意图进、状态出——所以 `mediad` **中继**帧而非翻译它们（§3.1、§3.2）。`updaterd` 只问问题：更新路径里没有东西能命令电机。

### 1.3 crate 边界

```text
duck-ipc-proto/  wire contract — serde only; no tokio, no http, no crypto
duck-control/    robot model · bus · IMU · RobotIo · obs · policy · safety
                 everything between reading the bus and writing it
                 no tokio, no sockets, no systemd
robotd/          the process: socket, JSON-RPC, systemd, health reporting
robotd-params/   the startup parameters: schema, defaults, validation (§4.2)
kinematics/      the MJCF model, forward kinematics, head and hand chains
odometry/        where the robot is, from foot contacts and the IMU
sounds/          the voice: synthesis, per-robot personality, the chorale's score
pet-detect/      the camera-side detector robotd polls, off the loop
robotctl/        CLI
padd/            gamepad → intents
updater/         engine + updaterd
```

该列表中 `robotd/` 以下的一切是它驱动的**库**，而非服务：没有自己的 tokio 运行时、没有 socket、没有 systemd 启动的东西。它们是独立 crate，原因与 `duck-control` 相同——编译器把 daemon 关注点挡在它们外面，特别是 `kinematics` 有两个消费者（`odometry` 和 `robotd` 的头部 FK），否则各自会长出一份模型副本。

`duck-control` 持有读总线和写总线之间的一切；`robotd` 是它周围的进程。编译器强制执行该边界，这阻止 daemon 关注点泄漏到控制代码——且意味着如果 runtime 在过渡期间需要消费它，该 crate 可被提升到自己的仓库，而那不是重写。

`safety` 持有 `RobotIo`，所以 policy、控制器和每个客户端都能*提议*目标，没有一个能发送目标。那是借用检查器，不是约定（§2.4）。

### 1.4 tick

50 Hz，在自己运行时上的一个 `tokio` 任务，所以 IPC 工作不能挡在它前面。每次 tick 两次总线事务，加每秒一次的第三次：

```text
read()          one sync_read  · IMU board + 15 servos · registers 124–136   (§2.1)
decide          observation → policy → targets → clamp                 (§2.2–§2.4)
write()         one sync_write · goal positions
publish         atomics always; a state frame only if someone subscribed     (§4.1)

every 1 s       slow_sensors() · registers 144–146 · voltage + temperature   (§2.1)
```

数据去哪，每期一次：

```text
  Dynamixel bus
       │  one sync_read: IMU board + 15 servos, one transaction
       ▼
   Sensors ──────────┬──────────────────────► safety.observe ──► fallen? (debounced)
   joints, IMU       │
                     ▼
              Observation::build  ◄──── Command ◄── gate(deadman) ◄── intent snapshot
                     │              (twist, head, body = nominal)
                     │  [f32; 61]
                     ▼
              Policy::infer ──── roulade > kick > ground pick > sit/rise >
                     │           stand (by |twist|, or forced) > walk
                     │  [f32; 14]   — mouth excluded
                     ▼
              home pose + scale × action ──► low-pass on head and legs
                     │
                     │  [f64; 15] proposed targets
                     ▼
        ╔═════════════════════════════════════════════╗
        ║  safety.apply   ← owns the only RobotIo      ║
        ║  · refuse non-finite                         ║
        ║  · clamp to actuator range                   ║
        ║  · no fall gate — the verdict only reports   ║
        ╚═════════════════════════════════════════════╝
                     │  sync_write goal positions
                     ▼
              Dynamixel bus
```

以及它周围的决策，上面的数据流没显示：

```text
  startup
    ├─ open the bus, waiting              an unpowered board is fixed with a switch,
    │                                     not by abandoning the loop
    ├─ Safety::new(io)                    safety takes the RobotIo; nothing else can write
    ├─ read() → hold = the pose the robot is already in    ── never move on start
    └─ policy
         disabled ─────────────► controller = None                    healthy
         loaded   ─────────────► controller = Some
         failed   ─────────────► controller = None + policy_error   unhealthy

  each tick
    read ─┬─ ok  ─► clear the consecutive-error count
          └─ err ─► count++, sensors = None   (the tick still runs)

    observe → fallen?     published; gates nothing

    driving = enabled ∧ policy loaded ∧ sensors this tick ∧ ¬limp-fall

    edges ─┬─ started driving ──► controller.reset()
           │                      else a stale last action, or a filter anchored to
           │                      where the robot was a minute ago, shows up as a lurch
           └─ stopped driving ──► hold = current pose, captured once
                                  re-reading each tick would sag under gravity

    driving ─┬─ limp-fall ─► the sequence's own targets and gain    (§2.4.1)
             ├─ yes ──────► step() → targets, gain, the active net's name
             └─ no  ──────► targets = hold, default gain, "held"

    safety.apply(targets, hold, gain)

    publish ─┬─ atomics            always      → robot.health, safeToRestart
             └─ state frame        only if subscribed   → robot.state
```

`driving` 的四个条件各是关键的。`sensors this tick` 是不明显的那个：失败的读取没留下东西可构建观测，编造一个会给 policy 喂一个不存在的机器人。`¬limp-fall` 是不是拒绝的那个——在 limp-fall 序列拥有机器人期间（§2.4.1）policy 刻意不驱动，目标来自序列。

循环保持为带 `interval` 的 `tokio` 任务。它没被做成实时（§5.4）。runtime 的一个变更值得点名：**`MissedTickBehavior::Skip`**。`Burst` 把积压背对背触发并把电机命令堆叠在一起。`Delay` 以不那么明显的方式错——它在每次后把下一 tick 调度在 *now + period*，所以每次唤醒延迟被加到周期而非被吸收，循环漂移得比它配置的速率慢。`Skip` 保持原调度并丢弃错过的 tick，那正是控制循环想要的。把感知移到 `mediad` 免费移除了大部分与循环竞争的东西。

### 1.5 不变量

五个性质，其他一切围绕它们安排。每个在某处被强制执行，不只是意图：

1. **`safety` 之上没有东西能写电机。** 它拥有唯一的 `RobotIo`；借用检查器是强制（§2.4）。
2. **`robotd` 从不因为进程启动而移动机器人。** 更新重启让站着的机器人保持站着（§3.3）。
3. **控制循环从不等待客户端。** 意图是原子加载，遥测是滞后丢弃（§4.1）。
4. **健康被发布，从不被询问。** 卡死的循环报告自己不健康而非挂起调用者（§3.4）。
5. **只有 release 能被归咎的东西才可以到达健康判定。** 电池和温度是描述，从不是回滚输入（§3.4）。

## 2. 控制路径

### 2.1 总线层和 `RobotIo`

`rustypot` 上的薄层：open、一次组合 `sync_read`、`sync_write` 目标位置、扭矩使能、增益、慢传感器读取、以及启动寄存器检查。新写而非移植，但**数字借自 runtime**，每个带注释说明：

- `RAD_PER_SEC_PER_COUNT = 0.229 × 2π/60`，以及位置计数↔弧度转换。
- 来自 `check_and_fix_config` 的 EEPROM 寄存器，启动时*断言并纠正*：`return_delay_time=0`、`baud_rate=3`、`pwm_slope=255`、`shutdown=52`。第一个是关键的——在 XL330 默认 250 时每个设备 500 µs 周转，所以十六个设备每次 tick 约 8 ms，预算的 40%。被工厂复位或换入的舵机到达时是 250，所以检查移除了一整类"为什么这台机器人慢"。`shutdown = 52` 是在过载、过热和输入电压故障上锁存的错误掩码。
- 位置 P 增益写入时 I 和 D 为**零**，runtime 的 `--ki`/`--kd` 默认。这些是 RAM 寄存器，所以掉电恢复舵机的工厂值，而工厂 D 不是零：留在原处它会阻尼舵机内部 PID，机器人在*相同* kP 下运行得明显更软。不是任何人做的调优选择，所以它被钉住而非暴露。

**每次 tick 两次总线事务，每秒一次第三次。** Tick 读 124–136 的连续块（pwm、电流、速度、位置）。电压和温度在 144–146，在它末尾之后八字节，中间有十二字节没人要的轨迹寄存器——所以它们每秒在自己的事务（约 1 ms）中一起采样，而非把 tick 的读加宽到 50 Hz 下每舵机 22 字节。采样间隔与测量达到速率的窗口相同，所以一个时钟驱动两者。

电压在舵机间取平均：十五个都在一个电池组上，所以单次读取是同一测量加更多噪声，应答零的设备被过滤掉而非当作电池半扁平均进去。温度*不*取平均——调用者得到每个关节并**按名**报告最热的，因为保持蹲姿的膝盖远高于嘴，十五个舵机的均值隐藏了接近过热关断的那个。

静默的舵机不产生短应答：`rustypot` 的 `sync_read` 等待每个 id，一个不应答则整个事务失败。所以两次读取都是全有或全无，调用者保持其上一个样本而非把一次丢失当新闻。

**板温是第三个来源，且根本不在总线上。** SoC 热区中最热的，在同样每秒一次的采样中从 `sysfs` 读取（`robotd/src/soc.rs`）。它住在 `robotd` 而非 `duck-control`，因为它是 Linux 板的性质而非机器人的性质——这也是为什么电机总线不行时它仍能应答，而那正是它获得位置的时刻：通风口被堵的板子和舵机死了的机器人在你能看到两个数字之前是同一症状。取跨区最大值而非按名取一个区，所以接线不同的板子不能静默省略正在攀升的那个。

**IMU 陈旧被跟踪，永久。** "读取成功但板返回相同样本"会给 policy 喂死方向，除非有人计数否则不可见，且已知会发生。总线层记住上一个块，计数相同的后继，并在运行足够长有意义时——半秒，与 SFLP 解码器在称芯片输出为测量之前等待的时间相同——在 journal 中说明。对第一个重复块警告会教会所有人忽略消息；过阈值后它被限速，因为停止刷新的板子每次 tick 产生一个，50 Hz 的相同警告会挤掉 journal。

接缝：

```rust
trait RobotIo {
    fn read(&mut self) -> Result<Sensors>;                 // joints + IMU, one transaction
    fn write(&mut self, targets: &JointTargets) -> Result<()>;
    fn set_gain(&mut self, kp: u16) -> Result<()>;         // one write per joint, not per tick
    fn set_torque(&mut self, on: bool) -> Result<()>;      // idem
    fn slow_sensors(&mut self) -> Result<SlowSensors>;     // voltage + per-joint temperature
    fn imu_stale(&self) -> ImuStale;                       // defaulted
    fn imu_ready(&self) -> bool;                           // defaulted
}
```

`set_gain` 和 `set_torque` 是每关节一次事务，这就是为什么两者都不是每 tick 调用，以及为什么 bring-up 是状态机而非循环持续应用的标志（§3.3）。两个固有方法坐在 trait 外因为只有 `init` 用它们：`present_positions`（较轻的读取，启动时用一次以采纳机器人已在的姿态）和 `interpolate_to`（阻塞的线性斜坡——刻意阻塞，因为它运行时其他什么都不该跟总线说话）。

两个实现：`DynamixelIo` 和 `FakeIo`（脚本化样本，可选冻结或按需失败）。`FakeIo` 是测试套件跑的对象，这就是 `cargo test` 不需要硬件、网络和 Docker 的原因。

**两者都没有 `cfg` 门控出 macOS。** 门控本想让 `serialport` 不出现在笔记本的依赖树里，但 `rustypot` 和 `serialport` 都能在那干净构建，所以它什么都没换来却损失了无需板子就能类型检查总线层的能力——那正是最可能被没有板子的人编辑的代码。只有打开真实端口的入口点被门控，所以 Mac 构建仍拒绝假装它有机器人：`robotd --fake` 是笔记本路径，且必须显式请求而非回退到。

### 2.2 一个观测构建器

每个 alpha policy 是 `obs[1,61] → actions[1,14]`——跨行走、站立、地面拾取、踢球和坐验证。所以恰好有一种布局：

```
[ gyro(3) | projected_gravity(3) | joint_pos(14) | joint_vel(14) | last_action(14) | command(13) ]
                                                                    command = vel(3) + head(4) + body(6)
```

关节全程排除嘴；动作映射回 15 个电机槽，索引 9 留零。51/54D 遗留、49D 轮式和 85D 跟踪布局随变体消失。

曾是唯一存疑部分的命令块已定——从原型的 `control_step` 读出而非猜测：

```text
48..51   vx, vy, vyaw
51..55   neck_pitch, head_pitch, head_yaw, head_roll
55..57   body x, y      — hardcoded zero, unbound in training
57..60   body z, roll, pitch
60       body yaw       — hardcoded zero, unbound in training
```

关于它的三件事各自合理且错：

1. **全零 body 是名义编码**，不是占位符——x、y 和 yaw 字面硬编码零作为"未绑定"，z/roll/pitch 除非 body-pose 模式活动否则为零。
2. **头部目标骑在命令里，不加在 policy 输出之上。** 原型在不同模式下两者都做，并把事后加法门控在 `if !new_cmd_obs` 后，注释"head\_offsets are a COMMAND fed via the obs vector instead — don't double-add it here"。两者都做会把头部弯两次。
3. **body 块顺序是 `z, roll, pitch`**——不是 `z, pitch, roll`。交换后两个会在被要求前倾时把机器人侧倾。

### 2.3 Policy

形态与 runtime 相同，刻意。`robotd/src/control.rs` 持有优先级链和来自 `control_step` 的每个数值默认，它取代后者：

```text
skill windows ← advance / expire (roulade window, kick timer, ground-pick phase, sit↔stand rise)
command       ← the caller's smoothed command, re-encoded for the active skill
net           ← roulade > kick > ground pick > sit/rise > stand-by-magnitude (or forced) > walk
action        ← ONNX
targets       ← home pose + action_scale × action
filters       ← first-order low-pass on head and legs
```

原型的两个微妙之处值得点名因为它们容易被意外"修复"。**踢球窗口以站立调优运行**——踢球的观测携带全零命令，站立转换恰好在那时触发，所以踢球以 `standing_action_scale` 和软化的站立增益运行。保留，因为踢球是针对它调优的。**sitstand *起立*也以站立增益运行**（其命令全零）而*坐下*不（其姿态标志让扭转幅度为 1）。同样机制，同样原因。一个刻意的分歧：原型通过在转换间保存和恢复 `action_scale` 来跟踪站立动作 scale，那可能在坐→站循环后留下过时值直到下一次行走；这里 scale 和增益每 tick 从活动状态重算。

Policy 文件来自 params 文件中的路径，默认指向 release 目录——所以普通更新携带针对二进制训练的 policy，开发者把路径指向自己的 `.onnx` 并迭代而无需切 release。

一切在**加载**时验证，而非推理时：观测宽度、动作数，以及 ONNX Runtime 是否存在。每个网络必须 61 输入 14 输出，在加载时检查而非走步中途发现。runtime 还发货使用遗留 3 值命令的 51-D 家族；那些只在其 `--new-cmd-obs=false` 路径下加载，`robotd` 以 `observation width is 51, expected 61` 拒绝它们。循环开始前跑一次预热推理，既把首次调用代价付在热路径外——在那里它看起来与错过截止时间一模一样——又证明 dylib 已解析。

**`ort` 在 ONNX Runtime 缺失时 panic。** 它在 `setup_api` 内 `expect`，在从任何 API 调用可达的惰性路径上，所以不能作为错误捕获。任其发展会杀死控制线程：没有 tick 落地，健康永远报告"循环未完成一个周期"，所以 daemon 看起来卡死而非点名原因——比本设计拒绝的崩溃循环更糟。`policy::ensure_runtime` 因此用与 `ort` 相同的加载器和搜索规则探测 dylib，在触碰 `ort` 之前，所以缺失的库变成普通错误。

**`policy.enabled` 区分"不想要 policy"和"policy 坏了"。** 第一个是健康的，是实验台 updater 测试的正确配置；第二个不健康，所以 updater 回滚 release。合并它们要么让实验台机器人看起来坏了，要么让不可用的 bundle 通过门控。`robotd --no-policy` 设置它，门控测试用它，因为 CI 和笔记本都没装 ONNX Runtime。

ONNX Runtime 是**板子前置条件**，由 `scripts/install.sh` 安装，不随 release 发货。它比 daemon 变化少得多，且每个 artifact 里约 20 MB 会无谓放大每次更新。代价是缺它的板子安装启动正常然后不能走——这就是健康报告搜索路径的原因。

没做，且刻意：预绑定 ONNX 输入/输出张量。当前路径每次推理分配一个 61 浮点向量，50 Hz 下约 244 字节。值得在板上测量后再优化。

**从 runtime 继承因为它工作**——头和腿低通滤波器、动作 scale、电压自适应缩放、站立转换增益变化。这些是可调参数（§4.2），不是要重访的决策。低通 alpha 特别是 alpha policies *训练*用的值，所以它们必须匹配训练否则迁移降级。规则是不回归已经在跑的东西。

### 2.4 Safety

`safety` 拥有唯一的 `RobotIo` 写句柄。没有 policy 也没有客户端有，所以它之上的东西*不能*命令电机——不变量是结构性的而非被记住的。与 updater 关于其恢复路径的论证相同：只在已经出错时才运行的代码最可能悄悄坏掉，所以让坏状态不可表示。

两条规则，无条件：

- **非有限拒绝。** `NaN` 目标不被钳制，被直接拒绝。
- **范围钳制。** 目标被保持在执行器行程内，每 tick，无论 policy 要什么。这是*执行器的*范围，不是逐关节解剖学限制——它抓 `NaN`、荒谬的动作 scale 和垃圾张量；它不会阻止关节被驱动到机械上不明智的地方（§9.3）。

加上命令上的 deadman：如果意图停止到达，速度归零。**停止不是软瘫**，且区别重要——失去通信让机器人*站着不动*，因为站立是双足的安全状态；失去平衡是不同事件，这层不回答它。

**跌倒判定是报告，不是规则。** 它每 tick 计算——躯干帧中的投影重力，去抖 0.2 秒以便坚实踏地不是跌倒——并发布，且它**什么都不门控**：跌倒的机器人像直立的一样被使能、init、驱动和发送技能。那是刻意的。在地上正是某人需要那些调用工作的时候，而一重力误读倾斜就屈服的机器人是在有人处理它时不断坐下的机器人。这也是让跌倒的答案住在这层之上无需豁免的原因：早期版本这里有 `fall_limp` 门和 `fall_recover` 自动起立，两者都被移除，因为恢复必须绕过才能工作的安全规则不是安全规则。

#### 2.4.1 跌倒是第三个事件

上面的跌倒判定回答"机器人倒了吗"。那对报告侧躺的机器人是对的问题，对软化落地是错的：重力超过 `fall_gravity_z` 持续 200 ms *就是*机器人在地上，值得行动的窗口那时已关。

所以 `limp_fall`（自它在机器人上验证后默认开）跑第二个独立的检测器——`duck_control::fall`——在速率而非位置上。投影重力随躯干旋转，所以 `ġ = −ω × g` 精确且直接来自同一 12 字节 IMU 块中的陀螺；外推约 0.3 秒说明重力往哪去。它在机器人已倾斜（约 26°）、仍在翻倒而非恢复、且预测超过跌倒阈值时触发——去抖三 tick。改为对 SFLP 四元数求导会把滤波器的滞后加到那个唯一价值在于早的数字上。

它换来的不是落地本身而是落地后的起立。站立 policy 能干净地把已知姿态中静止的机器人拉起来，把乱动的机器人拉起来要在对着地板试几次行走增益，那是电机负载的来源。所以序列把跌倒从 policy 手里拿走：以 `gain_limp` 软瘫跟随关节下行，等陀螺安静，在约 1 秒内回到站立姿态，交回。交回不过如此——扭转全程保持零，所以命令幅度选择站立网络，那就是起立。

它按构造在 policy 路径外运行：整个序列 `driving` 为假（§1.4），所以目标来自序列而非控制器，且像其他任何东西一样通过 `apply` 到达电机——无豁免、无后门。`safety` 里什么都不需要被告知，恰恰因为判定什么都不门控：姿态斜坡像任何其他 tick 一样移动躺在地上的机器人。

调优是特性，且它不对称：假阳性是机器人*造成*的跌倒，比它试图避免的生硬落地更糟。默认刻意偏向晚。

耗尽的电池是另一个未经要求移动机器人的东西：用 `safety.battery_empty_shutdown`（默认开），在平滑电压达到空电底限时让机器人坐下并断电。EMA 在约 10 秒内移动，所以负载瞬降不能触发它——达到 6.6 V 需要真正耗尽的电池。

### 2.5 机器人模型

仅 alpha 的 Rust 常量：15 个关节、Dynamixel ID `20–24 / 30–34 / 10–14`、名称、`DEFAULT_POSITION`、执行器范围。与 runtime 的 `motor.rs` 相同的表，去掉三个死变体。恰好有一个机器人；第二个修订可以是第二个表。

其中两个表在其 crate 之外是关键的。`JOINT_NAMES` *来自* `duck-ipc-proto`，因为线上按位置索引 `joints` 和 `targets`，两个顺序不能漂移——`const` 断言让"不能"成真。且 `DEFAULT_POSITION` 必须匹配训练环境中的 `HOME_FRAME`：policy 观测关节位置*相对于* home 姿态，所以这里的差异是 14 个观测槽上的常量偏移。

## 3. API

### 3.1 意图进

两种词汇——意图进、状态出——JSON-RPC 的两个消息族精确映射到它们：

```jsonc
// continuous: notifications, no id, no reply, last-writer-wins
{"jsonrpc":"2.0","method":"robot.move","params":{"vx":0.2,"vy":0.0,"vyaw":0.4}}
{"jsonrpc":"2.0","method":"robot.head","params":{"neck_pitch":0.35,"head_pitch":0.35,
                                                 "head_yaw":0.0,"head_roll":0.0}}

// discrete: requests, answered
{"jsonrpc":"2.0","id":7,"method":"robot.stop"}
{"jsonrpc":"2.0","id":8,"method":"robot.enable","params":{"on":true}}
```

一切是弧度、躯干帧、右手系，符号在协议定义中固定。runtime 携带 `--laser-track-yaw-sign`、`--laser-track-pitch-sign`、`--laser-fk-pitch-sign`、`--laser-fk-neck-sign` 和 `--imu-z-rotation-deg` 因为那个约定从未被写下，每个消费者经验地重新发现。把它写进协议删掉这类问题。

在 50 Hz 下，连续意图作为通知意味着无响应流量。当它们后来通过 WebRTC 传输时，通知路由到不可靠的 `teleop` 通道，请求路由到可靠的 `control` 通道——那是 `architecture.md` §5.2 要求的，从消息族自然得出而非任何人要记的规则。

**扭转和头部刻意是分开的槽。** 单一组合槽需要读-改-写来更新一个字段，两个客户端——一个手柄驱动身体一个东西驱动头部——会静默丢失彼此的更新。分开的槽让每个在实践中单写者，所以 last-writer-wins 名副其实。每个槽被打时间戳，因为循环真正的问题从不是"值是什么"而是"它多老"；那是 deadman 读的。

`look`（注视方向）推迟；两种注视形式都会暴露，它们之间的仲裁是 last-writer-wins 无混合。

### 3.2 状态出

一个流，可订阅，逐订阅者降采样。它必须报告**被拒绝**的，不只是发生了的——一个显示摇杆前推而机器人不动、无解释的 teleop UI 是不可用的，且 safety 不断钳制东西：

```jsonc
{"method":"robot.state","params":{
  "t":1234.567,
  "move":{"requested":[0.4,0,0],"applied":[0.15,0,0],"limited_by":["max_velocity"]},
  "policy":"walk", "safety":{"fallen":false,"limp":false},
  "loop":{"hz":49.8,"missed":0},
  "battery":{"volts":7.62,"percent":64}
}}
```

**电池同时带伏特和百分比**，这里和 `robot.health` 中。映射——6.6 V 空、8.2 V 满载下，NP-F550——住在 `duck_control::model::battery_percent` 且已应用后传输。原型只发伏特，app 从自己的常量重新推导百分比，这就是同一电池在两个屏幕上显示两个不同数字的方式。画电池图标的客户端不该需要知道这台机器人发货哪种电池。没有电量计：测量是舵机自己的供电电压（§2.1），所以它在负载下瞬降静止时恢复。

`robotctl monitor` 和稍后 app 用同一 payload。这取代了 runtime 在 9870 上的 180 字节帧、9871 上的 JPEG 流、9872 上的 UDP 命令 socket、9874/9875 上的 maploc 端口以及 web hub 的 `/state.json`。今天加一个字段意味着编辑四个可能静默不一致的地方；这里意味着一个结构体，旧客户端忽略它们不知道的。

`robot.subscribe` 把连接变成流；循环发布到有界广播且从不等待订阅者，所以慢客户端得到间隙而非施加背压——updater 已对进度使用并文档化的模式。降采样是服务端逐订阅者的，所以 10 Hz 仪表盘真的比 50 Hz 数字孪生耗机器人更少。

**确认点名 policy。** `robot.subscribe` 用 `SubscribeResult` 应答：此进程配置了哪些网络（按文件名），以及无东西驱动时的一句话——params 中禁用，或想要但不可加载。那属于握手而非帧，因为它在进程存活期间不变，而帧上的 `policy` 回答不同问题：哪个网络驱动了*这个 tick*。两个不同步态的 release 都报告 `walk`，"这是哪个网络？"是比较它们的人问的第一件事。把它放帧上会在控制线程上每 tick 分配两个字符串给一个从不不同的答案。

两个容易错的细节。**没人订阅时什么都不组装**——那是机器人的正常状态——因为构建帧会在不该无理由访问分配器的线程上分配。且限制名是**为线上拼出的**而非从 Rust 枚举派生，所以重命名变体不能静默破坏基于 `limited_by` 分支的客户端。

### 3.3 Bring-up：`enable`、`init`、`relax`

**`robotd` 从不自己移动机器人。** 启动时它读当前位置，采纳为目标，不碰扭矩。Dynamixel 在进程死时保持它们最后命令的目标，所以重启让姿态不变且无间隙——机器人在更新中站着而没察觉。启动时插值到默认姿态会让每次更新重启移动站着的机器人：跌倒风险，且当被测的是 updater 时是混杂因素。

`robot.enable` 曾只翻转一个标志。扭矩来自 `robotd init`，一个自己打开电机总线的独立子命令——所以它需要 daemon 停止、不出现在任何文档中、且在新机器人上按 Start 什么都看不见：policy 跑、循环写位置、舵机忽略。所以循环有 bring-up 状态，显式 `robot.enable` 推进它：

```
Limp ──enable (policy loaded, a fresh sample)──▶ Homing (torque on, 2 s ramp) ──▶ Ready ──▶ policy drives
```

**旧规则保护的不变量不变：这里什么都不因为进程启动而发生。** 被更新重启的 `robotd` 发现 `Limp`，不要扭矩，让站着的机器人站着——`a_restart_asks_for_no_torque` 精确断言这一点，基于无任何写而非 `false` 的写。改变的是"从不碰扭矩"是比它捍卫的性质更宽的规则，且它在每次驾驶前放了一个手动步骤。

两个条件门控 bring-up，各有原因：

- **已加载的 policy。** `enable` 意味着"使能 policy"；给关节上电跑一个禁用或加载不了的 policy 会在坏 release 上把机器人立起来然后托住。
- **新鲜样本。** 斜坡从关节所在处开始。从没人读过的位置开始是斜坡要避免的猛冲。

**倒下不是其中之一。** 早期版本在那里拒绝，那时 `apply` 以软瘫增益托住跌倒的机器人，斜坡会写一个不可能发生的起立；它在判定停止门控任何东西时去掉了（§2.4）。在躺在地上的机器人上启动正是某人要求它重新站起来的方式——斜坡像任何其他一样运行，站立 policy 从那接手。

policy 再次被禁用时扭矩*不*下降：机器人保持姿态，这是"站着的机器人保持站着"在这一侧的意思。

`robot.init` 和 `robot.relax` 是同样两个转换，直接要求——因为"站起来"和"松开"是各自的决定，且直到现在第一个是自己打开电机总线的子命令（§1.1）第二个根本不存在。两者都由 `robotd` 服务，所以都不需要 daemon 停止，都不能在控制循环做同样事时写总线。`init` 刻意不需要 policy：要求没有行走网络的机器人站起来是合理的，且这让 bring-up 可测试，因为 CI 没有 ONNX Runtime。

它们作为*请求*到达，循环每 tick 取一次而非作为它持续应用的标志：一次 `set_torque` 是每关节一次总线事务，所以电平会把十六次写放进每个 tick。后来的请求替换未读的更早请求——20 ms 内被要求站起来然后松开，第二个是本意。且 `relax` 清除 `enabled`，否则下一 tick 会看到被要求驾驶的机器人并直接把它立起来。

两者都不能通过 BLE 到达。让机器人掉地上的手机按钮不是该提供的，且站起来同时移动每个关节，那需要要求的人看着机器人。

### 3.4 健康，以及什么可以到达判定

| method | answer |
|---|---|
| `robot.health` | **the loop is meeting its deadline** — from achieved rate and missed-deadline count — plus a description of the robot the verdict never consults: loop, bus, IMU, battery, servo and board temperature |
| `robot.safeToRestart` | false while the policy is enabled and the robot is moving |
| `robot.modelApi` | constant |
| `robot.remoteSessionActive` | `false` — `mediad` owns the real answer |

健康由 IPC 侧从循环发布的原子计算——最后 tick 时间戳加计数器——从不通过问循环。这让*卡死*的循环报告自己不健康而非挂起调用者。

跑在目标 60% 的循环是活的，应答每个请求，且严重坏掉。让那个区分真实是为什么控制循环在任何会走的东西之前被构建（§5.1）。

**什么可以和不可以到达判定。** `healthy` 和 `degraded` 是更新系统的输入，所以只有*release*能被归咎的条件才可以设置它们——那正是 `degraded` 已存在来对未上电实验台板强制执行的。答案上的其他一切是**描述**，且没有自动决定可以读它：电池、电机温度和循环/总线/IMU 计数器。基于电池门控意味着在低电量电池上更新的机器人回滚 release，然后用同一低电量电池评判替代，直到有人搞清楚为什么之前都不能更新。电机温度在炎热下午会一样。

**为什么它们仍一起传输。** 一个方法，因为问题一次到达：行为古怪的机器人被问"怎么了"，没有背后数字的判定只会开始第二轮问题。循环段携带判定正是从其算出的数字，所以 `unhealthy: control loop at 43.9 Hz` 可与 `missed = 0` 并排读——那区分循环被晚唤醒和循环做太多事，两者有不同修复。`robotctl health` 加来自 `updaterd` 的软件一半并打印两者。

`safeToRestart` 在 policy 使能且机器人移动时为假：在步态中途重启电机控制是机器人跌倒的方式（`updater-design.md` §7.2）。

### 3.5 维护是独立命名空间

`init`、紧急扭矩关闭、校准和原始关节写入不是意图。它们住在自己的命名空间，所以中继的逐传输允许列表能让它们远离远程传输。信令门控决定*谁连接*；它不说远程操作员也是机修工——且 `update.*` 到达 DataChannel 意味着远程对端能触发回滚。

## 4. 循环周围

### 4.1 循环读快照，从不等待

意图和 params 由 IPC 线程发布，循环作为单次原子加载读取。没有东西能对循环施加背压，没有请求同步进入它。遥测通过有界广播出去，慢订阅者得到间隙——updater 的 IPC 层已使用并文档化的模式。

```text
   IPC tasks (tokio, multi-thread)        control thread (own runtime, 50 Hz)
   ═══════════════════════════════        ═══════════════════════════════════

     robot.move  ──► ┌────────────┐
     robot.head  ──► │intent slots│ ──atomic load, once per tick──►  read
                     │ twist│head │
                     └────────────┘
                     ArcSwap, stamped

     robot.init  ──► ┌────────────┐
     robot.relax ──► │power req.  │ ──taken once per tick──────────►  bring-up
     skills      ──► │skill flags │
                     └────────────┘
                     last request wins

     robot.health ◄── ┌──────────┐ ◄────────── publish ───────────  atomics
     safeToRestart    │ atomics  │              ticks, hz, missed,
                      └──────────┘              fallen, moving

     robot.state  ◄── ┌──────────┐ ◄─── send, only if subscribed ──  frame
                      │broadcast │
                      └──────────┘
                      bounded, drop-on-lag
```

**没有通道反向运行。** 健康是*发布*的，从不被询问，这让卡死的循环报告自己不健康而非挂起调用者。

技能标志是布尔而非队列：在一个 20 ms tick 内同一按钮第二次按没什么额外意义，而两个*不同*请求都该被看到——单个 last-writer 槽会丢掉一个。

### 4.2 Params

启动时读的 TOML 文件，**不监视**——热重载稍后。它住在 `releases/<ver>/` 外，所以在更新*和*回滚后存活，在 updater 自己的配置旁 `/etc/robot/robotd.toml`。

属于板子而非 release 是让手编的 policy 路径保持的原因：默认指向 `releases/<ver>/` 内，所以普通更新让 policy 与针对它训练的二进制在一起，删除 override 回到那个。文件可完全缺失——未 provisioned 的板子以构建内默认启动而非拒绝启动，那比不运行的 daemon 远容易远程诊断。推论，从惨痛教训学来：*未注释*的值在那块板上永远冻结而 release 继续前进，这就是一个机群最终以 kP 120 站立而 release 默认说 160 的方式。

约十个值，不是 142：控制率、增益、动作 scale、低通 alpha、最大速度、deadman 超时、policy 路径和更新门控阈值。runtime 中的标志爆炸主要是变体、死技能和死传感器，全都没了。

一个开关比其余更粗。`policy.mode` —— `walk` 或 `roller` —— 选择加载哪些 policy *和*调优默认，所以每个未设字段按模式解析，把机器人移到轮式上是一行加重启。它是预设，不是变体：轮式行是原型的，基于 alpha 默认重定基。

### 4.3 手柄是客户端

`padd` 读 `gilrs` 并通过 `robotd` 的 socket 发意图。它自己的 crate，所以手柄栈不进 `robotctl`——那个必须在坏机器人上工作的工具。

一次 socket 跳，几十微秒。它换来的：app、SDK 和任何远程客户端用的输入路径是开发者每天演练的，所以它不能悄悄腐烂。开发时，`ssh -L /tmp/robotd.sock:/run/robotd.sock` 给出笔记本上的 pad、板上的机器人，无需代码。

在机器人上它是 `padd.service`，从启动运行并驱动任何连接的 pad——无 pad 时安全，因为它什么都不发且 deadman 托住机器人。它保持**非特权**，这是"手柄是客户端"的关键部分：它的 `input` 和 `robot` 组成员身份是它的全部。配对 pad 因此属于 `configd`（`architecture.md` §1），不在这里——绑定设备需要 root 和 BlueZ，持有任一个的 `padd` 不再演练 app 将用的 API。

### 4.4 里程计，基于循环已采的样本

`odometry::Odometry::alpha()` 每 tick 从循环刚读的关节位置和 IMU 四元数步进一次，其估计作为 `odom: { position, yaw }` 发到状态流。`robotctl monitor` 在 3D 视图下把它画成路径图。

**基于接触**，从原型 runtime 移植：一个脚底角是地面锚，躯干的世界位姿通过 `kinematics` crate 的模型前向运动学从它推出，当另一个角降到它之下时锚移动——所以一步从不导致估计跳跃。航向是 IMU 的积分偏航，所以世界帧是机器人启动时看的地方。没有磁力计也没有东西纠正漂移；这是相对运动，每个消费者必须那样对待它。

它耗循环每 tick 两次链评估且无额外总线流量，这就是它以循环自身速率而非原型独立的 100 Hz 运行的原因。那个代价是 §7"无里程计"决定被反转而非重辩的原因：反对从不是算术，是没有东西读答案。

### 4.5 循环还驱动什么，以及为什么它们都没有设计页

`robotd` 在 slice 2 之后长出四个既非控制也非安全的子系统。它们共享一个形态：每个挂在 tick 上，没有一个可以阻塞它，且没有一个能通过循环已仲裁的意图之外到达总线。

| in the process | what it is | where the reasoning lives |
|---|---|---|
| `sound.rs` | the voice at play time — one `aplay` child, and a new sound kills the old one, because the codec's PCM is exclusive | the module header |
| `theremin.rs` | depth from `tofd` at 15 Hz → a note, a mouth opening, and a line of state, sampled by the 50 Hz loop and never waited on | the module header |
| `chorale.rs` | several ducks singing one piece: the lowest id conducts, the conductor owns the seating, `btd` carries the beacons and does no thinking | the module header |
| `pet-detect/` | a ~20 KB CNN over a 40-band log-mel window from the onboard mic, in its own worker | the crate header |

加上 `soc.rs`，它从 `sysfs` 读板子自己的热区——不在 `RobotIo` 后，因为它必须在电机总线不行时仍能应答。

**它们都没有设计页，那是规则在起作用而非缺口。** 当第二个读者否则必须从代码推导其契约时服务才值得有一个（`../README.md`）。这些各恰好有一个实现和一个消费者，且它们的决策足够局部，住在它们约束东西旁边的模块头部里。它们需要的反而是*可操作*，那是速查表的事：[the voice]、[the chorale]、[the theremin]。

[the voice]: ../robot/cheatsheet.md#the-voice
[the chorale]: ../robot/cheatsheet.md#the-duck-chorale
[the theremin]: ../robot/cheatsheet.md#play-the-duck-the-tof-theremin

## 5. 它为什么存在，以及它从哪来

### 5.1 目标，不是"一个好的 `robotd`"

想要两件事，第二个是重排工作的那个：

1. **在控制核心上快速迭代。**
2. **真正测试 updater。**

更新引擎已完成且从未在硬件上跑过。其最重要的路径因此未经验证：`on_apply` 里的 `systemctl restart` 从未遇到真实 systemd，30 秒健康门控超时是承认的猜测，且——最糟的——**自动回滚只有在 `robot.health` 有意义时才有意义。** 它曾意味着"控制循环 tick 了一次"，所以迄今测试的每个回滚都是针对占位符测试的。

这就是为什么第一个增量不会走。它存在是为了在真实板上成为诚实的健康信号（§3.4）。

### 5.2 `robotd` 取代什么

`robotd` 取代 runtime，但不是一步——runtime 做五个可分离的工作，只有一个是 `robotd` 的：

| runtime job | destination | when |
|---|---|---|
| control loop, policies, motors, IMU | `robotd` | done |
| gamepad | an intent client (`padd`) | done |
| camera, ball/laser/pet detection, JPEG | `mediad` | M5 |
| web hub, PWA, brain command socket | `mediad` / the app | M5+ |
| maploc — mapping, MCL, planning | unowned | — |

所以两者并行跑一阵。它们不能*同时*跑——一条串行总线，一个拥有者（§1.1）——所以一块板子跑一个或另一个，systemd unit 用 `Conflicts=` 说明。

远程/app 层不在这里设计。它是 `reachy_mini` 架构——WebRTC 做媒体、DataChannel 上 JSON-RPC 2.0 做控制——移植到 Rust，超出本文范围。它对 `robotd` 的一个要求在 §3.5。

### 5.3 两个 slice，以及"完成"意味着什么

**Slice 1 —— 保持姿态。** Tick、总线、模型、`RobotIo` 和诚实健康。什么都没计算：`held_pose` 是启动时采纳的常量。那是要点——它以真实速率把真实负载放到总线上，所以循环时序和健康是诚实的，且故意坏掉的 release 落地时什么都不倒下。你可以整天在实验台上锤 install/rollback/断电周期。完成条件：`robotd` 保持姿态一小时无总线错误；`robotctl update apply` 安装 release、重启它、通过门控、机器人不动；构建为不健康的 release 被自动回滚；更新中途断电通过启动计数器恢复。

**Slice 2 —— 行走和站立。** 观测、ONNX policy、安全层、意图和手柄客户端。完成条件：它在板上走，通过意图 API 驱动；用 `robotctl` 应用的更新干净重启它且门控通过；且 `--unhealthy` 仍回滚。

此后的一切——技能链、bring-up 状态机、轮式预设、limp-fall——在那个形态之上到来而非改变它。

### 5.4 不回归是验收标准

测量已存在：`bench_dynamixel_bus` 报告 50 和 100 Hz 下的达到速率、抖动、读取时间、总线时间、利用率、错误和 IMU 样本新鲜度。把今天的数字记为基线；`robotd` 必须匹配它们。这刻意不是 RT 工程项目——无 `SCHED_FIFO`、无亲和、无 `mlockall`。循环今天是可靠的，工作是在周围代码变简单时保持它那样。

## 6. 测试

带脚本化样本的 `FakeIo`，无硬件：

- 错过截止时间时健康变 false；
- 启动采纳当前姿态且从不命令运动；
- safety 拒绝非有限目标并钳制超过执行器范围的目标，且跌倒既不抢占 policy 也不抢占调用者的增益；
- limp-fall 预测器在跌倒时触发而在踏地或静态倾斜时不触发，且其姿态斜坡终止于站立姿态；
- deadman 在意图停止时把速度归零；
- **黄金观测向量** —— 从 mjlab 导出并提交的 `(inputs, expected 61-float array)` 对。观测中错误的索引不会大声失败；它产生一个会跌倒的看似合理的机器人。依赖来自 `microduck_brain` 的导出（§9.2）。

每个测试的注释说明它存在是为了防止哪种失败，按仓库约定。

## 7. 已记录的决策

| | |
|---|---|
| `duck-control` as a workspace crate | boundary enforced by the compiler, no second repo |
| bus layer written fresh, constants borrowed | thin code, but the tuned numbers are not re-derived |
| the IMU in the motors' `sync_read` | it is a device on the same bus; no IMU abstraction |
| Rust consts for the model | one robot exists |
| params file, not watched | establishes the file and its location; the watcher is later |
| policy path in params, default = release dir | updates carry the policy; devs override it |
| adopt current pose on start | an update must not move a standing robot |
| bring-up as a state machine, not a flag | `set_torque` is a transaction per joint |
| the fall verdict reports, it does not gate | what to do about a fall is a control decision (§2.4) |
| ~~no odometry~~ — reversed | `monitor`'s path map reads it, and it is one `kinematics` pass on a sample the loop already took (§4.4) |
| the priority chain keeps the runtime's shape | the skills were tuned against its quirks |
| gamepad as its own crate | keeps `gilrs` out of the recovery CLI |
| sim after slice 2 | hardware is the validation path; `FakeIo` covers laptop development |

## 8. 刻意推迟

MuJoCo 后端和 `RemoteIo` 协议 · 技能抽象 · policy bundle 清单和 `model_api` 门控 · `look`/`pose`/`do` 意图 · 注视 IK · 热重载 params 和配置存储 · 热限制 · 速率限制 · 逐设备 IMU 校准。

**里程计已离开此列表。** 它被推迟因为没东西读它；`robotctl monitor` 的路径图现在读了。§4.4。

## 9. 开放

1. **Radxa 上的控制率。** 50 Hz 继承自 Pi Zero 2W。现在有板子可测量——`bench_dynamixel_bus` 和循环自己的五分钟摘要报告相同数字。
2. **黄金向量**值得从 `microduck_brain` 获得——作为对训练环境的回归检查而非它们曾要成为的真相来源。不是任何发货东西的前置条件。
3. **逐关节限制不存在。** Safety 钳制到*执行器的*行程，抓 `NaN`、坏动作 scale 和垃圾张量——它不会阻止关节被驱动到机械上不明智的地方。真正的限制在 alpha MJCF（31 KB）里，不在此 vendored。一个看起来像解剖学但不是的限制会暗示没人有的保护。
4. **alpha MJCF 住在哪**如果想要 sim/real 一致性测试——31 KB XML、19 MB 网格，目前在 runtime 的 `scripts/alpha_assets/`。
5. **C 依赖的站立代价。** `gilrs` 在 Linux 上无条件拉 `libudev-sys`，所以 CI 和板子交叉构建安装它。同样的花费在下一个必须到达板子的 C 依赖上重演，所以在那条路径上偏好纯 Rust crate。*macOS 上未验证：*交叉构建需要 aarch64 sysroot，Mac 提供不了，所以 `cargo board --bins` 在那本地失败——用 `-p updater -p robotd -p robotctl` 构建发货集，或在 Linux 上构建。
