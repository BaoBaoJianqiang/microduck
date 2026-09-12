# tofd.service 文件解析

## 文件位置

`d:\microduck\tof\systemd\tofd.service`

## 核心设计决策

`tofd` 是头部 ToF 传感器守护进程，刻意独立于 `robotd`（architecture.md §1）：启动上传约 90 KB 固件耗时数秒、与音频编解码器共用 I²C 总线、大多数鸭子未装该传感器——这些都不属于拥有电机的进程。

## [Unit]

| 字段 | 值 | 说明 |
|---|---|---|
| `Description` | Robot head ToF sensor daemon | |
| `Documentation` | architecture.md | |
| `After` | local-fs.target | |

注：i2c3 总线随设备树 overlay 到达，unit 层无需等待——守护进程用退避重试总线，也覆盖之后才插入的传感器。

## [Service]

| 字段 | 值 | 说明 |
|---|---|---|
| `Type` | exec | |
| `ExecStart` | `/opt/robot/daemon/current/bin/tofd` | |
| `User`/`Group` | tofd/tofd | 非特权运行 |
| `SupplementaryGroups` | i2c, robot | i2c：`/dev/i2c-*` 为 root:i2c 0660，这是本守护进程全部特权访问；robot：可把 depth socket 交给可看机器人的组 |
| `Restart` | always | |
| `RestartSec` | 5s | |

### 硬化

与 `padd` 相同的硬化，**但移除会夺走总线的部分**：

- `PrivateDevices=` **故意不设**：它会用不含 i2c 节点的最小 /dev 替换，失败会读成"未安装传感器"
- `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `ProtectKernelTunables/Modules`, `ProtectControlGroups`, `RestrictSUIDSGID`, `RestrictAddressFamilies=AF_UNIX`, `RestrictNamespaces`, `LockPersonality`, `MemoryDenyWriteExecute`, `SystemCallFilter=@system-service`, `SystemCallErrorNumber=EPERM`, `CapabilityBoundingSet=`（空）

### 其他

- `Environment=RUST_LOG=info`
- `StandardOutput/Error=journal`
- `RuntimeDirectory=tofd`（承载 socket 与 identity.json）

## [Install]

`WantedBy=multi-user.target`

## 关键摘要

tofd.service 以非特权 `tofd` 用户运行，靠 `i2c` 组访问总线、`robot` 组发布 socket；硬化全套但**保留 /dev 完整**（否则 i2c 节点消失会被误判为无传感器）；无 unit 依赖，靠守护进程自身退避处理总线与传感器的到达时机。
