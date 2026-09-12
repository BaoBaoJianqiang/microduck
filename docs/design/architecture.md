# 机器人 Daemon —— 总体架构

状态：草稿 · 日期：2026-07-22 · 负责人：pierre

排期与里程碑见 [`roadmap.md`](../project/roadmap.md)。

本文是 [`updater-design.md`](updater-design.md) 的姊妹篇，后者详细覆盖更新系统。本文覆盖服务划分、服务间如何通信、状态存放在哪，以及如何控制机器人——本地、从 app、以及远程。

范围说明：本文描述我们向**首个发布版本**进发的目标，而非当前原型（`microduck_runtime`，它是探索性的，将被重写）。v1 面向**单一、明确规格的硬件配置**。

## 整体形态

一块板上七个 daemon，通过 unix socket 通信。其中一个驱动机器人；另外三个存在的意义是让第一个坏掉时板子仍可达；其余是不拥有任何东西的传输和传感器。

```text
   gamepad          phone          you, on a laptop     a peer, anywhere    a GitHub release
      │ BLE/USB        │ BLE             │ ssh                 │ WebRTC              │ https
      ▼                ▼                 ▼                     ▼                     │
  ┌────────┐      ┌────────┐       ┌──────────┐          ┌──────────┐                │
  │  padd  │      │  btd   │       │ robotctl │          │  mediad  │                │
  └───┬────┘      └───┬────┘       └────┬─────┘          └────┬─────┘                │
      │               │  a subset of the same API             │                      │
      │  robot.*      │  robot.health · update.* · net.* · pad.* · system.*           │
      ▼               ▼                 ▼                     ▼                      │
  ┌──────────────────────────────────────────────────────────────────┐               │
  │   one unix socket per service · JSON-RPC 2.0, one object a line  │               │
  └────┬──────────────────────┬─────────────────────────┬────────────┘               │
       ▼                      ▼                         ▼                            │
  ┌───────────┐        ┌─────────────┐           ┌─────────────┐                     │
  │  robotd   │        │  configd    │           │  updaterd   │◄────────────────────┘
  │ robot.*   │        │ net.* pad.* │           │ update.*    │
  │ 50 Hz     │        │ system.*    │           │ verify      │
  │ loop      │        │ wifi, name, │           │ swap        │
  │ safety    │        │ pad bonding │           │ health gate │
  └─────┬─────┘        └──────┬──────┘           └──────┬──────┘
        │ Dynamixel           │ D-Bus                   │ systemctl restart,
        ▼                     ▼                         │ then robot.health
  15 servos + IMU      BlueZ · NetworkManager           ▼
  on one UART                                    /opt/robot/daemon/current

  ┌ publishes, answers nothing ───────────────────────────────────────┐
  │  tofd — the head's 8×8 depth matrix, on /run/tofd/tof.sock.       │
  │         mediad and robotd read it; it reads no one.               │
  └───────────────────────────────────────────────────────────────────┘
```

**`robotd` 是唯一接触机器人的东西。** 十五个舵机和 IMU 板共享一条串行总线，50 Hz 控制循环拥有它。客户端发送*意图*——"走这么快"、"看那边"、"站起来"——而 `robotd` 内部的安全层决定什么是实际可执行的。系统中没有其他东西能命令电机（[`robotd-design.md`](robotd-design.md)）。

**其中三个在 `robotd` 死掉时仍存活。** `configd`、`updaterd` 和 `btd` 对它没有 systemd 依赖、没有 ML 运行时、没有媒体栈，因为它们是恢复路径：控制循环无法启动的机器人正是某人需要重新配置、更新或回滚的机器人。这也是配置放在 `configd` 而非 `robotd` 的原因（§1.1）。`mediad` 和 `padd` 确实依赖它，这是允许的：没有摄像头和没有手柄的机器人仍然是你能更新的机器人。

**`btd`、`padd` 和 `mediad` 不拥有机器人的任何东西。** 它们是传输。`btd` 把 API 的一个子集从 BLE 转发到应答它的 socket；`padd` 读手柄并发送与 app 相同的意图；`mediad` 通过 WebRTC 数据通道承载相同的调用，只拥有流水线。三者都可以在不触碰机器人行为的情况下被替换，且三者每天都被使用，所以 app 将使用的 API 不会悄悄腐烂。`tofd` 是个例外：它拥有一个传感器，发布帧，不读任何东西（§1）。

**发布是交换的，不是打补丁的。** 构建作为一个完整目录落到 `/opt/robot/daemon/releases/<version>/` 下；`updaterd` 验证其签名、移动 `current` 符号链接、重启 units，然后询问 `robotd` 是否健康。如果不健康，它自己把旧 release 放回去。越过这一关的崩溃循环由启动计数器捕获（[`updater-design.md`](updater-design.md)）。

| service | owns | listens on | reaches out to |
|---|---|---|---|
| `robotd` | motor control, sensing, policies, safety, `robot.health` | `/run/robotd.sock` | the Dynamixel bus |
| `configd` | wifi, robot identity and name, pairing PIN, gamepad bonding, reboot | `/run/configd.sock` | BlueZ and NetworkManager over D-Bus |
| `updaterd` | releases: verify, install, swap, health-gate, roll back | `/run/updaterd.sock` | GitHub releases, `systemctl`, `robotd` |
| `btd` | nothing — BLE transport for a subset of the API | a BLE GATT service | `robotd`, `configd`, `updaterd` — not `padd` or `tofd`, whose streams a radio this narrow cannot carry |
| `padd` | nothing — gamepad transport; serves a raw input tap | `/run/padd/pad.sock` (`pad.input` only) | `/run/robotd.sock` |
| `mediad` | the camera and audio pipeline; nothing of the robot — WebRTC transport and the remote front door (§5.2) | TCP: the console on `:8080`, signalling on `:8443` — no unix socket of its own | `robotd`, `configd`, `updaterd` |
| `tofd` | the head's ToF sensor: an 8×8 depth matrix it publishes and nobody else reads | `/run/tofd/tof.sock` (`tof.stream`) | the HAT's I²C bus |
| `robotctl` | nothing — the CLI, and the tool that must work on a broken robot | — | every socket above |

状态存放在哪，以及什么在更新后存活：

| | |
|---|---|
| `/etc/robot/robotd.toml`, `updater.toml` | per-board configuration; the installer writes it once and never overwrites it. `robotd.toml` is read by `robotd` and — for `[media]` alone, what the camera streams — by `mediad`, so a change there restarts `mediad` rather than `robotd` |
| `/var/lib/robot/config/config.json` | robot name and pairing PIN — a file plus `flock`, owned by `configd` (§3.1) |
| NetworkManager profiles | wifi credentials; we never store them (§3) |
| `/opt/robot/daemon/releases/<ver>/` | binaries, policies and shipped defaults — replaced atomically |
| `/opt/robot/daemon/current` | the symlink that says which release is live |
| `/run/<service>/identity.json` | what each daemon is actually running, published at startup |

`releases/<ver>/` 之外的一切都在更新和回滚后存活。这就是全部规则，也是为什么逐板配置不随 release 发布。

一个变更如何端到端地到达机器人：

```text
  push a branch ──► CI builds and signs a release ──► robotctl update apply
                                                            │
                                                            ▼
                                          updaterd: verify signature, unpack,
                                          move `current`, restart the units
                                                            │
                                                            ▼
                                          health gate: ask robot.health
                                            ├─ healthy  ──► keep it
                                            └─ not      ──► put the old one back
```

本文其余部分是推理：服务划分（§1）、服务如何通信（§2）、谁拥有哪些状态（§3）、API 及其传输（§4）、远程访问（§5），以及安全权限位于何处（§6）。

## 1. 服务

`systemd` 是监督者：生命周期、崩溃重启、顺序、看门狗。

| Service | Owns | Notes |
|---|---|---|
| `robotd` | motor control, kinematics, odometry, gait policies, sensor loop, safety | RT-ish core; authoritative on anything that can hurt the robot. Odometry is a struct in the loop, not a service: its inputs are exactly the sample the loop already read |
| `mediad` | camera/mic, encode, perception, WebRTC + remote gateway | Heaviest service; also the remote API front door (§5.2) |
| `btd` | BLE GATT server | **Transport adapter only** — owns no state (§4.1). See [`app-path-design.md`](app-path-design.md) |
| `configd` | wifi, robot identity, power, gamepad pairing | Config must be reachable when `robotd` is dead (§3.1), and `btd` must own nothing (§4.1) — so it is neither's business but its own. Gamepad pairing is here rather than in `padd` because bonding a device needs root and BlueZ, and `padd` is deliberately an unprivileged client (§4.1) |
| `tofd` | the head ToF sensor: an 8×8 depth matrix on the HAT's I²C bus | Perception, so split from `robotd` for the reason below. Owns one sensor and publishes frames; reads nothing. A board with no sensor fitted runs it anyway and says so |
| `updaterd` | update engine | See `updater-design.md` |

把 `mediad` 从 `robotd` 拆出来是刻意的：媒体/感知崩溃不能带走电机控制。`tofd` 是同样规则应用于一个更小的传感器，细节说明了理由：把 VL53L5/8CX 带起来要通过 I²C 上传约 90 KB 固件耗时数秒，总线与音频编解码器共享，而大多数鸭子根本没装传感器——那种重试循环不属于拥有电机的进程。控制循环里没有任何东西读深度，所以把它移出来没有损失。它刻意**不**是 `mediad` 的一部分：深度是总线上的一个传感器，不是媒体流水线，而且在有摄像头可标注之前很久它就有用了。

消费者访问它的方式与访问手柄原始流相同——在拥有者 daemon 自己的 socket 上订阅（`tof.stream`），绝不通过 `robotd`。把一帧重投影到机器人自身坐标系意味着通过 `kinematics` crate 的头部 FK 将其与 `robot.state` 的关节状态结合；`tofd` 发布传感器的视图，不假装去算它算不出的几何。

### 1.1 不变量

1. **`btd`、`configd` 和 `updaterd` 在 `robotd` 死掉时仍存活。** 它们是恢复路径；它们必须在恰好有东西坏掉的情况下工作。对 `robotd` 无 systemd 依赖，所有 IPC 可选且有超时上限，最小依赖面（无 ML 运行时、无媒体栈）。详见 `updater-design.md` §4.1。

   `configd` 在这里有具体原因而非对称：配置 wifi 正是机器人坏掉时某人需要的，所以把配置放在 `robotd` 会让它在唯一重要的情况下不可达。
2. **`robotd` 对安全有最终决定权。** 任何远程或本地客户端都不能绕过跌倒检测、关节/温度限制或安全姿态逻辑。客户端发送*意图*；`robotd` 决定什么可执行。
3. **`robotd` 的控制循环从不阻塞在另一个服务上。** 所有跨服务读取都是 last-value-wins 缓存，绝不是同步 RPC（§2.4）。
4. **每段状态单一写入者。** 每个值恰好有一个拥有服务；其他所有人读或订阅。

## 2. 服务间通信

### 2.1 控制平面 vs 数据平面

两种需求不同的流量。把它们混为一谈是这里的经典错误。

| | Control plane | Data plane |
|---|---|---|
| Content | commands, config, status, perception events | video/audio frames |
| Size/rate | tens of bytes, ≤100 Hz | ~27 MB/s for 640×480 RGB @30 fps |
| Mechanism | unix socket RPC | **never crosses a socket** |

### 2.2 控制平面：unix socket 上的 JSON-RPC 2.0

- 每个服务拥有**一个 unix socket**。客户端直接连接。N=4 时没有理由用 broker——总线是另一个会失败的组件，且它违反不变量 (1)。
- **线格式：JSON-RPC 2.0，每行一个对象（NDJSON）。** 标准协议而非定制：标准请求/响应关联、标准错误对象、标准**通知**——这正是推送进度和事件流的正确形态。帧用 `tokio_util::codec::LinesCodec`；消息类型是普通 `serde` 结构体。
- **处处异步加超时，无异常。** 任何对端都可能已死。关闭或静默的 socket 是正常、预期的应答。
- 订阅是开放连接上的通知流。

**已衡量的替代方案**（唯一依赖数，ARM-Linux 目标）：

| Option | Deps | Why not |
|---|---|---|
| **JSON-RPC/NDJSON + tokio** | **30** | chosen |
| `jsonrpsee-types` only (our transport) | 36 | reasonable; declined — trades frozen-spec code for a `0.x` dependency |
| `varlink` | 24 | close in spirit; less familiar, little gained over JSON-RPC |
| `zbus` (D-Bus, p2p+blocking) | 66 | see below |
| `axum` over UDS | 66 | viable; see "HTTP/WS" below |
| `tarpc` | 71 | ergonomic Rust↔Rust, but not human-readable and server-push is awkward |
| `tonic` (gRPC) | 81 | `.proto` + codegen overhead for a handful of methods |
| `jsonrpsee-server` | 112 | **cannot serve a unix socket** — HTTP/WS transports only; `jsonrpsee-ipc` was never implemented |

依赖数记录供参考，非决定因素。

**为什么用 unix socket 而非 localhost HTTP/WS。** 功能上几乎相同；区别在访问控制和失败模式：

- **文件系统权限是免费的授权。** mode 0660 加专用组的 socket 只允许进程访问。TCP 端口对机器上每个进程和用户都可达，所以必须构建一个认证层才能回到同等水平。
- **`SO_PEERCRED`** 得到调用者的 uid/gid/pid，用于*审计日志*（"谁触发了这次回滚"是支持人员问的第一件事）和**强制执行**。两层，因为它们回答不同问题：
  - socket 的组（mode 0660）决定谁可以**和** daemon **对话**；
  - `allow_uids`/`allow_gids` 决定谁可以**改变机器人**——仅限变更调用。daemon 运行的 uid 总是被允许（它反正能替换 daemon）；其他所有人都需要被列出，未知对端被拒绝。
  只读调用刻意不加门控：支持必须能检查它无权更改的机器人。单凭组成员身份就说"可以替换固件"对一个面向 BLE 的服务是客户端的设备来说太粗了。
- **错误接口类 bug 不再存在。** 因笔误、配置或"让它能从我笔记本工作"的补丁而绑定 `0.0.0.0`，会把*固件更新控制*暴露给网络。在 unix socket 上这种错误无法表达。权重最大——不是今天的威胁模型，而是失败模式。
- 与计划中的 **SDK** 相关：如果第三方或用户代码将来在板上运行，localhost 端口对它开放；组拥有的 socket 不开放。

**为什么不用 HTTP/WS 作为控制平面**（与传输无关——`axum` 能很好地服务 UDS）：协议需要服务端→客户端推送进度。在 HTTP 上意味着调用用 POST *加上* WebSocket/SSE 做通知——两种机制，且 `curl` 无法消费流式那一半，所以可调试性收益只覆盖请求/响应。全在 WebSocket 上做恢复了一种机制但加了握手才能得到帧定界的 JSON，又失去了 `curl`。在持久 NDJSON 连接上，调用和通知是一种机制且无握手：**概念更少，这才是真正的目标。**

**将来的选项，如果需要 `curl` 能力做诊断：** 在同一个 unix socket 上加一个小型*只读* HTTP 端点（`GET /status`、`GET /log`），提供 `curl --unix-socket … http://x/status`，而不把控制操作移到 HTTP 上或复制流式路径。诊断和控制有不同需求；分开两者对两者都好。

**为什么不用 D-Bus：** BlueZ（通过 `bluer`）已经引入了它，所以它反正就在机器上，且 `zbus` 支持无总线点对点。但同样的消息类型也必须通过 **BLE 和 WebRTC/WebSocket** 传输（§4.1、§5.2），在那里普通 serde 结构体可用而 D-Bus 类型不行。一个定义、多种传输是目标，JSON 让这免费实现。我们只在 OS 要求的地方用 D-Bus（BlueZ、NetworkManager）。

**重新审视触发条件：** 如果 `btd` 最终为 BlueZ 深度投入 D-Bus，那么通过 `zbus` 也暴露更新接口对*`btd`*来说几乎免费，且能换来 `busctl` 自省用于调试。改动成本低——类型不变，只是帧定界移动。

### 2.3 等待处异步，计算处同步

异步（tokio）用在服务真正等待的地方：在长操作运行时服务 IPC、给对端查询或子进程设超时、取消进行中的工作。

CPU 密集和长时间文件系统工作保持**同步**，异步调用者把它交给 `spawn_blocking`。在 updater 中具体是：对 artifact 做 SHA-256、minisign 流验证、`zstd`+`tar` 解压、以及递归删除解压出的树。在 Pi 上这些运行数秒；留在异步 worker 上会使本应在更新期间继续应答 `status`/`subscribe` 的 IPC 任务停滞。

真正快的文件系统操作——符号链接 `rename`、fsync、小追加——直接调用。把这些分发到线程池得不偿失。

### 2.4 数据平面：特征，不是帧

`robotd` 不需要摄像头帧——它需要*派生特征*（"球在 (x,y)"、"检测到人"、"大声响"）。几十字节，10–30 Hz，通过 socket 微不足道。

**原则：把感知放在传感器旁边。** `mediad` 拥有摄像头、运行推理、发布特征。把帧运到 `robotd` 让它跑自己的视觉会浪费板子大部分内存带宽。

**在控制循环中：** `robotd` 订阅一次并读取本地缓存的*最新*快照——非阻塞，last-value-wins。停滞的 `mediad` 于是降级感知而非给电机控制加抖动。

**如果帧必须跨进程边界**（最后手段）：共享内存（shm/dmabuf 环形缓冲区），socket 只携带"帧 N 在偏移 X 就绪"。libcamera 提供 dmabuf，所以这可以零拷贝。优先特征而非帧，避免需要这个。

## 3. 状态归属

三个不同类别；更新/回滚含义见 `updater-design.md` §5.7。

| State | Owner | Mechanism |
|---|---|---|
| Wifi credentials | **NetworkManager** | We never store them. `configd` drives NM over D-Bus; NM persists profiles root-only and reconnects on its own. |
| Robot identity, user prefs, tunables | **config store** (§3.1) | File + `flock` + `rename(2)`, owned by `configd` |
| Calibration, learned state, generated per-device assets | owning service | Outside release dirs; survives update *and* rollback |
| Shipped defaults, binaries, policy bundles | update system | Under `releases/<ver>/`, swapped atomically |

让 NetworkManager 拥有 wifi 凭证代码更少、安全性更好、少一件要迁移的事。

**板子到货时没有 NetworkManager**，这一行原本假设它有。Armbian 的 headless 镜像跑 netplan + `systemd-networkd` + `wpa_supplicant`，而 netplan 是配置*生成器*：它没有扫描 API，`netplan apply` 报告"配置已应用"而非关联是否成功。这正是手机配置机器人最需要的两件事——"给我看网络"和"密码错了"——所以决定不变，`scripts/migrate-network.sh` 把板子一次性迁到 NM。推理以及在板上测得的内容见 [`app-path-design.md`](app-path-design.md) §2。

### 3.1 配置存储

一个普通文件加一个小共享 crate——**刻意不是服务**：

- `flock` 做写入序列化；写临时文件 + `rename(2)` 做原子性。
- `inotify` 做变更通知。
- 无单点故障，任何服务宕机时仍可读，updater 从不触碰它（`updater-design.md` §5.7）。

实现在 `configd/src/store.rs`，持有机器人名称和蓝牙配对 PIN。`inotify` 目前**还没**加，刻意如此：当*第二个*进程读该文件时它才有价值，而今天 `configd` 是唯一的一个。监视一个你是唯一写入者的文件是仪式。

配置**必须**在 `robotd` 死掉时可达——wifi 配置正是出问题时客户端需要的——所以它不能住在 `robotd`。

**配置是状态，不是动作。** "连接这个 wifi"、"重启"、"应用更新"、"选择模型"是动作，作为 RPC 分发到拥有服务。

## 4. 机器人 API

### 4.1 一个定义，多种传输

`btd` **不拥有任何东西**。BLE 是几个前门之一。如果配置或 provisioning 住在 `btd`，其他服务会依赖它，SDK 会荒谬地必须经过 BLE。

```
        ┌──────── one API definition (shared crate: types + operations)
        │
   ┌────┴─────┬────────────┬──────────────┬────────────────┐
  BLE       unix socket   WebSocket     WebRTC datachannel
 (btd)      robotctl,     server-side   telepresence,
  subset    on-robot SDK  agents/LLM    full fidelity
```

每种传输都是同一 API 上的薄适配器。BLE 暴露**子集**（provisioning、状态、更新触发/进度）——它太慢太受限，无法承载完整表面，负载从不经过它。

### 4.2 跨领域规则

- **逐传输授权。** BLE 意味着物理存在 + 配对；网络传输需要 token。同一 API，不同授权——从一开始就决定检查放在哪。
- **API 版本握手。** SDK 和 daemon 版本*会*不一致。一个整数，不匹配时用清晰消息拒绝（与 `model_api` 同方法）。
- **意图，不是电机写入。** 见 §6。

## 5. 远程访问

### 5.1 需求

所有机器人和媒体数据必须能通过 **WebRTC** 连接访问，以支持 (a) 远程临场和 (b) 观察并控制机器人的服务端程序（如 LLM）。后者必须*简单*。

### 5.2 WebRTC 会话

一个 PeerConnection 承载一切：

```
peer (browser / phone / server)
   ├── video track(s)   ── camera
   ├── audio track(s)   ── mic + speaker (two-way for telepresence)
   ├── datachannel "control"   reliable, ordered      → the robot API (§4)
   └── datachannel "teleop"    unreliable, unordered  → input + high-rate telemetry
```

两个数据通道，对应 §2.1：teleop 输入和遥测走**不可靠**（`maxRetransmits: 0`），因为重传一个 80 ms 前的摇杆命令比没用更糟——永远取最新的。

**`mediad` 拥有 PeerConnection。** 一个 PC 不能跨进程拆分（轨道和数据通道共享一个 DTLS/SCTP 关联），且它需要编码后的媒体，所以它和 `mediad` 住在一起，后者把 `control` 消息代理到拥有服务的 unix socket。`mediad` 因此是**远程网关**。

隔离代价可接受：没有媒体的远程临场会话毫无价值，所以共处不损失拆分能保留的任何东西。本地恢复通过 BLE / `robotctl` 保持独立（不变量 1）。

### 5.3 服务端代理：不要强迫它们走 WebRTC

对 LLM 驱动的控制器，WebRTC 是*更难*的路径。代理不想要 30 fps 的 H.264 轨道去解码——它想要每秒一两帧加一个状态 blob。先要求 ICE/DTLS/SDP 和解码流水线是糟糕的权衡。

| Consumer | Transport | Media |
|---|---|---|
| Telepresence (human) | WebRTC | tracks, low latency |
| Server-side agent / LLM | **WebSocket** | `get_frame` → JPEG on demand, or 1–2 fps push |
| On-robot SDK, `robotctl` | unix socket | snapshot API |
| App | BLE + WebRTC | as needed |

所有背后是同一 API。"在控制机器人的服务器上跑 LLM"变成：开一个 WebSocket、轮询一帧、发送意图——几十行代码，无媒体栈。这才是真正简单的原因。

还要注意 LLM 延迟（数百毫秒到秒）意味着代理是**高层**控制器："去厨房"、"看那个人"。反应式控制留在本地 `robotd`。无论传输如何，这都是正确的拆分。

### 5.4 基础设施现实检查

"机器人有自己的 wifi"**不等于**"从互联网可达"。远程 WebRTC 需要：

- 一条**信令**路径（SDP/ICE 交换），
- **STUN**，以及作为对称 NAT 中继回退的 **TURN**——有真实带宽成本的真实基础设施。

这与更新设计的"零后端"前提矛盾。**仅 LAN 远程临场完全避免它；互联网远程临场不能。** 开放问题（§9）。

### 5.5 实现说明

库选择取决于硬件编码：如果想要 V4L2 M2M 硬件 H.264，GStreamer `webrtcbin` 更务实；`webrtc-rs` 是纯 Rust，更易推理但要我们自己构建流水线。这大幅塑造 `mediad`——尽早决定。

远程临场驾驶的延迟预算：目标**<200 ms 端到端**。意味着低延迟编码器设置、无 B 帧、帧内刷新而非大关键帧。

## 6. 安全与权限

通过有损链路远程控制行走机器人。这些设计进去比事后改装便宜得多。

- **Deadman / 心跳。** 如果命令停止到达或 RTT 飙升超过阈值，`robotd` 自己停下机器人。不可谈判：网络会分区、LLM 推理中途停顿、笔记本会睡眠。
- **意图，不是电机写入。** 远程客户端发送速度、注视目标、"坐下"——从不发送原始关节命令。`robotd` 对跌倒检测、关节/温度限制和安全姿态保持最终决定权。一个困惑的代理绝不能命令机器人执行会撞墙的动作。
- **显式权限仲裁。** 物理控制器、app、远程对端和自主行为层都想要控制。定义优先级和交接，而非 last-writer-wins。本地/物理应能抢占远程。
- **会话限制。** v1：一次一个媒体会话，加 M 个仅控制客户端。多对端视频（simulcast、一次编码多路发送）推迟。

## 7. 隐私

这是别人家里的摄像头和麦克风。

- **显式同意**才能开始远程会话（每次会话，或用户可撤销的清晰持久选择）。
- **可见的板上指示器**在推流活动时亮起。
- DTLS-SRTP 让媒体端到端加密**即使经过 TURN 中继**——值得向客户明说。
- BLE 配置写入携带 wifi 凭证：该特征必须配对 + 加密。

## 8. 可观测性：日志与版本

跨领域的，因为别人家里的机器人不能靠接调试器调试。支持人员能要的东西必须已经在机器人上。

部署细节——journald drop-in、安装步骤、验证命令——见 [`../deploy/README.md`](../../deploy/README.md)。本节是每个服务必须满足的契约。

### 8.1 每个服务都记到 stderr

`tracing` → stderr → journald，级别通过 `RUST_LOG`（发布 unit 中为 `info`）。没有服务写自己的日志文件：一种机制、一种保留策略、一个查看地方。

**每个 daemon 写的第一行是它自己的身份**，级别 `warn`，以便在长时间运行的板上 `RUST_LOG=warn` 时仍存活：

```
WARN starting service="robotd" build=0.2.0 (rev a1b2c3d, built 2026-07-28T13:50:00Z)
     exe=/opt/robot/daemon/releases/0.2.0/bin/robotd pid=814
```

`exe` 有其价值：它说明进程实际从哪个 release 目录启动，这是"更新成功了"和"符号链接移动了但 systemd 仍在跑旧路径"之间的区别。

**日志量是保留决策，不是装饰。** `robotd` 的每 tick 心跳在 `debug`；在 `info` 它每五分钟记一条摘要，携带达到的 tick 率占目标的百分比。每 tick 一行在 `info` 会让一个空闲机器人每天产生约 8.6 万条，而在 journal 大小上限下，这些条目正是*挤掉*事故所需日志的东西。摘要还说明更多：一个跑在目标 60% 的循环是活的且通过健康检查，没有别的东西会显示它。

### 8.2 两份记录，刻意不同的持久性

| | where | survives power loss | capped by |
|---|---|---|---|
| service logs | journald | only if configured, see `deploy/README.md` | `SystemMaxUse` |
| **update history** | `/var/lib/robot/updater/update-log.jsonl` | **yes** | 200 entries |

更新历史刻意不在 journal 里。它住在引擎的 `state_dir` 下 `/var/lib`，每条追加时 `fsync`，重写是原子的（临时 + rename + 父目录 `fsync`）。所以"这台机器人装了什么，发生了什么"在 journal 是易失或被擦除的机器人上也能回答——这才是现实的支持场景，而非理想场景。

### 8.3 运行版本和已安装版本是不同问题

`updaterd` 不能在更新中途重启自己（`updater-design.md` §4.1），所以每次更新后的几秒内，运行中的二进制合法地落后于已安装 release。任何报告一个版本号的工具因此在那个窗口内是错的，且错的方向是让正常的机器人看起来坏掉。

几秒，不是"直到重启"：引擎在回复后调度自己和 `btd` 的 5 秒后重启，下一次 `updaterd` 启动检查那些是否落地并重启没落地的（`restart-order.md` §5）。

`robotctl version` 报告两者并点名不一致：

- `updaterd` 落后于已安装 release → 短暂预期，且是唯一不会自愈的不一致：后继者报告它而非重启自己，所以如果持续存在说明调度的重启没发生；
- `robotd` 落后 → *不*预期，因为它在 `on_apply` 的重启集中，所以重启没生效。

这些是不同诊断，不能共用一条消息。`--json` 为支持包提供相同内容，且该命令在 `updaterd` **宕机**时也工作——把那作为一行报告而非退出，因为那正是有人伸手用它的时候。

版本可从四个独立地方恢复，所以丢一个也能撑住：启动日志行；通过 IPC 的 `robotctl version`；每个二进制的 `--version`；以及每个 release 目录内的 `version.toml`（加上 `robotctl update list`，显示每个已安装 release 构建自哪个 revision）。

`revision` 从 `DUCK_REVISION` 编译进来——由 CI 设置，本地没有，那里二进制诚实地报告 `rev unknown, not a CI build`。编译时读取，运行时从不从 git 读：发布的机器人没有仓库。它比看起来更重要：一旦分支安装落地（路线图 M2），多个构建共享一个版本号，revision 是区分它们的唯一东西。

### 8.4 健康是一个问题，所以是一个命令

`robotctl health` 在一个应答中报告来自 `robotd` 的硬件和来自 `updaterd` 的软件。这不是便利："这台机器人怎么了"在*回答之后*才分为硬件和软件，而一小时前回滚了 release 的机器人与舵机没上电的机器人看起来完全一样，直到两半同时出现在屏幕上。拆开它会让调用者在知道哪一半有问题之前先选一半。

它**在机器人不健康或不可达时非零退出**，所以可以作为脚本的门控——任何构建在其上的东西依赖的契约。没有其他东西影响退出码：扁平电池、电机过热和被钉住的组件被报告，不被评判。release 绝不能因它落地的板子的状态而被回滚。

`--json` 为支持包携带相同内容。

## 9. 开放问题

1. **v1 做互联网可达远程临场，还是仅 LAN？** 大问题：零后端与运营信令 + TURN 的区别（§5.4）。
2. **SDK 是团队内部还是发布给终端用户？** 决定 API 兼容性承诺的强度（§4.2）。
3. **权限优先级**当 app、远程对端和自主行为不一致时——哪怕是粗糙的固定顺序，也要决定而非自然出现（§6）。
4. **感知放在 `mediad` 还是独立的 `perceptiond`？** 打包在一起让推理挨着摄像头且更简单；拆分意味着感知崩溃不会杀死视频流。取决于视觉变得多重。
5. **行为/大脑层**（驱力、情绪、习惯）：`robotd` 的一部分，还是独立服务和更新通道？无论哪种，其学习状态都是 `updater-design.md` §5.7 的材料。
6. **通过 BLE 撤销绑定。** 没有东西解除手机配对；`bluetoothctl untrust` 是手动逃生。需要 API 和关于谁可调用它的规则（[`app-path-design.md`](app-path-design.md) §5）。
7. **逐设备 provisioning 状态**——每机器人配对 PIN，目前没有别的。序列号曾是另一个主张者，不再需要槽位：它熔入 SoC，从 `/proc/device-tree/serial-number` 读取（`updater-design.md` §5.6，[`app-path-design.md`](app-path-design.md) §8.2）。PIN 不能共享身份，这曾是计划：身份在广播中发布，所以从它派生的任何东西都是公开的。秘密仍必须在制造时生成、记录和打印。

## 10. 构建顺序

`updaterd` **最先**构建，然后用于交付该架构其余部分的每次后续迭代。这把更新系统风险前置到失败仍免费的时候（无客户、没什么有价值的东西可破坏），且意味着更新路径在机器人发货前被演练数百次。

做这件事时要尊重的后果：

- `updaterd` 针对 `robotd` 的**接口**构建（健康探测、safe-to-restart），而非实现——初始用桩，这也使它可测试。
- 早期健康探测会很弱（"进程活着"）。自动回滚信心随 `robotd` 成熟而增长；首先被测试的是*机制*。
- **在早期开发全程保留手动恢复路径（SSH / 重刷）。** updater 既未经验证又快速变化，且它发货在它更新的 artifact 内部——别让它成为唯一的回去的路。
- 无法后装的 schema 字段（`min_supported`、`schema_version`、`model_api`）从第一个 release 起就在，即使未使用。
