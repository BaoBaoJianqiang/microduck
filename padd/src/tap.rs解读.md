# `tap.rs` 解读

## 概述

padd 的原始输入 tap：同一个手柄的第二个读取者，在自己的 socket 上。

### 为什么 padd 长出了一个 socket

> One question about a gamepad cannot be answered from anywhere else in this system. `padd` polls the last known stick value and sends it at a steady 50 Hz, so a radio that has stopped delivering reports still produces perfectly fresh intents: `robotd` sees a live driver, the deadman never fires, and the robot keeps walking on a command nobody is still giving.

一个关于手柄的问题不能从系统中任何其他地方回答。padd 轮询最后已知摇杆值并以稳定 50Hz 发送，所以停止传送报告的无线电仍然产生完美新鲜的意图：robotd 看到活跃驾驶员，deadman 从不触发，机器人在没有人仍在给的命令上继续行走。

每个下游界面——`robot.state`、监控器的 `requested` 列、journal——都显示健康机器人，因为从它们那边看它是健康的。

**证据住在 padd 下面一层**：事件流本身，其中从未到达的报告在节奏中留下空洞。所以这个原样交出该流而非总结它，让调查者做算术。

### 为什么第二次读取设备而非转发 gilrs 给的

> `Gilrs::next_event` is not the raw stream and cannot be made into one. It applies three filters by default — `axis_dpad_to_button`, `Jitter`, `deadzone` — which rewrite values, drop small movements, and swallow whole events; `gilrs-core` turns `SYN_DROPPED` into an internal resync flag that never reaches a consumer; and neither `SYN_REPORT` nor `MSC_SCAN` survives the trip.

`Gilrs::next_event` 不是原始流，也不能变成原始流。它默认应用三个过滤器——`axis_dpad_to_button`、`Jitter`、`deadzone`——重写值、丢弃小移动、吞掉整个事件；`gilrs-core` 把 `SYN_DROPPED` 变成内部重同步标志，永远不到达消费者；`SYN_REPORT` 和 `MSC_SCAN` 都不能存活。

**打开节点两次不花代价也不拿走什么**：evdev 读取者得到自己的队列，所以这不能让 gilrs 饿死事件。节点来自 gilrs 自己（`gilrs::LinuxGamepadExt::devpath`），这是唯一确定这正在监视实际驱动的手柄的方式——一个 Xbox 控制器注册多个输入设备，`/proc/bus/input/devices` 中的第一个是从不发送任何东西的媒体键键盘。

### 无人观看时花什么代价

> Nothing. The device is not opened until a subscriber connects, and it is closed again after the first report following the last one leaving — the same bargain `robotd` strikes by only assembling a `robot.state` frame when someone is subscribed. A pad at rest is silent, so a parked reader is not a wakeup either.

什么都不花。设备直到订阅者连接才打开，最后一个订阅者离开后的第一个报告后再次关闭——和 robotd 只在有人订阅时组装 `robot.state` 帧的交易相同。静止的手柄是安静的，所以停放的读取者也不是唤醒。

---

## 关键常量

```rust
const QUEUE: usize = 256;
```

订阅者可以落后的帧数，超过则开始丢帧。忙碌手柄两秒。慷慨，因为队列的代价是内存，丢帧的代价是这个存在来做的测量中的空洞——但有界，因为替代方案是慢客户端变成机器人上的无界内存。

```rust
const GROUP: &str = "robot";
```

可以读取 tap 的组，匹配 robotd 自己的 socket。谁可以观看 `robot.state` 就可以观看驾驶它的手柄，没有其他人从这个存在中获得任何东西。

```rust
const SOCKET_MODE: u32 = 0o660;
```

socket 模式。组决定谁可以请求。

```rust
const REOPEN_AFTER: Duration = Duration::from_millis(250);
```

流结束后再次打开节点的等待时间。限制自旋。两种流结束方式让开始它的状态不变——节点根本不能打开（没有 `input` 组，每次尝试相同地失败），或打开时设备已经消失——没有这个读取者会以内核能拒绝它的最快速度重新打开、失败、重新打开。四分之一秒：每秒四次尝试不算什么，真正回来的手柄被捡起时也没人注意到延迟。

---

## 数据结构

```rust
pub struct Tap {
    shared: Arc<Shared>,
}

struct Shared {
    state: Mutex<State>,
    wake: Condvar,
}

struct State {
    wanted: Option<PathBuf>,
    subscribers: Vec<Subscriber>,
    attached: Option<Arc<proto::PadReport>>,
}

struct Subscriber {
    reports: SyncSender<Arc<proto::PadReport>>,
    dropped: Arc<AtomicU64>,
}
```

- **`Tap`**：主循环持有的句柄
- **`Shared`**：accept 循环、每个订阅者的写入者、读取者之间共享
- **`State`**：要读取的节点路径、订阅者列表、当前打开设备的 Attached 报告（新订阅者到达时立即告知）
- **`Subscriber`**：报告通道 + 丢帧计数器（其写入者将其盖到下一帧中）

---

## Tap::serve

```rust
pub fn serve(socket: &Path) -> std::io::Result<Self> {
    // 创建目录（RuntimeDirectory=padd 已在板子上创建，手动运行时试一下）
    // 删除陈旧 socket
    // 绑定 UnixListener
    // 设置权限 0o660
    // 交给 robot 组（非致命，失败时 warn）
    // 启动 accept 线程和 read 线程
}
```

**仅在 socket 不能创建时失败**。调用者预期在没有 tap 的情况下继续驾驶：padd 的工作是让机器人可驾驶，拒绝作为可选的调试设施会成为这个文件阻止机器人的方式。

### 组分配

```rust
fn give_to_group(socket: &Path, group: &str) -> std::io::Result<()> {
    let entry = unsafe { libc::getgrnam(name.as_ptr()) };
    let gid = unsafe { (*entry).gr_gid };
    unsafe { libc::chown(path.as_ptr(), u32::MAX, gid) }
}
```

> `robotd` gets the same effect from `Group=robot` in its unit, because a socket inherits its creator's primary group. `padd` cannot copy that: its primary group is its own, and it reaches the robot group as a supplementary one — which is enough for this, since POSIX lets the owner of a file give it to any group the owner belongs to.

robotd 从 unit 的 `Group=robot` 得到相同效果，因为 socket 继承创建者的主组。padd 不能复制：它的主组是自己的，它通过补充组到达 robot 组——这就够了，因为 POSIX 允许文件所有者将文件交给所有者所属的任何组。

---

## Tap::watch / idle

```rust
pub fn watch(&self, pad: &gilrs::Gamepad<'_>) {
    self.want(Some(pad.devpath().to_path_buf()));
}

pub fn idle(&self) {
    self.want(None);
}

fn want(&self, node: Option<PathBuf>) {
    let mut state = self.shared.lock();
    if state.wanted == node { return; }
    state.wanted = node;
    drop(state);
    self.shared.wake.notify_all();
}
```

**每 tick 安全调用**：在扰动任何东西之前先比较，因为替代方案是每秒 50 次拆除并重建读取者。

---

## Shared 的关键方法

### wait_for_work

```rust
fn wait_for_work(&self) -> PathBuf {
    let mut state = self.lock();
    loop {
        if let Some(node) = state.wanted.clone() && !state.subscribers.is_empty() {
            return node;
        }
        state = self.wake.wait(state)...;
    }
}
```

等到既有手柄要读又有人要为它读。**两个条件都满足才工作**：无手柄或无订阅者时读取者睡眠。

### done_with

```rust
fn done_with(&self, node: &Path) -> Option<&'static str> {
    let state = self.lock();
    if state.subscribers.is_empty() {
        return Some("nobody is watching any more");
    }
    if state.wanted.as_deref() != Some(node) {
        return Some("the pad changed");
    }
    None
}
```

读取者是否应该放开打开的节点？两种结束条件：无人观看、或手柄变了。

### subscribe

```rust
fn subscribe(&self) -> (Receiver<Arc<proto::PadReport>>, Arc<AtomicU64>) {
    let (reports, rx) = sync_channel(QUEUE);
    let mut state = self.lock();
    if let Some(attached) = state.attached.clone() {
        reports.try_send(attached);
    }
    let dropped = Arc::new(AtomicU64::new(0));
    state.subscribers.push(Subscriber { reports, dropped: Arc::clone(&dropped) });
    drop(state);
    self.wake.notify_all();
    (rx, dropped)
}
```

新订阅者被播种当前设备信息——立即知道它在看什么，而非等手柄拔插后才发现。

### send（从不阻塞）

```rust
fn send(&mut self, report: &Arc<proto::PadReport>) {
    self.subscribers.retain(|subscriber| {
        match subscriber.reports.try_send(Arc::clone(report)) {
            Ok(()) => true,
            Err(TrySendError::Full(_)) => {
                subscriber.dropped.fetch_add(1, Ordering::Relaxed);
                true
            }
            Err(TrySendError::Disconnected(_)) => false,
        }
    });
}
```

**从不阻塞**。读取线程是唯一能测量手柄节奏的东西，订阅者让它停转会损坏它要求的测量——停转后的每帧都会带一个与无线电无关的空洞。

---

## 读取线程

### read（主循环）

```rust
fn read(shared: &Arc<Shared>) {
    loop {
        let node = shared.wait_for_work();
        let why = stream(shared, &node);
        shared.detach(why);
        thread::sleep(REOPEN_AFTER);
    }
}
```

打开想要的节点、流式读取、结束后回去等待。永远。

### stream（读取一个设备）

```rust
fn stream(shared: &Arc<Shared>, node: &Path) -> String {
    let mut device = RawDevice::open(node)?;
    shared.attach(describe(&device, node));

    let mut seq = 0u64;
    let mut previous_us: Option<u64> = None;
    let mut events: Vec<proto::PadEvent> = Vec::new();
    let mut resyncing = false;
    let mut after_drop = false;

    loop {
        let batch = device.fetch_events()?;
        for event in batch {
            if syn == Some(SynchronizationCode::SYN_DROPPED) {
                events.clear();
                resyncing = true;
                after_drop = true;
                continue;
            }
            events.push(proto::PadEvent { kind, code, value, name: event_name(...) });
            if syn != Some(SynchronizationCode::SYN_REPORT) { continue; }

            let report = std::mem::take(&mut events);
            if resyncing { resyncing = false; continue; }

            let at_us = micros(event.timestamp());
            seq += 1;
            shared.frame(proto::PadFrame {
                seq, at_us,
                since_us: previous_us.map(|p| at_us as i64 - p as i64),
                events: report,
                after_drop,
                socket_dropped: 0,
            });
            previous_us = Some(at_us);
            after_drop = false;
        }
        if let Some(why) = shared.done_with(node) { return why.to_owned(); }
    }
}
```

#### SYN_DROPPED 处理

> The kernel's contract after `SYN_DROPPED`: everything up to the next `SYN_REPORT` is a half-report and must be thrown away, because the events that completed it are already gone.

内核在 `SYN_DROPPED` 后的契约：到下一个 `SYN_REPORT` 的所有内容都是半报告，必须丢弃，因为完成它的事件已经消失了。

**不是无线电的问题**——这个读取者落后了，内核清空了队列，这使得周围的空洞不可测量——所以下一个完整报告如此说明（`after_drop: true`）。

#### SYN_REPORT 边界

一个报告在其 `SYN_REPORT` 结束，`SYN_REPORT` 被保留：这是流的副本，因为是簿记而移除的事件是某人必须信任的事件。

---

## describe（设备描述）

```rust
fn describe(device: &RawDevice, node: &Path) -> proto::PadInputDevice {
    let id = device.input_id();
    let axes = device.get_absinfo()...;
    let held = device.get_key_state()...;
    let buttons = device.supported_keys()...;
    proto::PadInputDevice { name, node, unique, bus, vendor, product, axes, buttons }
}
```

一个帧中的数字必须对照来读的所有设备信息。**当前按住的键从内核查询而非假设清零**，所以订阅者在扳机按下时附着不会被告知手柄静止。

---

## event_name（事件命名）

```rust
fn event_name(kind: u16, code: u16) -> String {
    let named = match EventType(kind) {
        EventType::SYNCHRONIZATION => format!("{:?}", SynchronizationCode(code)),
        EventType::KEY => format!("{:?}", KeyCode(code)),
        EventType::ABSOLUTE => format!("{:?}", AbsoluteAxisCode(code)),
        ...
    };
    if named.is_empty() || named.contains(' ') {
        format!("{kind}:{code}")
    } else {
        named
    }
}
```

evdev 的 `Debug` 打印常量，对没有常量的代码打印 `unknown key: 42`。那段文字不能到达线上：有未映射轴的手柄正是有人在读这个流来理解的手柄，`3:42` 他们可以查字典，胜过他们无法 grep 的句子。名字中有空格就是未知形式。

---

## 订阅者线程

```rust
fn subscriber(stream: UnixStream, shared: &Arc<Shared>) {
    // 读取请求行
    // 验证是 PadInput 方法（其他方法返回 METHOD_NOT_FOUND）
    // 先回答（在订阅之前），所以回复不能在通知之后到达
    // 订阅
    // 循环：接收报告 → 将丢帧数盖到下一帧 → 写入 socket
}
```

### 丢帧盖印

```rust
let stamped = match (&*report, dropped.load(Ordering::Relaxed)) {
    (proto::PadReport::Frame(frame), missed) if missed > 0 => {
        dropped.store(0, Ordering::Relaxed);
        Some(proto::PadReport::Frame(proto::PadFrame {
            socket_dropped: missed,
            ..frame.clone()
        }))
    }
    _ => None,
};
```

> Whatever this subscriber missed goes on the next frame it does get, where it belongs: beside the gap it explains. Left on the counter until then, so it cannot be lost to an `Attached` or a `Detached` that carries nowhere to put it.

订阅者错过的一切放在它确实得到的下一帧上，在它属于的地方：在它解释的空洞旁边。在此之前留在计数器上，所以它不会丢失到没有地方放它的 `Attached` 或 `Detached` 上。

---

## 测试

### a_subscriber_arriving_mid_stream_is_told_the_device

流中途到达的订阅者被告知它在看什么。没有这个它会看到没有范围可读的值和没有设备名，直到手柄拔插——这是没有人调试链路时想被告诉做的事。

### a_slow_subscriber_is_dropped_from_rather_than_blocking_the_reader

慢订阅者丢帧并被告知丢了多少，在下一帧上。读取者从不阻塞：能让它停转的客户端会损坏它要求的节奏。

### a_departed_subscriber_is_forgotten

socket 关闭的订阅者被遗忘，所以打开关闭 50 次的监控器不会留下 50 个发送者让读取者序列化。

### the_device_is_only_held_while_a_pad_and_a_watcher_both_exist

设备只在手柄和观看者都存在时被持有。机器人上无人观看时，每个报告的唤醒是 robotd 也拒绝为 `robot.state` 付的代价。

### every_code_gets_a_name_or_a_number

每个代码得到内核表中的名字，没有名字的变成数字——不是 evdev 的 `unknown key: 42` 文字，那无法 grep 且会落在 JSON 行中间。

### a_frame_survives_the_wire

tap 的线上类型是协议的，所以它构建的帧是 `robotctl` 解析的帧。

---

## 与其他文件的关系

- **`main.rs`**：padd 主程序，调用 `Tap::serve()`、`Tap::watch()`、`Tap::idle()`
- **`duck_ipc_proto`**：协议类型（PadReport、PadFrame、PadEvent、PadInputDevice 等）
- **`evdev`**：Linux 输入子系统，提供 RawDevice、事件读取
- **`gilrs`**：提供 `devpath()` 获取正确的设备节点
- **`robotctl monitor`**：消费 tap 流的客户端
- **`scripts/pad-link-test.sh`**：也读取同一节点的测试脚本

---

## 关键踩坑点总结

1. **gillrs 不是原始流**：它应用三个过滤器（axis_dpad_to_button、Jitter、deadzone），吞掉 SYN_DROPPED/SYN_REPORT/MSC_SCAN。必须第二次打开 evdev 节点读原始流。

2. **Xbox 控制器注册多个输入设备**：`/proc/bus/input/devices` 中的第一个可能是媒体键键盘。节点必须来自 `gilrs::LinuxGamepadExt::devpath()`，而非自己找。

3. **SYN_DROPPED 不是无线电问题**：是读取者落后了，内核清空了队列。必须清空事件缓冲区、标记 `after_drop`、丢弃下一个半报告。

4. **读取者从不阻塞**：慢订阅者丢帧并计数，不能让它停转读取线程——停转后每帧的空洞都是客户端自己的错，不是无线电的。

5. **设备只在手柄+观看者都存在时打开**：无人观看时不打开节点，不花 IO 也不花唤醒。

6. **新订阅者被播种 Attached 报告**：立即知道设备信息（名称、轴范围、按钮），不用等手柄拔插。

7. **丢帧数盖到下一帧的 `socket_dropped` 字段**：不单独发消息，在下一个实际数据帧中说明错过了多少——这样空洞和解释在一起。

8. **`input` 组权限**：读取 evdev 节点需要 `input` 组，padd 的 unit 授予它，手动运行的 padd 通常没有。失败时记录"cannot open"并继续。

9. **事件命名用内核常量名而非 evdev Debug 文字**：`ABS_X`/`BTN_SOUTH`/`SYN_REPORT` 可 grep，`unknown key: 42` 不可。

10. ** poisoning lock 不导致 tap 死亡**：`lock().unwrap_or_else(PoisonError::into_inner)`——锁中毒时取出数据继续，因为数据是订阅者列表和路径，panic 不能将其半写为无效形状。
#（注：内容由AI生成）
