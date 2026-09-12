# main.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/main.rs`
- **角色**：`configd` 二进制入口。按文件头自述，本文件负责三件事：**Unix socket 服务端、对端授权策略（PeerPolicy）、请求分发（dispatch）**。为什么独立成服务见 `lib.rs`。

## 2. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `SOCKET_MODE` | `0o660` | 属主与组可读写、其他无权限。访问控制第一层：能连上 socket 先得有组 |
| `MAX_LINE` | `64 * 1024` | 单行请求上限，拒绝缓冲荒谬的长行 |
| `DEFAULT_STATE_DIR` | `/var/lib/robot/config` | 配置文件默认目录，位于任何发布目录之外，更新与回滚都不影响 |

## 3. 命令行参数 `Args`（clap）

命令说明为「Wifi and robot identity」，长说明点明：通过 unix socket 提供 `net.*`/`system.*`、驱动 NetworkManager、永不存凭据、与 robotd 分开以便机器人挂掉时仍可配置。

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--socket` | `proto::socket::CONFIG` | socket 路径 |
| `--state-dir` | `DEFAULT_STATE_DIR` | 配置文件目录 |
| `--allow-user` | 空，逗号分隔（`value_delimiter=','`） | 允许做**变更**的用户名（只读调用永不门控）。按名不按号：`systemd-sysusers` 动态分配 uid，数字在这块板对、下一块就错；无法解析的名字只告警不致命，缺可选用户的机器人仍可提供全部只读服务 |
| `--allow-group` | 空，逗号分隔 | 允许变更的组名 |
| `--fake-net` | flag | 用内存 wifi 栈替代 NetworkManager，演练包括错口令在内的失败 |
| `--fake-pads` | flag | 用内存手柄集合替代 BlueZ。与 `--fake-net` **分开两个开关**：真机 NM 配假无线电这种组合有价值，单一 `--fake` 会逼台架二选一 |

## 4. `PeerPolicy`：两层授权

仿照 `updaterd`（§2.2）：socket 的组决定谁能**说话**，本结构决定谁能**改动**。只读调用刻意不门控——支持人员应能检视一台未被授权重配的机器人，`btd` 也应能只报 wifi 状态而不被信任去加入网络。

字段：`owner_uid`、`allow_uids: Vec<u32>`、`allow_gids: Vec<u32>`。

- `may_mutate(peer: Option<&UCred>) -> Result<(), String>`：
  - `peer` 为 `None`（`SO_PEERCRED` 拿不到凭证）直接拒绝——决策是「能否重启机器人」时，拿不到凭证不是可以耸肩放过的事；
  - `uid == owner_uid`，或 uid 在白名单，或 **gid** 在白名单，放行；
  - 否则返回明确错误文案，提示把用户/组加入 `--allow-user`/`--allow-group`（在 configd.service 中）或以 owner uid 运行。文案特意使用正确的参数名（曾误写 `--allow-uid` 把人引到「unexpected argument」）。

### 名字解析

- `resolve_uid(name)`：`libc::getpwnam`（unsafe，立即读取返回的静态缓冲、不留存指针）；无此用户记 `warn` 返回 `None`，命中记 `info`（含 uid）返回。
- `resolve_gid(name)`：`libc::getgrnam`，同理。
- 启动时解析一次：SO_PEERCRED 报数字，单元文件写名字即可跨 sysusers 分配差异保持正确。

## 5. `Service`、`hostname()` 与 `main()`

- `struct Service`：`net: Arc<dyn Net>`、`pads: Arc<dyn Pads>`、`store: Store`、`policy: PeerPolicy`、`serial: Option<String>`（启动读一次：序列号来自熔丝，进程生命周期内不会变）。
- `fn hostname() -> String`：读 `/etc/hostname`，trim；空/失败回退 `"robot"`。

### `main` 流程

1. 初始化 tracing（stderr，`RUST_LOG`，缺省 `info`）；解析参数；`log_startup_identity!("configd")`。
2. **两个后端都不是拒绝启动的理由**（关键韧性设计）：
   - wifi 后端失败 → 记 `error`，使用 `UnavailableNet::new(e)`；
   - 手柄后端失败（如蓝牙约 73 秒才出现）→ 记 `warn`，使用空的 `FakePads::with(Vec::new())`。
   - 理由：`configd` 还提供 `system.pin`（`btd` 获取手机认证 PIN 的来源），一旦因依赖缺失退出，就把「wifi 不可用」升级成「机器人完全不可达」；同时「等待依赖而非退出」是进入启动恢复网的资格，使 `failed` 单元只代表发布损坏。
3. 身份：读 `identity::serial()`；有序列号则 `default_name`，无则回退 hostname 并 `warn`（同镜像板子在改名前无法区分）。
4. 构造 `Service`（含 `owner_uid = getuid()` 与解析出的 uid/gid 白名单）。
5. `tokio::select!`：`serve` 返回（成功 SUCCESS/失败 FAILURE）或 `shutdown()` 触发（记日志、SUCCESS）。
   - 注意使用默认的**多线程** runtime（`#[tokio::main]` 与 `rt-multi-thread`），与 btd 的 current_thread 不同。

### 后端选择函数

- `backend(fake)`：Linux 且非 fake → `nm::NetworkManager::new()`；fake → `FakeNet`。注释澄清：`new` 只打开系统总线、并不在总线上找 NM，所以此处失败代表 D-Bus 不可达（提示「is dbus running?」），「还在 netplan」这一病因在 `wifi_device` 中诊断并报 `Unavailable`。
- 非 Linux：一律 `FakeNet`（说清楚比拒绝更有用）。
- `pad_backend(fake)`：Linux 非 fake → `bluez::BlueZ::new()`；fake 或非 Linux → `FakePads`（`FakePads::new()` 或空集合由上层决定，非 Linux 直接 `new()`）。

## 6. `serve` 与 `handle`

### `serve(service, socket_path)`

1. 若路径已存在，记 `warn` 并删除残留 socket（被杀进程留下的 socket 不得阻止启动）；确保父目录存在。
2. `UnixListener::bind`，随后 `set_permissions(… 0o660)`。
3. 记 `info`（含 path 与 mode）后循环 `accept`：accept 失败只 `warn` 继续；每个连接 `tokio::spawn` 一个 `handle`。

### `handle(service, stream)`

- **每连接读取一次**对端凭证（`peer_cred().ok()`）：活动 socket 上凭证不会变，按请求问是浪费系统调用。
- 拆分流，按行读取：
  - 空行跳过；
  - 超 `MAX_LINE` → 回 `INVALID_REQUEST`（"request too large"）；
  - JSON 解析失败 → `PARSE_ERROR`；
  - 通知（无 `id`）按规范不回复；
  - `as_call()` 失败回对应错误，否则交 `dispatch`；每行应答经 `write_line` 写出。

## 7. `dispatch`：方法分发

变更类调用（`call.is_mutating()`）**先授权**：失败记 `warn`（method + reason）并回 `PERMISSION_DENIED`；成功记 `info`（method、uid、gid——「谁让这台机器人重启」是支持人员第一个要问的）。

| Call | 处理 |
| --- | --- |
| `Hello` | 返回 `api_version`、从 `CARGO_PKG_VERSION` 解析的 `daemon_version`、`build_info!().revision` |
| `NetStatus` / `NetScan` / `NetForget` | 直接 `reply` 对应后端方法 |
| `NetConnect(params)` | 记 `info!(?params)`——`NetConnectParams` 的手写 `Debug` 会脱敏 PSK；调 `connect(ssid, psk)` |
| `SystemServices` | `units::all()`。**只读不门控**：「实际跑的是哪个发布」是诊断者最先要问的 |
| `SystemInfo` | 返回 store 名字、序列号、`uptime_seconds()` |
| `SystemSetName` | `store.set_name`：成功记 info（`btd` 数秒内会按新名字协调广播，手机下次扫描即见）并返回实际名；失败回 `INVALID_PARAMS` |
| `SystemPairingPin` | `store.name_and_pin_result()`（含 is_default） |
| `SystemSetPairingPin` | 成功记「pairing PIN changed」（**PIN 本身不入日志**：默认 PIN 虽公开，每机 PIN 应保密，journal 不是放它的地方）；校验失败回 `INVALID_PARAMS` |
| `PadStatus` | 成功返回 `pads` 并附带 `driver: units::state(units::PADD)`；后端失败回 `INTERNAL_ERROR` |
| `PadPair` | 用 `pad::pair_timeout(timeout_seconds)` 裁剪超时，记 mac/timeout 后调 `pair` |
| `PadForget` | 调 `forget(mac)` |
| `SystemReboot` | 先 `power::schedule()`，立即返回 `RebootResult { in_seconds: 3 }` |
| 其他（`update.*`/`robot.*` 等） | `METHOD_NOT_FOUND`，文案「{method} is not served by configd」——客户端打错了 socket，应明说而非泛化失败 |

### `reply<T>(id, Result<T, String>)`

后端结果转响应：`Ok` → 成功响应；`Err`（机器坏了）→ 记 `warn` 并回 `INTERNAL_ERROR`。业务*拒绝*是携带 `Failed` 的成功调用，两者区别决定客户端如何行动。

### 辅助

- `uptime_seconds()`：读 `/proc/uptime` 首个浮点秒数取整；无 procfs（开发笔记本）返回 0。
- `write_line`：序列化为 JSON、追加 `\n`、写并 flush。
- `shutdown()`：监听 SIGTERM（systemd stop）与 SIGINT（Ctrl-C）；SIGTERM 监听建立失败时记 warn 并永久 pending。

## 8. 要点小结

- 0660 socket 组 + uid/gid 白名单构成「能连/能改」两层授权；只读不门控、未知对端拒绝、按名解析身份。
- 后端缺失一律降级为 `UnavailableNet`/空 `FakePads` 而不退出，保住 PIN 下发与启动恢复网资格。
- 每连接 spawn、按行 JSON 通信，超长/解析错误有明确错误码，通知不应答。
- 敏感信息处理：PSK 靠类型的手写 Debug 脱敏，PIN 绝不入日志。
- 重启先调度后应答（3 秒）；非本服务方法返回带说明的 METHOD_NOT_FOUND；SIGTERM/SIGINT 优雅退出。
