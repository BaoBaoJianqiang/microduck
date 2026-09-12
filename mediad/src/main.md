# main.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\main.rs`

## 核心设计决策

`mediad` 守护进程入口。在本进程内运行信令服务器，向任何连接者流化视频，并给每个对等端一个 `control` 数据通道作为通往机器人 API 的管道。设计见 `docs/design/remote-webrtc.md`。

### 它不做什么

1. **不认证。** 能到达信令端口的任何人都能驱动机器人、看摄像头。这是决策而非遗漏：配对 PIN 是共享的 `000000`，门禁只会给每次连接加一步却证明不了什么。让机器人从 LAN 外可达的桥在会话到达前两侧都认证。
2. **不在恢复路径上。** `mediad` 起不来，机器人仍能走、仍能更新、仍可通过蓝牙到达。所以它可以依赖发布资产里的插件和设备节点的组，而 `updaterd` 不行。

## 命令行参数

| 参数 | 默认 | 含义 |
|---|---|---|
| `--host` | `0.0.0.0` | 信令服务器绑定地址（全接口，LAN 对等端才能直达） |
| `--port` | 8443 | 信令端口（`webrtcsink` 默认） |
| `--web-port` | 8080 | 控制台端口（与信令端口分离，因 `webrtcsink` 只接受 host/port） |
| `--config` | `/etc/robot/robotd.toml` | 参数文件（可缺省，缺省则用内置默认） |
| `--camera-device` | `/dev/video0` | 采集节点 |
| `--exposure` | 600 | 起始曝光行数（约 19 µs/行） |
| `--analogue-gain` | 1024 | 起始模拟增益（256=1x） |
| `--rotate` | 90 | 摄像头安装偏离直立的顺时针角度（**不再旋转像素**，告诉消费者） |
| `--no-auto-exposure` | false | 不测光，保持起始曝光 |
| `--flip-in-pipeline` | false | 也在管线内旋转（破坏编码器零拷贝路径，默认关） |

## 启动流程

1. `tracing_subscriber` 初始化。
2. `duck_ipc_proto::log_startup_identity!("mediad")`——任何可能失败的操作之前，让启动失败的 journal 也报告哪个构建失败。
3. 创建 tokio runtime。
4. 校验 `--rotate`（错误角度在开摄像头前拒绝）。
5. 加载 `[media]`/`[detect]` 配置。
6. 从同角度构造 `duck_detect::Turn`（检测器把转向折叠进已有的重采样）。
7. `rotation`：`--flip-in-pipeline` 时用 `mount`，否则 `None`。
8. **先起控制台**（管线失败时页面能说"管线起不来"，且控制台不依赖 GStreamer；绑定失败只 warn 不退出）。
9. **`producer::learn`**（在管线前，因 `webrtcsink` 的 `meta` 在元素构建时设置；configd 不应答只 warn）。
10. 构建 `Source`（Camera 或 Test）与 `Settings`。
11. `pipeline::start` 启动（失败则退出并指明哪步）。
12. 真实摄像头且非 `--no-auto-exposure` 时 `exposure::spawn`。
13. `detect` 有模型时 `detect::spawn_first`（失败只 warn，不影响摄像头/控制台/控制通道）。
14. 为每个对等端（`channels.recv`）：建 `Pool`、推 `video_notification`（礼貌性，可能被丢）、订阅检测广播（`Lagged` 跳过旧的）、`session::run`。

## 关键摘要

- 不认证、不在恢复路径。
- 启动顺序：控制台 → producer → 管线 → 曝光 → 检测器 → 对等端循环。
- 控制台绑定失败、检测器启动失败都是 warning，不丢视频/控制。
- 旋转默认交给消费者（CSS 变换 / 检测器重采样），不在管线内做，因代价是 145% 核 + 22 fps。
- 非 Linux 平台 `main` 是 stub（"mediad runs on the robot; this host is not Linux"）。
