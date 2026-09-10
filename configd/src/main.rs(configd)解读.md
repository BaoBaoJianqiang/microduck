# `main.rs`（configd）解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 二进制入口（main.rs） |
| 角色 | `configd` 二进制的参数解析、日志和启动 |
| 平台 | Linux 运行；非 Linux 可构建测试但不能实际操作硬件 |

## 二、CLI 参数

### 核心参数

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `--socket` | `PathBuf` | `duck_ipc_proto::socket::CONFIG` | configd 监听的 unix socket 路径 |
| `--store` | `PathBuf` | `/etc/robot/config.json`（或类似） | 配置存储文件路径 |
| `--name` | `Option<String>` | None | 固定机器人名称，覆盖 identity 派生的默认名 |
| `--allow-user` | `Vec<String>` | [] | 被允许执行变更操作的用户名（窄授权） |
| `--fake-net` | bool | false | 使用假网络实现（测试/台架） |
| `--fake-pads` | bool | false | 使用假手柄实现（测试/台架） |

### `--allow-user` 的设计

- 与 `updater.toml` 的 `allow_users` 相同模式：按名称授权而非按组。
- 窄主张："btd 可以中继来自应用的请求"。
- 命名 `robot` 组反而会把"可读状态"和"可改变配置"合并成一个权限。
- 多个 `--allow-user` 可重复指定。

## 三、启动流程

### 1. 初始化日志

- `tracing_subscriber::fmt()`，`env-filter` 从 `RUST_LOG` 环境变量。
- 默认 `info` 级别。
- 输出到 stderr。

### 2. 记录启动身份

- `duck_ipc_proto::log_startup_identity!("configd")`。
- 发布版本、git 修订到 `/run/configd/identity.json`。

### 3. 加载身份

- `Identity::load()`：从 SoC 序列号派生名称和机器 ID。
- 如果 `--name` 指定，覆盖派生名称。

### 4. 加载存储

- `Store::load(&args.store)`：从文件加载持久化配置。
- 文件不存在则返回空存储。

### 5. 构建子系统

根据 `--fake-*` 参数选择真实或假实现：

| 子系统 | 真实实现 | 假实现 |
|---|---|---|
| 网络 | `nm::NetworkManager::new()` | `net::FakeNet::new()` |
| 手柄 | `bluez::BlueZ::new()`（实现 Pads） | `pad::FakePads::new()` |

### 6. 构建调度器

- 将所有子系统组合成一个 `Dispatcher`。
- `Dispatcher` 负责：
  - 解析 JSON-RPC 请求。
  - 按方法路由到对应子系统。
  - 检查 `--allow-user` 授权。
  - 返回响应。

### 7. 监听 unix socket

- 创建 unix socket，权限 `0660`，属主 `root:robot`。
- 接受连接，每个连接一个任务。
- 读取 NDJSON 行，调度，写回响应。

### 8. 信号处理

- `SIGTERM`（systemd stop）或 `SIGINT`（Ctrl-C）时优雅退出。
- 关闭 socket，等待进行中的请求完成。

## 四、单线程 vs 多线程

### configd 可能使用多线程运行时

- 与 btd 不同（btd 必须单线程防分块乱序）。
- configd 的请求是完整的 JSON-RPC 行，不存在分块乱序问题。
- 可以使用 `rt-multi-thread` 以并发处理多个连接。
- 但 NetworkManager/BlueZ 的 D-Bus 调用可能阻塞，需要多线程避免一个慢调用阻塞所有请求。

## 五、授权模型

### 变更操作需要 `--allow-user`

- 只读操作（`net.status`, `net.scan`, `pad.status`, `system.info`, `system.services`）对所有连接开放。
- 变更操作（`net.connect`, `net.forget`, `pad.pair`, `pad.forget`, `system.setName`, `system.reboot`）需要连接的对等凭据用户名在 `--allow-user` 列表中。

### 对等凭据获取

- 通过 unix socket 的 `SO_PEERCRED` 获取连接进程的 UID。
- 通过 `/etc/passwd` 或 `getpwuid` 解析为用户名。
- 这就是为什么 configd 以 root 运行——需要读取其他进程的凭据。

## 六、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `btd` | 通过 unix socket 连接 configd，转发 `net.*`/`pad.*`/`system.*` 调用 |
| `updaterd` | 更新后可能重启 configd；`allow_users` 模式相同 |
| `robotd` | configd 查询其运行状态（`units.rs`） |
| `padd` | 非特权，使用相同 API；不配置 BlueZ |
| `NetworkManager` | wifi 配置的实际执行者 |
| `BlueZ` | 蓝牙配对的实际执行者 |
| `logind` | 重启的实际执行者 |
| `systemd` | 服务状态查询的实际执行者 |
| `configd.service` | systemd 单元，以 root 运行，沙箱化 |

## 七、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **窄授权** | `--allow-user` 按名称授权变更操作，只读开放 |
| **对等凭据** | 通过 `SO_PEERCRED` 获取连接进程身份，这是 root 的原因之一 |
| **可测试** | `--fake-net`/`--fake-pads` 使无无线电的笔记本可测试整个 API |
| **优雅退出** | 信号处理，关闭 socket，等待进行中请求 |
| **身份发布** | 启动时发布版本/git 修订到 `/run/configd/identity.json` |
| **持久化** | 配置存储到文件，原子写入防崩溃损坏 |
#（注：内容由AI生成）
