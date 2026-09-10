# `lib.rs`（configd）解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust crate 根模块（lib.rs） |
| 角色 | `configd` crate 的入口，定义模块结构与顶层架构文档 |
| 定位 | 机器人的配置守护进程——以 root 运行的窄沙箱，拥有 wifi、蓝牙配对、机器人名称、重启等配置能力 |

## 二、crate 顶层文档

### configd 的定位

`configd` 是机器人的**配置守护进程**，核心职责：

1. **网络配置**：wifi 扫描、连接、忘记（通过 NetworkManager）。
2. **蓝牙配对**：游戏手柄配对/忘记（通过 BlueZ）。
3. **机器人身份**：名称、序列号、机器 ID（从 SoC 派生）。
4. **系统控制**：重启（通过 logind）。
5. **服务状态**：查询哪些守护进程在运行、运行的是哪个版本。

### 为什么以 root 运行

- configd 是唯一以 root 运行的守护进程。
- 原因：
  - logind 的 `Reboot` 是 polkit 门控的，此板上没有 polkit，非 root 无法调用。
  - NetworkManager 的系统连接修改需要 root。
  - BlueZ 的配对/信任操作需要 root 或 `bluetooth` 组成员。
- 但它是**窄的、沙箱化的 root**（见 `systemd/configd.service`），不是全能 root。

### 与 btd 的权限分层

- `btd` 非特权运行，解析来自无线电的不可信字节。
- `configd` 以 root 运行，但只看到来自对等凭据本地 socket 的类型化 JSON。
- 把解析器放在权限边界的安全侧，比加固分发器更重要。

## 三、模块布局

| 模块 | 角色 | 说明 |
|---|---|---|
| `identity` | 机器人身份 | 从 SoC 序列号派生名称和机器 ID |
| `store` | 配置存储 | 持久化键值对到文件（原子写入） |
| `net` | 网络 trait | `Net` trait + `FakeNet` 假实现，可在无无线电的笔记本上测试 |
| `nm` | NetworkManager 客户端 | `Net` trait 的真实实现，通过 D-Bus 操作 NetworkManager |
| `bluez` | BlueZ 适配器 | 蓝牙设备管理，实现 `Pads` trait 的真实部分 |
| `pad` | 游戏手柄 trait | `Pads` trait + `FakePads` 假实现 |
| `power` | 重启 | 通过 logind 优雅重启 |
| `units` | 服务状态 | 查询 systemd 单元状态 + 读取守护进程发布的身份文件 |

## 四、trait + 假实现模式

### 为什么用 trait

- 测试套件在没有无线电的笔记本上运行。
- 值得测试的逻辑是调度、授权和决策——不是 BlueZ 或 NetworkManager。
- 因此每个外部依赖都抽象为 trait：
  - `Net` trait（网络）
  - `Pads` trait（游戏手柄）
- 真实实现（`nm`、`bluez`）在 Linux 上使用。
- 假实现（`FakeNet`、`FakePads`）用于测试和 `--fake-*` 模式。

### 假实现的价值

- 使得整个 API 表面可从笔记本锻炼。
- 可以安排用真实硬件难以安排的失败场景（如两个手柄同时在配对模式）。
- `--fake-net` / `--fake-pads` 命令行参数启用假实现。

## 五、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `btd` | 通过 unix socket 转发 `net.*`、`pad.*`、`system.*` 调用到 configd |
| `updaterd` | `updater.toml` 的 `allow_users` 可能包含 configd；更新后重启 configd |
| `robotd` | configd 不直接控制机器人，只查询其运行状态 |
| `padd` | 非特权进程，使用与手机应用相同的 API；不配置 BlueZ |
| `NetworkManager` | wifi 配置的实际执行者 |
| `BlueZ` | 蓝牙配对的实际执行者 |
| `logind` | 重启的实际执行者 |
| `systemd` | 服务状态查询的实际执行者 |

## 六、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **最小 root** | configd 以沙箱化 root 运行，仅用于需要 root 的操作 |
| **权限分层** | btd 非特权解析不可信字节，configd root 执行特权操作 |
| **trait 抽象** | 外部依赖抽象为 trait，可在无无线电的笔记本上测试 |
| **假实现** | FakeNet/FakePads 使整个 API 表面可测试，可安排困难场景 |
| **单一职责** | 每个模块负责一个配置领域（网络/蓝牙/身份/电源/状态） |
| **优雅操作** | 重启通过 logind 而非 `reboot(2)`，确保服务优雅退出 |
#（注：内容由AI生成）
