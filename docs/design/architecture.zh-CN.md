# 架构

Microduck 机器人的整体架构。

本页面拥有守护进程拆分、IPC、状态所有权、机器人 API、远程访问和安全。[`updater-design.md`](updater-design.md) 拥有更新系统；[`robotd-design.md`](robotd-design.md) 拥有 `robotd` 控制循环；[`app-path-design.md`](app-path-design.md) 拥有手机应用路径。

## 设计原则

### 1. 一个进程做一件事

每个守护进程有一个明确的职责。没有"主守护进程"做所有事情。这使得每个守护进程都可以独立重启、独立测试，并且在崩溃时不会带走其他一切。

### 2. 状态所有权是明确的

每条状态有且只有一个所有者。其他守护进程通过 IPC 读取它，但不缓存它（或缓存时间很短）。这消除了"哪个版本是真的"的问题。

### 3. 机器人 API 是唯一的硬件接口

只有 `robotd` 接触电机总线、IMU 和传感器。其他守护进程通过 `robotd` 的 API 与硬件交互。这意味着电机控制逻辑在一个地方，并且可以在不接触硬件的情况下进行测试。

### 4. 故障隔离

一个守护进程崩溃不应该导致其他守护进程崩溃。`robotd` 死亡时，`btd`、`configd` 和 `updaterd` 继续服务——用户仍然可以通过 BLE 连接、配置 wifi、触发更新。

### 5. 版本交换而非补丁

更新是完整的版本交换，而不是增量补丁。这意味着任何版本都可以从任何其他版本安装，没有"先升级到 X 再升级到 Y"的链。回退是交换到 previous 或 golden，而不是撤销补丁。

## 守护进程

七个守护进程，每个有一个职责：

| 守护进程 | 职责 | 接触硬件？ | 拥有状态？ |
|---|---|---|---|
| `robotd` | 电机控制 + 机器人 API | 是（电机总线、IMU） | 是（电机状态、姿势） |
| `mediad` | 摄像头 + 音频 + WebRTC | 是（摄像头、麦克风） | 否 |
| `btd` | BLE 网关 | 是（蓝牙适配器） | 否 |
| `configd` | 配置 + wifi + 电源 | 否（通过 NM） | 是（配置存储） |
| `padd` | 板载音频守护进程 | 是（音频 codec） | 否 |
| `updaterd` | 更新守护进程 | 否 | 是（更新状态、发布目录） |
| `robot-boot-check` | 启动恢复检查 | 否 | 否（一次性） |

### `robotd` — 唯一接触机器人的进程

`robotd` 是唯一与电机总线、IMU 和传感器通信的进程。它运行一个 50Hz 的控制循环，读取 IMU，计算电机命令，并将其写入总线。

其他守护进程通过 unix socket API 与 `robotd` 通信：

- `mediad` 读取遥测（关节角度、电池）
- `btd` 转发手机命令
- `configd` 读取机器人名称和型号

`robotd` 的 API 是 JSON-RPC over unix socket。它是无状态的——每个请求都是独立的，没有会话。这使得 `robotd` 可以在不影响客户端的情况下重启（客户端只需重试）。

### `btd`、`padd`、`mediad` — owns nothing

这三个守护进程不拥有任何持久状态。它们是无状态的管道：

- **`btd`** 将 BLE GATT 特征转换为对其他守护进程的 IPC 调用。它不存储配置、不缓存状态、不记住任何东西。
- **`padd`** 管理音频 codec 的 ALSA 配置。它不存储音频数据、不记住音量设置（那些在 `configd` 中）。
- **`mediad`** 将摄像头和音频流式传输到 WebRTC 会话。它不存储视频、不记住客户端（会话结束时一切都消失）。

这意味着它们可以在任何时候被杀掉并重启，而不会丢失任何东西。这也意味着它们在 `robotd` 死亡时继续工作——`btd` 仍然可以接受 BLE 连接，`mediad` 仍然可以流式传输摄像头（如果摄像头不依赖 `robotd`）。

### `configd` — 配置所有者

`configd` 拥有所有持久配置：

- 机器人名称
- wifi 凭据
- PIN 码
- 音频音量
- 电源设置

它将配置存储在 `/var/lib/robot/config/` 下的 TOML 文件中。其他守护进程通过 IPC 读取配置，但不直接读取文件。

`configd` 还管理 wifi（通过 NetworkManager）和电源（通过 logind）。它不授予 `CAP_SYS_BOOT`——重启必须通过 logind 优雅关机。

### `updaterd` — 更新所有者

`updaterd` 拥有更新状态和发布目录。它管理：

- 发布目录（`/opt/robot/daemon/releases/`）
- `current` 符号链接
- `golden` 符号链接
- 更新日志
- 启动计数器

它是唯一移动 `current` 的进程。其他守护进程通过 IPC 触发更新，但不直接操作符号链接。

## IPC

守护进程之间通过 unix socket 通信，使用 JSON-RPC 2.0。

```
btd ──→ robotd（命令、遥测）
btd ──→ configd（配置读取、PIN 验证）
btd ──→ updaterd（更新触发、状态）
configd ──→ robotd（名称、型号）
mediad ──→ robotd（遥测）
updaterd ──→ robotd（健康检查）
```

### 为什么是 unix socket 而非 D-Bus

- **简单**：JSON-RPC over unix socket 比 D-Bus 更容易理解和调试
- **无依赖**：不需要 D-Bus 守护进程或库
- **性能**：unix socket 比 D-Bus 快（没有中间守护进程）
- **权限**：socket 权限（0660 + robot 组）比 D-Bus 策略更容易推理

### 为什么是 JSON-RPC 而非 gRPC

- **人类可读**：可以用 `socat` 或 `nc` 手动调试
- **无代码生成**：不需要 protoc 或生成的代码
- **灵活**：可以在不破坏客户端的情况下添加字段
- **足够快**：对于 50Hz 的控制循环，JSON 解析开销可以忽略不计

## 状态所有权

| 状态 | 所有者 | 存储位置 | 其他如何读取 |
|---|---|---|---|
| 电机状态、姿势 | `robotd` | 内存（运行时） | IPC |
| 配置 | `configd` | `/var/lib/robot/config/` | IPC |
| 更新状态 | `updaterd` | `/var/lib/robot/updater/` | IPC |
| 发布目录 | `updaterd` | `/opt/robot/daemon/` | 文件系统（只读） |
| BLE 连接状态 | `btd` | 内存（运行时） | 不暴露 |
| WebRTC 会话 | `mediad` | 内存（运行时） | 不暴露 |

关键规则：**如果两个守护进程需要相同的状态，其中一个是所有者，另一个通过 IPC 读取。** 没有共享内存，没有共享数据库，没有共享文件（除了发布目录，它是只写的 `updaterd` 和只读的其他）。

## 机器人 API

`robotd` 的 API 是 JSON-RPC 2.0 over unix socket。方法按命名空间组织：

| 命名空间 | 方法 | 用途 |
|---|---|---|
| `motor` | `set_speed`, `get_position`, `home` | 电机控制 |
| `posture` | `set`, `get`, `list` | 姿势预设 |
| `do` | `action`, `list` | 动作库 |
| `teleop` | `drive`, `stop` | 遥操作 |
| `telemetry` | `get`, `stream` | 遥测读取 |
| `system` | `info`, `health`, `reboot` | 系统信息 |

API 是无状态的——每个请求都是独立的。没有"开始会话"或"结束会话"。这使得 `robotd` 可以在不影响客户端的情况下重启。

### 健康检查

`updaterd` 在更新后调用 `system.health`。如果 `robotd` 报告不健康，`updaterd` 自动回滚到 previous 版本。

健康检查只看控制循环是否满足其截止线——它不看电机是否实际移动（那需要硬件）。这意味着健康检查可以在没有电机的情况下通过，但一个真正不健康的 `robotd`（崩溃循环、卡住的循环）会被捕获。

## 远程访问

两种远程访问方式：

### 1. BLE（通过 `btd`）

手机通过 BLE 连接到机器人。`btd` 暴露 GATT 特征，手机写入 JSON-RPC 命令，`btd` 将它们转发到相应的守护进程。

BLE 用于：
- 初始设置（wifi 配置、命名）
- 更新触发
- 基本控制（姿势、动作）

不用于：
- 视频流（BLE 带宽太低）
- 遥操作（延迟太高）

### 2. WebRTC（通过 `mediad`）

在 LAN 上，客户端通过 WebRTC 连接到机器人。`mediad` 运行信令服务器，协商 WebRTC 会话，并流式传输视频/音频。

WebRTC 用于：
- 视频流
- 音频流
- 遥操作（低延迟数据通道）
- 控制台（完整 API 访问）

## 安全

### 沙箱化 root

`configd` 以 root 运行，但被 systemd 沙箱化：

- `ProtectSystem=strict`
- `ProtectHome=true`
- `PrivateTmp=true`
- `NoNewPrivileges=true`

它需要 root 是因为：
- polkit 缺失（没有非 root 方式管理 NetworkManager 系统连接）
- 需要读取对等凭据（SO_PEERCRED）
- 需要写入 `/etc/` 下的配置

但它不授予 `CAP_SYS_BOOT`——重启必须通过 logind 优雅关机。

### 两层权限

`configd` 的 socket 有两层权限：

1. **对话权限**：socket 是 0660 + robot 组。任何在 robot 组中的进程都可以连接和发送命令。
2. **变更权限**：`--allow-user btd` 标志授予 `btd` 变更配置的权限。其他进程（如 `mediad`）可以读取但不能写入。

这意味着 `btd` 可以代表手机变更配置，但 `mediad` 不能。

### BLE 配对

BLE 配对使用 just-works 加密——没有 PIN 码配对（BLE 无法表达固定 PIN）。相反，PIN 检查在传输层：`btd` 在执行任何变更命令之前验证 `system.authenticate` PIN。

出厂 PIN 是 `000000`。字符串比较保留前导零——`000000` 不等于 `0`。

### 不依赖 `robotd`/`btd`

`configd` 不依赖 `robotd` 或 `btd`。配置必须在机器人死亡时可达——如果 `robotd` 崩溃，用户仍然需要能够配置 wifi 或触发更新。

`updaterd` 也不依赖 `robotd`。更新必须在机器人不健康时可达——这就是回滚的工作方式。

## 什么未被测试

- 守护进程崩溃时的 IPC 重连——客户端应该重试，但尚未在真实故障条件下测试
- `robotd` 重启时的 `btd` 连接——BLE 连接应该保持，但命令可能会失败
- 高负载下的 IPC 延迟——50Hz 控制循环 + 多个 IPC 客户端可能会导致延迟
- 沙箱逃逸——systemd 沙箱应该阻止 `configd` 写入未经授权的路径，但尚未审计
- BLE 范围内的恶意设备——just-works 配对意味着任何在范围内的设备都可以连接，但 PIN 检查应该阻止变更
#（注：内容由AI生成）
