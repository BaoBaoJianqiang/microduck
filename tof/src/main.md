# main.rs 文件解析

## 文件位置

`d:\microduck\tof\src\main.rs`

## 核心设计决策

1. **为何单独成守护进程。** `architecture.md` §1 刻意把感知从 `robotd` 拆出：感知崩溃绝不能带走电机控制。本传感器是具体案例——启动要通过 I²C 上传约 90 KB 固件耗时数秒、与音频编解码器共用总线、大多数鸭子根本没装该传感器。50 Hz 控制循环不读深度，放进去毫无收益。

2. **一写者拥有传感器（不变量 4）。** 所有消费者（今天是 `robotctl monitor`，将来是建图/避障）从同一处读同一帧，而非争抢总线。

3. **阻塞线程驱动传感器，异步服务器只服务 socket。** 传感器线程是普通 `std::thread`（非 tokio task），因为每次驱动调用都阻塞在 I²C 上且不可取消安全；socket 服务器用 tokio 当前线程运行时，不读硬件。慢消费者被丢弃而非拖慢传感器（`broadcast` 自带此行为，缺口在 `seq` 中可见）。

4. **指数退避同时服务两种失败。** "未安装"（大多数鸭子永久如此）与"总线瞬态"共用同一退避：瞬态 1 秒内恢复，永久则收敛到每分钟一次，避免打爆与音频编解码器共用的总线。

## 常量

| 常量 | 值 | 含义 |
|---|---|---|
| `SOCKET_MODE` | `0o660` | 与所有其他 socket 一致：组决定谁可读取 |
| `GROUP` | `"robot"` | 与 `robotd`/`padd` tap 同组：可看机器人者可看它所见 |
| `FRAME_BUFFER` | 32 | 慢消费者可落后的帧数上限（15 Hz 下约 2 秒），丢失在 `seq` 中可见 |
| `POLL` | 10 ms | 询问帧是否就绪的周期；1 字节寄存器读，几百微秒总线开销 |
| `RETRY_MIN` / `RETRY_MAX` | 1 s / 60 s | 指数退避上下限 |
| `BUS_CANDIDATES` | `/dev/i2c-pihat`, `/dev/i2c-3` | 自动探测的总线候选（udev 符号链接 + 旧 overlay 设备） |
| `ADDRESS_CANDIDATES` | `0x29`, `0x52` | 地址候选：0x29 是出厂默认，0x52 是原型时代因 IMU 占 0x29 而迁移遗留 |

## CLI 参数（`Args`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `--socket` | `proto::socket::TOF` | 服务 `tof.stream` 的 socket 路径 |
| `--bus` | 无 | I²C 总线设备；未设则尝试候选 |
| `--address` | 无 | 7 位地址；支持 `0x` 前缀十六进制 |
| `--hz` | 15 | 测距频率，8×8 下传感器最高 15 Hz，约占 400 kHz 总线 5% |
| `--fake` | false | 发布合成场景而非读硬件；同时含三类区带（有效/空旷/失败） |

## 主要函数

### `main()`

- 初始化 tracing
- 调用 `duck_ipc_proto::log_startup_identity!("tofd")` 发布启动身份到 `/run/tofd/identity.json`（`tofd` 曾是唯一不发布身份的守护进程，导致 `robotctl health` 报其沉默）
- 创建 `Status` 与 `broadcast` 通道
- 启动传感器线程（真实或 fake）
- 异步 `serve()`，完成后设置 shutdown 标志并 join 传感器线程

### `sensor_loop(...)`

外层循环：

1. `open_sensor()` 失败 → 记录一次（非每次尝试），更新 status，退避翻倍后重试
2. 成功 → 重置退避，日志"ranging"，status 置 up
3. 内层：`data_ready()` 轮询 → 读帧 → `seq` 自增 → 广播 `TofFrame`
4. 传感器中途掉线 → break 到外层重新 bring-up

`seq` 与 `at_us`（启动后微秒数）随帧发布，无订阅者时 send 结果忽略（多数时候无人观看是常态）。

### `fake_loop(...)`

合成场景：一列失败测量（status 4）、一列空旷（status 255）、其余为随时间扫动的距离墙。刻意非平坦渐变——三类区带渲染不同，后两类最易出错，`--fake` 首帧即全展示。

### `open_sensor(bus, address, hz) -> Result<Sensor>`

遍历总线×地址候选矩阵；总线不存在时直接跳过（说明 overlay 未加载）；找到即 `Sensor::open` + `start(hz)`，返回首个成功的。

### `serve(...)`

- 创建 socket 目录，清理旧 socket（systemd 停止时会删 runtime 目录，残留 socket 说明被外部杀死）
- 绑定后设 `0o660` 权限，尝试 `give_to_group` 把 socket 组设为 `robot`（失败仅警告，不致命）
- `tokio::select!` 监听 accept、SIGTERM、SIGINT
- 每个连接 spawn `subscriber`

### `subscriber(stream, status, frames)`

1. 读一行请求；解析失败回 `PARSE_ERROR` 并保持连接
2. 请求必须是 `Call::TofStream`；其他回 `METHOD_NOT_FOUND` 并保持连接（拼写错误被告知而非断开）
3. 成功后回复 `status.result()`，进入帧通知循环
4. 每帧以 notification 写入；`Lagged` 仅 debug 记录；`Closed`（sender 随进程消亡）关闭连接
5. 写入时 `BrokenPipe` 是 `robotctl monitor` 正常结束，非故障

### `give_to_group(socket, group)`

通过 `libc::getgrnam` 查组名→gid，`libc::chown(-1, gid)` 改组（owner 留空）。`getgrnam` 返回线程本地存储，注释声明本进程无其他并发调用。

### `parse_address(s)` / `sleep_unless_shutdown(...)`

地址解析支持 `0x` 十六进制与十进制；退避睡眠以 100 ms 切片，使退出不必等满一分钟。

## 单元测试

- `addresses_parse_in_both_bases` — `0x29`、`41` 正确解析；`0x1ff`（超宽）与非数字拒绝。
- `the_backoff_is_capped` — 退避翻倍至 `RETRY_MAX` 后封顶（无传感器的鸭子终生在此循环）。

## 关键摘要

main.rs 是 `tofd` 守护进程：一个阻塞线程独占传感器、一个异步 socket 服务器广播帧；单写者 + 慢消费者丢帧可见；指数退避处理永久未安装与瞬态故障；权限模型继承自 `padd`（i2c 组读总线，robot 组读 socket）；`--fake` 让离板开发能看到三类区带。
