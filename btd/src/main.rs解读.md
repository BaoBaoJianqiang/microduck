# `main.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 二进制入口（main.rs） |
| 行数 | 190 行 |
| 角色 | `btd` 二进制的参数解析、日志和启动 |
| 平台 | Linux 运行；非 Linux 构建并测试但不能服务 BLE |

## 二、CLI 参数（`Args` 结构体）

### `--update-socket`

- `updaterd` 的 socket。
- 默认值：`duck_ipc_proto::socket::UPDATER`。

### `--robot-socket`

- `robotd` 的 socket。
- 默认值：`duck_ipc_proto::socket::ROBOT`。

### `--config-socket`

- `configd` 的 socket——wifi 和机器人身份。
- 默认值：`duck_ipc_proto::socket::CONFIG`。

### `--require-pairing`（bool）

- 要求配对的、加密的链接。

#### **默认关闭，且这不是最终状态**

- 要求配对使版本读取在 macOS 上挂起——CoreBluetooth 发出 Read Request，BlueZ 因加密不足拒绝它，什么都不解决——因此服务安全配置的机器人根本无法被对话（`docs/design/app-path-design.md` §5.5）。
- 在安全但不可用的默认值和可用但不安全的默认值之间，这是发货前的开发工具，可用的赢。
- **代价是真实且未对冲的**：配对关闭时，范围内任何人都可以在 PIN 穿过时读取它，并写入任何允许的请求——包括携带 wifi 密码的 `net.connect`。每个运行此的机器人都是其 wifi 凭据可被旁观者读取的机器人。
- **这必须在任何东西交给任何人之前翻转。§8.1 是阻碍。**

### `--insecure-no-pairing`（bool，隐藏）

- 被接受并忽略：不要求配对现在是默认值。
- 仅保留以便携带 `--insecure-no-pairing` 的板——这是该标志存在时的使用方式——在删除它的更新上不会启动失败。
- 未知参数会恰好在正在使用它的板上使 BLE 宕机。
- `hide = true`：不在帮助中显示。

### `--name`（Option<String>）

- 固定广告名称，而非询问 `configd` 机器人叫什么。
- 台架使用。某人在手机蓝牙列表中看到的名称是 `configd` 的——用 `robotctl system set-name` 或应用的 `system.setName` 设置——传递此停止它被协调，因此重命名在标志被移除前不会生效。

## 三、`hostname()` 函数

```rust
fn hostname() -> String {
    std::fs::read_to_string("/etc/hostname")
        .map(|s| s.trim().to_owned())
        .ok()
        .filter(|s| !s.is_empty())
        .unwrap_or_else(|| "robot".to_owned())
}
```

- 读 `/etc/hostname` 而非 `hostname` crate 或 libc 调用：一次文件读取，无依赖，且这是板实际配置的内容。
- 空或失败回退到 `"robot"`。

## 四、单线程运行时（承重设计）

```rust
#[tokio::main(flavor = "current_thread")]
async fn main() -> ExitCode {
```

### **故意单线程**，`bluer` 自己的示例也这样做

- 分块请求作为几次 `WriteValue` 调用到达，`dbus-crossroads` 将每个作为自己的任务分发。
- 在多线程运行时上，这些任务可以乱序调用，重排的块不会失败——它重组为解析为错误内容的东西。
  - 示例：`{"id":1,"jsonrpc":"2.0","method":"system.info","params":{}}` 作为块 1, 3, 2 到达变成 `{"id":1,"jsonrpc":"2.info","params":{}}`：有效的 JSON，缺少字段，解析错误归咎于客户端。
  - 在硬件上花了两轮调试。
- 在一个线程上，分发器按从 D-Bus socket 读取的顺序调用处理程序，这是客户端发送它们的顺序——客户端在发送下一个之前确认每个写入，因此该顺序是良定义的。

### 可负担

- 此守护进程不做 CPU 工作：它在无线电和三个 unix socket 之间移动字节。
- 以后添加的任何阻塞东西都会停滞整个服务，这是保持它这样的理由而非反对它的论据。

## 五、`main()` 启动流程

1. **初始化日志**：`tracing_subscriber::fmt()`，`env-filter` 从 `RUST_LOG` 环境变量，默认 `info`，输出到 stderr。
2. **解析参数**：`Args::parse()`。
3. **记录启动身份**：`duck_ipc_proto::log_startup_identity!("btd")`。
4. **构建 Sockets**：三个 socket 路径。
5. **构建 NameChoice**：
   - `pinned: args.name`
   - `fallback: hostname()`
   - 主机名现在只是最后手段：`configd` 从板的 SoC 序列号派生可区分的默认值，因此 `radxa-zero3` 只在完全无法到达 `configd` 时出现。
6. **警告 `--insecure-no-pairing`**：如果设置，警告它现在是默认值且什么都不做。
7. **警告无配对**：如果 `!require_pairing`，每次启动都大声警告（选择可用默认值的全部意义是不安全保持可见而非变成没人记得的东西）。
8. **调用 `run(sockets, name, require_pairing)`**。

## 六、`run()` 函数

### Linux 版本

```rust
#[cfg(target_os = "linux")]
async fn run(sockets: Sockets, name: NameChoice, require_pairing: bool) -> ExitCode {
    tokio::select! {
        result = btd::bluez::serve(sockets, name, require_pairing) => match result {
            Ok(()) => { tracing::error!("..."); ExitCode::FAILURE }
            Err(e) => { tracing::error!(...); ExitCode::FAILURE }
        },
        () = shutdown() => { tracing::info!("shutting down"); ExitCode::SUCCESS }
    }
}
```

- `serve` 原地重试无线电，预期根本不返回：缺失、未供电或楔住的适配器在那里处理而非死亡并让 `Restart=always` 做。
- 因此两个分支都是 `serve` 中的 bug，不是无线电故障——保留因为签名允许它们，非零因为已停止服务 BLE 的 `btd` 不能看起来健康。
- `shutdown()` 在 SIGTERM（systemd stop）或 SIGINT（Ctrl-C）时解析。

### 非 Linux 版本

```rust
#[cfg(not(target_os = "linux"))]
async fn run(...) -> ExitCode {
    tracing::error!("btd needs BlueZ, which is Linux-only. ...");
    ExitCode::FAILURE
}
```

- 此守护进程在非 Linux 上没有东西可服务，直说而非假装。
- crate 仍然在这里构建和测试，这是重点：笔记本上的 `cargo test` 是入职路径，只有无线电是 Linux-only。

## 七、`shutdown()` 函数

- 解析 SIGTERM（systemd stop）或 SIGINT（Ctrl-C）。
- 如果无法监听 SIGTERM，警告并返回 `pending()`（永远等待）——Ctrl-C 仍然有效。
- `tokio::select!` 在两者任一发生时返回。
#（注：内容由AI生成）
