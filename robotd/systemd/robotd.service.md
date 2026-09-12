# robotd.service 文件解析

## 文件位置

`d:\microduck\robotd\systemd\robotd.service`

## 核心设计决策

### 1. 两个必须保留的性质

本单元是 `on_apply = { action = "restart", units = ["robotd"] }` 重启的对象，更新健康门等待其就绪。由此推出两条铁律：

1. **与 updaterd 无任何方向依赖**：updaterd 是恢复路径，必须在 robotd 不能运行时运行；robotd 也不能等 updater 才开始控机。
2. **socket 必须对 `robot` 组可读**：updaterd（及后续 btd、SDK）由此访问。

### 2. 不依赖网络

机器人必须在无任何连接时也能站立、保持关节、回答健康。只有 mediad 的远程网关需要网络。

## 配置项分析

### `[Unit]`

| 项 | 值 | 说明 |
|---|---|---|
| `Description` | Robot control daemon | |
| `Documentation` | `file:///opt/robot/daemon/current/docs/architecture.md` | |
| `After` | `local-fs.target` | 仅等本地文件系统，不等网络 |

### `[Service]`

| 项 | 值 | 说明 |
|---|---|---|
| `Type` | `exec` | |
| `ExecStart` | `.../robotd --socket /run/robotd.sock` | 不写 `--params`，使参数文件可选（缺省=内置默认，不拒绝启动） |
| `User` | `root` | 暂需 root（电机控制要 i2c/spi/gpio 字符设备），M4 后收紧 |
| `Group` | `robot` | **负载配置**：socket 继承进程主组，0660 才对 `robot` 组生效 |
| `Restart` | `always` | 干净退出也重启——失控即无控制 |
| `RestartSec` | `2s` | 短，因健康门在等；非零防 spin |
| `NoNewPrivileges` / `ProtectKernelTunables` / `RestrictSUIDSGID` | yes | 适度硬化，**不**用 `ProtectSystem=strict`/`PrivateDevices`（要碰硬件） |
| `Environment` | `RUST_LOG=info` | info 安全：循环每 5 分钟一行摘要，不每 tick 打（50Hz 下每天 ~430 万行会挤掉关键日志） |
| `RuntimeDirectory` | `robotd` | `/run/robotd/identity.json` 由 systemd 创建，属本单元 User，单元停止时自动删除 |

### 故意缺失的参数

- **`--unhealthy` / `--busy`**：仅用于台架测试更新回滚路径，手动添加，绝不随发布发货（`--unhealthy` 会回退每个更新）。

## 关键摘要

`robotd.service` 的核心约束：无 updater 依赖（恢复路径独立）、`Group=robot` 使 0660 socket 对 robot 组可读、不依赖网络、`Restart=always`（失控即无控制）、`RuntimeDirectory` 让 identity.json 随单元生死。参数文件不写在 ExecStart 中以保持可选。`--unhealthy/--busy` 仅台架用，不发货。
