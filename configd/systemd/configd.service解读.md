# `configd.service` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | systemd service unit 文件 |
| 行数 | 111 行（含注释） |
| 安装路径 | `/etc/systemd/system/configd.service` |
| 角色 | configd 守护进程的 systemd 单元定义——启动、沙箱、权限、日志 |
| 运行用户 | `root`（沙箱化的窄 root） |
| 运行组 | `robot` |

## 二、文件头注释：三个必须保留的属性

文件开头的注释明确了此服务存在的三个核心属性（来自 `docs/architecture.md §3.1`），任何修改都必须保留：

### 1. 不依赖 robotd（任何方向）

- 配置必须在机器人死亡时可达——配置 wifi 正是出问题时客户端需要的。
- 这就是它不是 robotd 一部分的全部原因。
- **如果 configd 依赖 robotd，robotd 崩溃时无法通过 BLE 配置 wifi，机器人变成砖头。**

### 2. 不依赖 btd

- BLE 是多个前门之一；btd 是此 socket 的客户端，SDK 或 robotctl 不必经过蓝牙。
- **configd 的 unix socket 可以被任何本地进程直接连接，不强制走 BLE。**

### 3. socket 必须对 `robot` 组可读

- 这是 btd、robotctl 和以后的 mediad 到达它的方式。
- socket 权限 `0660` + 属组 `robot` 是访问控制的第一层。

## 三、`[Unit]` 段

### 基本信息

| 指令 | 值 | 说明 |
|---|---|---|
| `Description` | `Robot config service (wifi, identity)` | 服务描述 |
| `Documentation` | `file:///opt/robot/daemon/current/docs/architecture.md` | 文档链接 |

### 依赖关系

```
After=dbus.service NetworkManager.service
Wants=NetworkManager.service
After=local-fs.target
```

| 指令 | 说明 |
|---|---|
| `After=dbus.service` | 在 D-Bus 之后启动——configd 通过 D-Bus 与 NetworkManager/logind 通信 |
| `After=NetworkManager.service` | 在 NetworkManager 之后启动 |
| `Wants=NetworkManager.service` | **弱依赖**而非 `Requires`：如果 NM 缺失，configd 仍启动并报告 `net.state=unavailable`，这是可诊断的答案，告诉你板仍在 netplan 上 |
| `After=local-fs.target` | 在本地文件系统之后启动 |

### 故意没有的依赖

- **没有 `network-online.target` 依赖**：此服务是你*获取*网络的方式，不能等网络在线才启动。
- **没有 `Requires=NetworkManager`**：弱依赖 `Wants` 允许 configd 在 NM 缺失时仍启动并报告不可用。

## 四、`[Service]` 段

### 启动类型与命令

| 指令 | 值 | 说明 |
|---|---|---|
| `Type` | `exec` | systemd 认为服务在 `ExecStart` 进程执行后即启动（比 `simple` 更严格，会等待 exec 系统调用完成） |
| `ExecStart` | `/opt/robot/daemon/current/bin/configd --allow-user btd` | 启动命令 |

### `--allow-user btd` 是承重的，不是整洁

- 没有它，每个通过蓝牙的变更调用都被拒绝，因为 `SO_PEERCRED` 只报告对等方的*主要* gid，而 btd 的主要 gid 是它自己的。
- `btd.service` 中的 `SupplementaryGroups=robot` 让 btd 通过 socket 的 `0660` 模式，但仅此而已——这是正确的两层拆分（§2.2）。
- 但配置正是 BLE 存在的目的，所以 btd 在这里被显式命名。

#### 按名称而非按 uid

- systemd-sysusers 动态分配 uid，在一块板上正确的数字在下一块板上是错的。
- 因此用 `--allow-user btd`（用户名）而非 `--allow-uid 123`。

#### 注意什么不在这里

- 没有任何东西授予整个 `robot` 组变更权限。
- **能对话 configd 和能重新配置机器人保持分离**——这是两层权限模型的核心。

### 运行身份

```
User=root
Group=robot
```

#### 为什么是 root——需要正当理由而非假设

configd 恰好需要两个特权：

1. **NetworkManager 的 D-Bus API**：修改连接需要 polkit 或 root。
2. **logind 的 Reboot**：同样需要 polkit 或 root。

此镜像上**没有 polkit**，没有它 systemd 对任何无会话的非 root 调用者都拒绝两者。所以选择是：root，或安装一个 JS 策略引擎来授权两个调用。

#### 信任边界仍在正确位置

- btd——解析来自无线电范围内任何人的字节的进程——是非特权的。
- configd 只看到通过对等凭据本地 socket 到达的类型化 JSON。
- **让解析器非特权比让调度器非特权更重要。**

#### 未来可能降级

- 如果 polkit 因其他原因到达，此服务应降级为专用用户 + 两个 polkit 规则。
- 在那之前，下面的沙箱是限制它的东西。
- 与 robotd 不同——robotd 需要原始设备访问——configd 可以被正确锁定，因为它不接触硬件。

### 重启策略

| 指令 | 值 | 说明 |
|---|---|---|
| `Restart` | `always` | 任何原因退出都重启（包括正常退出） |
| `RestartSec` | `2s` | 重启前等待 2 秒，避免崩溃循环打爆 CPU |

### 沙箱配置——窄 root

这是整个文件最关键的部分。每一行都是故意的：

| 指令 | 值 | 说明 |
|---|---|---|
| `NoNewPrivileges` | `yes` | 进程及其子进程不能获得新特权（setuid 等） |
| `ProtectSystem` | `strict` | 整个文件系统只读，除 `ReadWritePaths` 外 |
| `ProtectHome` | `yes` | `/home`、`/root`、`/run/user` 不可访问 |
| `ReadWritePaths` | `/var/lib/robot/config /run` | **唯一可写路径**：状态目录（保存机器人名称，必须存活更新和回滚，所以在 `/var/lib` 下而非 release 目录）+ `/run`（运行时目录） |
| `PrivateTmp` | `yes` | 私有 `/tmp`，与其他进程隔离 |
| `PrivateDevices` | `yes` | 私有 `/dev`，只有最基本的设备节点 |
| `ProtectKernelTunables` | `yes` | 内核参数（`/proc/sys`）只读 |
| `ProtectKernelModules` | `yes` | 不能加载/卸载内核模块 |
| `ProtectControlGroups` | `yes` | cgroup 层次结构只读 |
| `RestrictSUIDSGID` | `yes` | 不能创建 setuid/setgid 文件 |
| `RestrictAddressFamilies` | `AF_UNIX` | **只允许 unix socket**——`AF_NETLINK` 缺席：configd 通过 D-Bus（unix socket）询问 NetworkManager，从不自己接触网络接口 |
| `RestrictNamespaces` | `yes` | 不能创建新的命名空间 |
| `LockPersonality` | `yes` | 锁定执行域，不能通过 `personality()` 系统调用改变 |
| `MemoryDenyWriteExecute` | `yes` | 不能同时具有写和执行权限的内存页（防 JIT/ROP） |
| `CapabilityBoundingSet` | （空） | **不授予任何 capability**——包括 `CAP_SYS_BOOT`：因为 logind 执行重启，configd 只请求。授予此 capability 会让它直接调用 `reboot(2)`，这正是此服务存在要避免的不干净关机 |
| `StateDirectory` | `robot/config` | 自动创建 `/var/lib/robot/config`，属主为 `User=`，权限正确 |

#### 沙箱设计的核心逻辑

- **configd 不接触硬件**，所以可以比 robotd 更严格地锁定。
- 只允许 `AF_UNIX`：所有外部通信通过 D-Bus（unix socket）和 configd 自己的 unix socket。
- 不授予 `CAP_SYS_BOOT`：重启必须通过 logind（优雅关机），不能直接 `reboot(2)`（不干净关机）。
- 唯一可写路径是状态目录和 `/run`：配置必须持久化，但不能写入系统其他部分。

### 日志

| 指令 | 值 | 说明 |
|---|---|---|
| `Environment` | `RUST_LOG=info` | 日志级别 info，可通过环境变量覆盖 |
| `StandardOutput` | `journal` | stdout → journal |
| `StandardError` | `journal` | stderr → journal |

#### 日志设计要点

- 启动身份行在 `warn` 级别，这样它能在长期运行的板上 `RUST_LOG=warn` 时存活。
- `net.connect` 被记录，但其**密码不被记录**：`NetConnectParams` 有手写的 `Debug` 实现来脱敏密钥（`duck-ipc-proto`），并有测试保持这种方式。

### 运行时目录

```
RuntimeDirectory=configd
```

- 自动创建 `/run/configd`，属主为 `User=root`，在 `ProtectSystem=strict` 下仍可写。
- **这是唯一能同时做到以下两点的机制**：
  1. 创建由此单元的 `User=` 拥有的目录。
  2. 在 `ProtectSystem=strict` 下存活（否则整个文件系统只读）。
- systemd 在单元停止时也会删除它，所以停止的守护进程不能留下声称正在运行的身份文件。
- 守护进程在此发布 `/run/configd/identity.json`，被 `robotctl health` 和 updaterd 的启动检查读取。

## 五、`[Install]` 段

| 指令 | 值 | 说明 |
|---|---|---|
| `WantedBy` | `multi-user.target` | 多用户模式下启动（无图形界面的标准运行级别） |

## 六、与其他 systemd 单元的关系

| 单元 | 关系 | 说明 |
|---|---|---|
| `dbus.service` | `After` | D-Bus 是 configd 与 NetworkManager/logind 通信的通道 |
| `NetworkManager.service` | `After` + `Wants` | 弱依赖，NM 缺失时 configd 仍启动并报告不可用 |
| `btd.service` | 无依赖 | btd 是 configd 的客户端，不是依赖；configd 不依赖 btd |
| `robotd.service` | 无依赖 | 双向无依赖——配置必须在机器人死亡时可达 |
| `updaterd.service` | 无直接依赖 | 更新后可能重启 configd，但启动顺序无依赖 |
| `multi-user.target` | `WantedBy` | 安装目标 |

## 七、安全模型总结

### 两层权限模型

| 层级 | 机制 | 授予谁 | 能力 |
|---|---|---|---|
| 第一层：能对话 | socket `0660` + 组 `robot` | btd、robotctl、mediad 等 `robot` 组成员 | 只读查询（status、scan、info） |
| 第二层：能变更 | `--allow-user btd` | 仅 btd（按名称） | 变更操作（connect、pair、setName、reboot） |

### 沙箱限制

- **零 capability**：不授予任何 Linux capability。
- **仅 AF_UNIX**：不能直接操作网络接口。
- **只读文件系统**：除状态目录和 `/run` 外全部只读。
- **无硬件访问**：`PrivateDevices` 限制设备节点。
- **无新特权**：`NoNewPrivileges` 防止提权。

### 信任边界

- **btd（非特权）**：解析来自无线电的不可信字节。
- **configd（沙箱化 root）**：只看到来自对等凭据 socket 的类型化 JSON。
- **让解析器非特权比让调度器非特权更重要。**

## 八、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **配置独立于机器人** | 不依赖 robotd，机器人崩溃时仍可配置 wifi |
| **多前门** | 不依赖 btd，SDK/robotctl 可直接连接 socket |
| **弱依赖 NM** | `Wants` 而非 `Requires`，NM 缺失时报告不可用而非启动失败 |
| **不等网络** | 无 `network-online.target` 依赖，configd 是获取网络的方式 |
| **窄 root** | 沙箱化 root，零 capability，仅 AF_UNIX，只读文件系统 |
| **优雅重启** | 不授予 `CAP_SYS_BOOT`，必须通过 logind |
| **两层权限** | socket 组权限（能对话）+ `--allow-user`（能变更）分离 |
| **按名称授权** | `--allow-user btd` 而非 uid，适配动态分配 |
| **状态持久化** | `/var/lib/robot/config` 在 release 目录外，存活更新和回滚 |
| **身份文件自动清理** | `RuntimeDirectory` 在单元停止时删除，防止假身份残留 |
| **密码不进日志** | `NetConnectParams` 手写 Debug 脱敏，有测试保证 |
#（注：内容由AI生成）
