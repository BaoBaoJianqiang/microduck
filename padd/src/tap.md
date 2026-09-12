# tap.rs 文件解析

**文件位置**：`d:\microduck\padd\src\tap.rs`

## 核心设计决策

原始输入 tap：同一手柄的**第二个读者**，在自己的 socket 上提供只读事件流。

**为什么需要**：`padd` 轮询最后已知摇杆值并以 50Hz 稳定发送，所以一个已停止上报的无线电仍产出"新鲜"意图——robotd 看到活跃驾驶员，deadman 永不触发，机器人继续按无人发出的命令走。所有下游表面（robot.state、monitor 的 requested 列、journal）都显示健康机器人，因为从它们的角度确实如此。证据在 padd 之下一层：事件流本身，未到达的报告会在节奏中留下空洞。所以这里把流原样交出而非汇总，让调查者自己算数。

**为什么读两次设备而非转发 gilrs 给的**：`Gilrs::next_event` 不是原始流且无法变成原始流——它默认应用三个过滤器（`axis_dpad_to_button`、`Jitter`、`deadzone`）改写值/丢小动作/吞整个事件；`gilrs-core` 把 `SYN_DROPPED` 变成内部 resync 标志永不到达消费者；`SYN_REPORT` 与 `MSC_SCAN` 也过不去。这些都是追查不可靠链路的人需要看到的。打开节点两次不消耗任何东西：evdev 读者有自己的队列，不会饿死 gilrs。

**节点来自 gilrs 自身**（`LinuxGamepadExt::devpath`）——确保看的是正在驾驶的那个设备。一个 Xbox 手柄注册多个输入设备，`/proc/bus/input/devices` 里第一个是从不发任何东西的媒体键键盘。

## 常量

- `QUEUE = 256`：订阅者落后多少帧后开始丢帧。2 秒忙手柄。有界因慢客户端不能变成机器人上无界内存；丢帧计数并报告（`PadFrame::socket_dropped`）而非隐藏。
- `GROUP = "robot"`：可读 tap 的组，与 robotd socket 一致。
- `SOCKET_MODE = 0o660`。
- `REOPEN_AFTER = 250ms`：流结束后等多久再开节点。防自旋——两种流结束方式（无 input 组、设备已消失）会让读者无限快速 reopen-fail。

## 类型

- `Tap`：主循环持有的句柄，含 `Arc<Shared>`。
- `Shared`：accept 循环、每个订阅者 writer、reader 共享。`state: Mutex<State>` + `wake: Condvar`。
- `State`：`wanted: Option<PathBuf>`（要读的节点）、`subscribers: Vec<Subscriber>`、`attached: Option<Arc<PadReport>>`（当前设备的 Attached 报告，供中途加入的订阅者立即知道在看什么）。
- `Subscriber`：`reports: SyncSender` + `dropped: Arc<AtomicU64>`（该订阅者因队列满丢的帧数，写入下帧时盖章）。

## 函数

### `Tap::serve(socket)`
绑定 socket、设权限 0660、`give_to_group` 把组设为 robot（失败 warn 不致命）、启动 accept 与 read 线程。socket 创建失败才失败——调用方应在无 tap 时继续驾驶。

### `Tap::watch(pad)` / `Tap::idle()`
每 tick 安全调用（先比较再变更，避免 50Hz 拆建 reader）。从 gilrs 取 devpath。

### `Shared::lock()`
处理 poisoned lock——panic 不会让 tap 一起挂。

### `Shared::wait_for_work()`
等到有手柄**且**有订阅者才返回节点。

### `Shared::done_with(node)`
判断 reader 是否该放开节点：无订阅者→"nobody is watching any more"；wanted 变了→"the pad changed"。

### `Shared::subscribe()`
添加订阅者，若有 attached 则先塞一份。返回 receiver 与 dropped 计数器。

### `State::send(report)`
**永不阻塞**——reader 是唯一能测手柄节奏的东西，订阅者阻塞它会污染它要的测量。满则计数丢弃，断开则移除。

### `accept(listener, shared)`
永久接受订阅者，每订阅者一个线程（writer 不能阻塞 reader）。

### `subscriber(stream, shared)`
读一行请求，必须是 `PadInput`（否则 METHOD_NOT_FOUND 并说明此 socket 只服务 pad.input）。先应答再订阅（避免回复落在它之前的通知之后）。然后循环写报告，把 missed 数盖章到下一帧的 `socket_dropped`。

### `read(shared)`
永久循环：wait_for_work → stream → detach → sleep REOPEN_AFTER。

### `stream(shared, node)`
用 `RawDevice::open`（不 resync SYN_DROPPED）读。处理 `SYN_DROPPED`：清空当前事件集，标记 resyncing+after_drop，丢弃到下一个 SYN_REPORT 为止的半报告。SYN_REPORT 处出帧：seq、at_us、since_us（与上帧间隔）、events、after_drop、socket_dropped=0。

### `describe(device, node)`
设备描述：name、node、unique、bus、vendor、product、axes（min/max/flat/fuzz/value）、buttons（含当前按下状态——问内核而非假设清空）。

### `event_name(kind, code)`
用内核表给名（ABS_X、BTN_SOUTH 等）。无名则输出 `kind:code` 数字——evdev 的 `unknown key: 42` 散文无法 grep 且会落在 JSON 行中间。含空格视为无名形式。

### `micros(at)`
内核时间戳转微秒纪元。

### `give_to_group(socket, group)`
用 `libc::getgrnam` + `chown` 把 socket 组设为 robot，owner 不变（u32::MAX）。

## 单元测试

- 中途到达的订阅者被告知设备（无需拔插）。
- 慢订阅者被丢帧而非阻塞 reader，丢帧计数。
- 断开的订阅者被遗忘。
- 设备仅在手柄+观察者同时存在时持有。
- 每个 code 有名或数字，无散文。
- 帧经线路往返幸存（协议类型一致）。

## 关键摘要

tap 是为回答"无线电是否还在上报"这一系统别处无法回答的问题而存在。核心原则：永不阻塞 reader、原始流原样交出（含 SYN_DROPPED/MSC_SCAN/SYN_REPORT）、节点必须来自 gilrs、设备仅在有观察者时打开。
