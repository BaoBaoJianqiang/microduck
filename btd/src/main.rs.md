# `main.rs` 文件解析 —— 程序入口、参数解析与启动

## 1. 文件定位

- 路径：`src/main.rs`
- 角色：`btd` 二进制的入口。只负责参数解析、日志初始化和启动流程；"它是什么、为什么不持有状态"见 `lib.rs` 的 crate 文档。
- 该二进制只在机器人上运行；在非 Linux 平台上编译出来也只会报错退出（见第 7 节）。

## 2. 命令行参数（`Args`）

使用 `clap` 的派生宏 `#[derive(Parser)]`。`long_about` 说明：对外提供一个 GATT 服务，承载与其他传输相同的 JSON-RPC 行，把每个请求转发给拥有它的服务；暴露的是一个子集——状态查询、更新触发与进度，**绝不包含电机控制**。

各参数：

- `--update_socket`：`updaterd` 的 socket 路径，默认值 `duck_ipc_proto::socket::UPDATER`；
- `--robot_socket`：`robotd` 的 socket 路径，默认值 `duck_ipc_proto::socket::ROBOT`；
- `--config_socket`：`configd` 的 socket 路径（wifi 与机器人身份），默认值 `duck_ipc_proto::socket::CONFIG`；
- `--require-pairing`（`bool`）：要求已配对、加密的链路。**默认关闭**；
- `--insecure-no-pairing`（`bool`，`hide = true`）：接受但忽略，仅为兼容旧板卡上的 systemd drop-in；
- `--name`（`Option<String>`）：钉死广播名，不再向 `configd` 询问机器人名字，仅供台架（bench）使用。

### 2.1 关于 `--require-pairing` 默认关闭

文档明确记录这是出厂前的开发期取舍，而非最终状态：

- 强制配对会让 macOS 上的版本读取挂起——CoreBluetooth 发出 Read Request，BlueZ 以加密不足拒绝，且没有任何机制能解开（见 `docs/design/app-path-design.md` §5.5）；
- 在"安全但不可用"与"可用但不安全"之间，开发期选择可用；
- 代价真实且未对冲：配对关闭时，无线电范围内任何人都能读到明文经过的 PIN，也能写入任何被允许的请求——包括携带 wifi 口令的 `net.connect`；
- 注释标注：交付给任何人之前必须翻转此默认值，§8.1 是阻塞项。

### 2.2 关于 `--insecure-no-pairing`

保留这个被忽略的参数是出于升级安全：使用该标志的旧 board 带了一个 drop-in 文件；若升级后参数变成"未知参数"，恰好会让正在使用它的板卡上的 BLE 起不来。

## 3. `hostname()` 函数

```rust
fn hostname() -> String
```

- 直接读 `/etc/hostname`，去空白，空内容或读取失败时回退为 `"robot"`；
- 不引入 `hostname` crate、也不做 libc 调用：一次文件读取、零依赖，且读到的正是板卡实际配置；
- 现在它只是**最后手段**：`configd` 会从板载 SoC 序列号派生一个可区分的默认名，`radxa-zero3` 只在完全联系不上 `configd` 时才出现。

## 4. 故意使用单线程运行时

入口标注为：

```rust
#[tokio::main(flavor = "current_thread")]
```

原因（也是硬件上两轮调试换来的教训）：

- 一个分片请求由多次 `WriteValue` 调用组成，`dbus-crossroads` 把每次调用分发为独立任务；
- 多线程运行时这些任务可能乱序执行，而乱序分片**不会报错**，只会重组成"能解析但内容错误"的东西；
- 示例：`{"id":1,"jsonrpc":"2.0","method":"system.info","params":{}}` 按 1、3、2 的分片顺序到达，会变成 `{"id":1,"jsonrpc":"2.info","params":{}}`——合法 JSON、缺字段、解析错误却归咎于客户端；
- 单线程下，分发器按从 D-Bus socket 读到的顺序（即客户端发送顺序，且客户端每写一片都有应答）调用处理函数。

可承受的原因：本守护进程不做 CPU 计算，只在无线电与三个 unix socket 之间搬运字节。注释同时提醒：以后加入任何阻塞操作都会拖垮整个服务——这是保持单线程的理由，而非反对它的理由。

## 5. `main()` 启动流程

1. 初始化 `tracing_subscriber`：输出到 **stderr**，过滤器取 `RUST_LOG` 环境变量，缺省为 `info`；
2. `Args::parse()` 解析参数；
3. `duck_ipc_proto::log_startup_identity!("btd")` 记录启动身份行（版本、revision、可执行路径）；
4. 组装 `Sockets { updater, robot, config }`；
5. 组装 `NameChoice { pinned: args.name, fallback: hostname() }`；
6. 若给了 `--insecure-no-pairing`，打一条 warn 提示它已无作用、应从 unit 文件移除；
7. 若未要求配对，**每次启动都高声警告**：PIN 与 wifi 口令以明文经过链路，范围内任何人可读，仅限开发使用。刻意保持醒目，以免不安全变成没人记得的默认；
8. 调用 `run(sockets, name, args.require_pairing).await` 并以其 `ExitCode` 退出。

## 6. Linux 下的 `run()`

```rust
#[cfg(target_os = "linux")]
async fn run(sockets: Sockets, name: NameChoice, require_pairing: bool) -> ExitCode
```

核心是一个 `tokio::select!`，两个分支：

- `btd::bluez::serve(...)`：本应就地无限重试、永不返回。因此：
  - 返回 `Ok(())` 属于 `serve` 的 bug（"BLE 服务居然返回了，它本该重试"），记 error 并以 `ExitCode::FAILURE` 退出；
  - 返回 `Err(e)` 同样记 error 并失败退出。
  - 非零退出的理由：一个已停止提供 BLE 的 `btd` 不应该看起来健康。
- `shutdown()`：优雅关闭分支，记 info 日志后以 `ExitCode::SUCCESS` 退出。

## 7. 非 Linux 下的 `run()`

```rust
#[cfg(not(target_os = "linux"))]
async fn run(...) -> ExitCode
```

- 明确报错：btd 需要 BlueZ，而 BlueZ 仅 Linux 可用；
- 这个二进制在此平台存在的意义只是让 crate 能编译、测试能跑（笔记本上的 `cargo test` 是新人入门路径），并如实说明无法提供 BLE 服务；
- 返回 `ExitCode::FAILURE`，不假装自己在工作。

## 8. `shutdown()` 信号处理

```rust
#[cfg(target_os = "linux")]
async fn shutdown()
```

- 监听 SIGTERM（systemd 停止服务）与 SIGINT（Ctrl-C），任一到达即返回；
- 若无法注册 SIGTERM 监听，记 warn 并永久挂起（`std::future::pending()`），仍保留 Ctrl-C 的可能。

## 9. 本文件要点小结

1. 入口只管参数、日志、启动，逻辑全部在库 crate 中；
2. 三个上游 socket 均可通过命令行覆盖，默认值来自协议 crate；
3. 配对默认关闭是有明确记录的开发期取舍，且每次启动高声警告；
4. `current_thread` 单线程运行时是为防止 BLE 写分片被多线程乱序调度；
5. 无线电故障在 `bluez::serve` 内部重试；能从 `run()` 返回的都按 bug/失败处理；
6. 非 Linux 平台可编译可测试，但如实报错、拒绝伪装。
