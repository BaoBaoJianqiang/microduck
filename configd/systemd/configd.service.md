# configd.service 文件解析

## 1. 文件定位

- **路径**：`configd/systemd/configd.service`
- **角色**：`configd` 的 systemd 单元文件（安装到 `/etc/systemd/system/configd.service`）。文件把「为什么存在这个服务」翻译成三条必须保持的运行时属性，并配了一套收紧到近乎只读的沙箱。

## 2. 文件头：必须保持的三条属性

源自 `docs/architecture.md` §3.1：

1. **与 robotd 双向都无依赖**：机器人死的时候配置必须可达——出故障时客户端最需要的就是配网，这是它不属于 robotd 的全部理由。
2. **与 btd 也无依赖**：BLE 只是若干前门之一；btd 是这个 socket 的客户端，SDK 或 robotctl 不应被迫绕蓝牙。
3. **socket 必须可被 `robot` 组读取**：这是 btd、robotctl 以及日后 mediad 到达它的方式。

## 3. `[Unit]` 段

| 指令 | 值 | 说明 |
| --- | --- | --- |
| `Description` | `Robot config service (wifi, identity)` | 单元描述 |
| `Documentation` | `file:///opt/robot/daemon/current/docs/architecture.md` | 文档指向 |
| `After` | `dbus.service NetworkManager.service`（另含 `local-fs.target`） | 启动排序 |
| `Wants` | `NetworkManager.service` | **用 Wants 而非 Requires**：NM 缺失时 configd 仍启动并报 `net.state=unavailable`——这是可诊断的答案，并提示板子仍在 netplan 上 |
| `After=local-fs.target` | — | 刻意**不**加 `network-online` 依赖：本服务就是用来*获得*网络的 |

## 4. `[Service]` 段

### 4.1 启动与身份

- `Type=exec`。
- `ExecStart=/opt/robot/daemon/current/bin/configd --allow-user btd`：
  - **`--allow-user btd` 是承重配置而非整洁习惯**：没有它，所有经蓝牙来的变更调用都会被拒绝，因为 `SO_PEERCRED` 只报告对端的*主 gid*，而 btd 的主 gid 是它自己的。btd.service 中的 `SupplementaryGroups=robot` 只能让 btd 通过 socket 的 0660 模式、再无更多权限——这恰是正确的两层切分（§2.2）；但配网正是 BLE 存在的目的，故在此显式点名 btd。
  - **按名不按 uid**：systemd-sysusers 动态分配号，一块板上对的数字到下一块就错。
  - 特别说明**没有**整体授予 `robot` 组：能与 configd 说话和能重配机器人始终分离。
- `User=root`、`Group=robot`。root 是本文件中唯一值得专门论证而非默认的设置：
  - configd 恰好需要两项特权：NM 的 D-Bus API（改连接需 polkit 或 root）与 logind 的 Reboot（同理）。镜像中**没有 polkit**，无会话非 root 调用者会被 systemd 拒绝；选择是 root，或装一个 JS 策略引擎只为授权两个调用。
  - 信任边界仍在正确位置：btd——解析无线电范围内任何人字节的进程——是非特权的；configd 只见来自带凭证本地 socket 的有类型 JSON。**让解析器非特权比让分发器非特权更重要。**
  - 若将来因别的理由引入 polkit，这里应降为专用用户加两条 polkit 规则；在此之前由下面的沙箱限制它。与需要裸设备访问的 robotd 不同，configd 不碰硬件，可以被真正锁死。
- `Restart=always`、`RestartSec=2s`。

### 4.2 沙箱（「狭窄的 root」，每项均刻意）

| 指令 | 作用 |
| --- | --- |
| `NoNewPrivileges=yes` | 禁止提权 |
| `ProtectSystem=strict` | 整个文件系统只读，`ReadWritePaths` 除外 |
| `ProtectHome=yes` | 隔离 `/home` 等 |
| `ReadWritePaths=/var/lib/robot/config /run` | **唯一可写路径**：状态目录持有机器人名字，必须扛过更新与回滚（故在 `/var/lib` 而非发布目录）；`/run` 用于运行时 |
| `PrivateTmp=yes` | 私有 /tmp |
| `PrivateDevices=yes` | 无设备节点访问 |
| `ProtectKernelTunables=yes` | 内核可调参数只读 |
| `ProtectKernelModules=yes` | 不能加载内核模块 |
| `ProtectControlGroups=yes` | cgroup 只读 |
| `RestrictSUIDSGID=yes` | 禁止 setuid/setgid 语义 |
| `RestrictAddressFamilies=AF_UNIX` | **只剩 Unix 域套接字**：没有 AF_NETLINK——configd 经 D-Bus（unix socket）问 NM，自己从不碰网络接口 |
| `RestrictNamespaces=yes` | 禁止新建命名空间 |
| `LockPersonality=yes` | 锁定执行域 |
| `MemoryDenyWriteExecute=yes` | 内存不可同时写执行 |
| `CapabilityBoundingSet=` | **能力边界集为空**；特别地**不给 CAP_SYS_BOOT**——重启由 logind 执行，configd 只负责请求；给了该能力就能直接 `reboot(2)`，那正是本服务要避免的不干净关机 |
| `StateDirectory=robot/config` | 声明状态目录 |

### 4.3 日志与运行时目录

- `Environment=RUST_LOG=info`（stderr → journal；启动身份行用 `warn` 级，以便在长跑板子 `RUST_LOG=warn` 时仍保留）。
- `StandardOutput=journal`、`StandardError=journal`。
- **`net.connect` 会被记录，但口令不会**：`NetConnectParams` 有手写 `Debug` 脱敏（在 duck-ipc-proto，并有测试保持该行为）。
- `RuntimeDirectory=configd`：守护进程把自身身份发布到 `/run/configd/identity.json`，供 `robotctl health` 与 updaterd 启动检查读取。选 `RuntimeDirectory=` 而非自行授权路径，因为它是唯一同时满足「目录属主为本单元 User」且「在 ProtectSystem=strict 下仍可写」的机制；systemd 在单元停止时删除该目录，使停止的守护进程不可能留下一份声称自己在运行的身份。

## 5. `[Install]` 段

- `WantedBy=multi-user.target`：随多用户目标启用。

## 6. 要点小结

- 与 robotd、btd 均无依赖；对 NM 仅 `Wants` 且不加 `network-online`，保证坏机器人仍可配网。
- `--allow-user btd` 是让蓝牙配网变更获授权的承重参数；授权按名不按号，且不整体放开 robot 组。
- 因无 polkit 而用 root，但用 `ProtectSystem=strict`、仅 AF_UNIX、空能力集（无 CAP_SYS_BOOT）、单一可写状态目录等把 root 关进窄沙箱。
- 身份文件经 `RuntimeDirectory` 发布到 `/run/configd/identity.json`，单元停止即清除。
- 口令经协议类型的手写 Debug 保证不入 journal。
