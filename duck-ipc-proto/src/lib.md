# lib.rs 文件解析

**文件位置**：`d:\microduck\duck-ipc-proto\src\lib.rs`

## 一、核心设计决策

### 1.1 定位与边界

`duck-ipc-proto` 是机器人各服务（`updaterd`、`robotd`、`configd`、`padd`、`tofd`）与其客户端（`robotctl`、`btd`、未来的 App、`mediad`）之间的 **IPC 契约**。它定义：

- 线格式（wire format）：JSON-RPC 2.0 over Unix socket，NDJSON 帧（一行一个对象）
- 方法名、参数类型、返回类型
- 错误码
- 服务套接字路径、身份发布机制

**依赖极少**（serde、serde_json、semver），因为 `btd` 处于恢复路径上，不能引入 http/tar/crypto/异步运行时。

### 1.2 两个命名空间

- `update.*`：`updaterd` 的 API（更新、回滚、查询）
- `robot.*`：`robotd` 的 API，刻意精简——只包含 `updaterd` 判断"更新是否安全、是否生效"所需的内容

后续扩展出 `net.*`、`system.*`、`pad.*`、`tof.*`、`chorale.*` 命名空间。

### 1.3 协议版本（API_VERSION）

当前 `API_VERSION = 16`。关键设计原则：

- **版本号不承诺任何双向兼容**——v5 是 additive，v4 不是，但常量不区分
- **不在握手处拒绝**：`updaterd` 过去用 `!=` 拒绝 `hello`，现已移除；版本差异只记录在 journal 与 `HelloResult::api_version` 中
- **真正拒绝的地方更窄**：
  - 未知方法 → `METHOD_NOT_FOUND`（指名方法）
  - 未知 `params` 成员 → `INVALID_PARAMS`（指名成员），因为所有 params 类型都 `deny_unknown_fields`
- 这解决了 v7 的实际事故：`ApplyOptions::from_dir` 若被旧 daemon 静默忽略，操作者以为在 sideload，实际从配置源安装

### 1.4 类型配对机制

每个请求通过 `Call` 枚举构建（`Request::call`），通过 `Request::as_call` 解析回 `Call`。方法名与参数类型在 `Call::method()`、`Call::params()`、`Call::parse()` 三处配对，编译器无法将一个方法配到另一个方法的参数上。

### 1.5 通道（Lane）与服务（Service）分离

- 每个服务一个套接字，无 broker（直连）
- 按"调用持有连接多久"分四车道：`Prompt`（即时）、`Slow`（秒级）、`Operation`（分钟级，改变机器人）、`Stream`（永不回复，推通知）
- 每个服务每个 session 最多 4 个套接字，避免单连接队列把快调用堵在慢调用后（如 `update.apply` 后 `update.status` 超时）

### 1.6 持续意图 vs 离散意图

JSON-RPC 的两类消息正好映射：
- **持续**（`robot.move`、`robot.head`、`robot.pose`、`robot.mouth`、`robot.sound`）：作为 notification 发送（无 id、无回复），20-50 Hz，last-writer-wins。后续走 WebRTC 时自然落在不可靠通道
- **离散**（`robot.stop`、`robot.enable`、`robot.do`）：作为 request 发送，需要回复（被拒绝时给出理由）

## 二、常量与模块

### 2.1 顶层常量

| 常量 | 值 | 含义 |
|---|---|---|
| `JSONRPC_VERSION` | `"2.0"` | JSON-RPC 版本字符串 |
| `API_VERSION` | `16` | 协议版本，每次不兼容变更递增 |
| `UPDATE_MAX_SILENCE_SECONDS` | `600` | 更新期间客户端静默上限（预安装 hook 的天花板），所有客户端必须将自己的 idle budget 设得比它大 |
| `DEFAULT_SOCKET` | `"/run/updaterd.sock"` | updaterd 默认套接字 |

### 2.2 `socket` 模块

各服务默认监听路径（匹配 shipped units，非硬约束，均可 `--socket` 覆盖）：

- `UPDATER` = `/run/updaterd.sock`
- `ROBOT` = `/run/robotd.sock`
- `CONFIG` = `/run/configd.sock`
- `PAD` = `/run/padd/pad.sock`（`padd` 自己回答 `pad.input`，位于其 `RuntimeDirectory=` 下，systemd 停止时删除）
- `TOF` = `/run/tofd/tof.sock`

### 2.3 运行时路径函数

- `identity_path(service)` → `/run/<service>/identity.json`
- `runtime_root()` → `/run`，或 `DUCK_RUNTIME_DIR` 环境变量（仅为测试与笔记本手动运行存在）

每服务一个目录而非共享目录，因为 `RuntimeDirectory=<service>` 是 `ProtectSystem=strict` 下唯一可写处，且 systemd 停止时自动删除，不会留下陈旧身份。

### 2.4 `JOINT_NAMES`

15 个关节名，顺序为：左腿(5) · 颈/头/嘴(5) · 右腿(5)。

**放在此处而非 `duck-control::model`** 是因为它本身就是协议：`RobotState::joints` 与 `targets` 是裸数组，客户端需要按索引命名。`duck-control` 重新导出此表，使线序与驱动序为同一份。

### 2.5 `method` 模块

所有方法名常量，按命名空间分组：

- `hello`、`update.*`（check/apply/rollback/resetToGolden/select/pin/status/listInstalled/log/show/subscribe/progress）
- `robot.*`（safeToRestart/health/modelApi/remoteSessionActive/move/head/look/stop/enable/init/relax/do/pose/mouth/sound/theremin/chorale/shutdown/mode/setMode/subscribe/state）
- `net.*`（status/scan/connect/forget）
- `system.*`（info/services/setName/reboot/pairingPin/setPairingPin/authenticate）
- `pad.*`（status/pair/forget/input/report）
- `tof.*`（stream/frame）
- `chorale.*`（subscribe/beacon/heard）

### 2.6 `code` 模块

JSON-RPC 错误码。Spec 保留段（-32768..-32000）：`PARSE_ERROR`、`INVALID_REQUEST`、`METHOD_NOT_FOUND`、`INVALID_PARAMS`、`INTERNAL_ERROR`。

应用段：`BUSY(1)`、`UNKNOWN_COMPONENT(2)`、`PROTOCOL_MISMATCH(3)`（已退役，保留编号不复用）、`PREFLIGHT_FAILED(4)`、`NETWORK(5)`、`VERIFICATION_FAILED(6)`、`INCOMPATIBLE(7)`、`HOOK_FAILED(8)`、`HEALTH_CHECK_FAILED(9)`、`ROLLBACK_FAILED(10)`、`NOT_INSTALLED(11)`、`WOULD_DOWNGRADE(12)`、`ARCHIVE_TOO_LARGE(13)`、`PERMISSION_DENIED(14)`。

## 三、核心类型

### 3.1 `Call` 枚举

每个变体 = 一个方法 + 其参数类型。关键方法：

- `method()` → 方法名字符串
- `is_mutating()` → 是否改变机器人软件/配置（`updaterd` 据此按 uid/gid 授权）
- `destination()` → `Option<(Service, Lane)>`，`None` 仅 `SystemAuthenticate`（由传输层处理）和 `ChoraleBeaconSet`/`ChoraleHeard`（走 `chorale.subscribe` 已开的连接）
- `component()` → 涉及的组件（若有）
- `params()` → 序列化参数
- `parse(method, params)` → 解码，未知方法返回 `METHOD_NOT_FOUND`，参数不符返回 `INVALID_PARAMS`

### 3.2 `Service` 枚举

`Updater`、`Robot`、`Config`、`Pad`、`Tof`。

### 3.3 `Lane` 枚举

`Prompt`、`Slow`、`Operation`、`Stream`。

### 3.4 `Id`

`#[serde(untagged)]` 的 `Number(u64)` / `Text(String)`。`None` 表示 notification。

### 3.5 `Request` / `Response` / `Error`

- `Request`：`jsonrpc`、`id`（Option）、`method`、`params`（Option）。构造器：`call()`、`notify()`、`notify_state()`、`notify_pad_report()`、`notify_tof_frame()`、`notify_progress()`。解析：`as_call()`、`as_state()`、`as_pad_report()`、`as_tof_frame()`、`as_progress()`
- `Response`：`ok()` / `err()`，`result_as::<T>()`
- `Error`：`code`、`message`、`data`，实现 `Display` 与 `std::error::Error`

## 四、参数类型（params）

所有带参数的类型均 `#[serde(deny_unknown_fields)]`，未知成员拒绝并指名。

### 4.1 更新相关

- `ComponentParams { component }`
- `ApplyParams { component, target: Target, options: ApplyOptions }`
- `Target` 枚举：`Latest`、`Exact(Version)`、`Ref(String)`（绕过降级守卫，因 dev build 是 prerelease）、`Staging`、`StagingExact(Version)`
- `ApplyOptions { dry_run, interrupt_sessions, from_dir: Option<String> }`——`from_dir` 为笔记本到板的 sideload 路径，用 `String` 而非 `PathBuf`（JSON 线类型）
- `SelectParams`、`PinParams`、`LogParams { limit }`、`ShowParams { run: Option<u64> }`

### 4.2 意图参数（单位与坐标系）

**单位约定**：全部弧度与弧度/秒，躯干坐标系，右手系：x 前、y 左、z 上；正 `vyaw` 左转。此段删除了原型机累积的 `--laser-track-yaw-sign` 等一系列符号 flag。

- `MoveParams { vx, vy, vyaw }`（连续，notification）
- `HeadParams { neck_pitch, head_pitch, head_yaw, head_roll }`（关节空间）
- `LookParams { x, y, z, neck_pitch }`（躯干坐标系下的注视点，米；z 是躯干坐标系而非地面，地面约在 -0.12m）
- `EnableParams { on, toggle }`——`toggle` 为手柄 Start 键，机器人持有状态避免客户端信念漂移
- `DoParams { skill: Skill }`——`Skill` 枚举：`GroundPick`、`KickLeft`、`KickRight`、`SitToggle`、`Roulade`
- `PoseParams { z, roll, pitch, active }`（站立身体姿态偏移，训练范围小：z -0.025..+0.010，roll/pitch ±0.26）
- `MouthParams { open }`（0 闭 1 开）
- `SoundParams { tag: SoundTag, hold: Option<bool> }`——`wheee` 为持续音，hold 衰减防止客户端死掉后机器人一直叫
- `ThereminParams { active }`
- `ChoraleParams { active, piece: Option<u8> }`
- `SetModeParams { mode: String }`、`SubscribeParams { hz: Option<u32> }`

### 4.3 网络与系统参数

- `NetConnectParams { ssid, psk: Option<String> }`——**手写 `Debug` 脱敏 psk**，只显示 `<redacted>` 或 `<none>`（presence 有诊断价值）
- `NetForgetParams`、`SetNameParams`、`SetPairingPinParams { pin: String }`（字符串而非整数，保留前导零，默认 `000000`）
- `AuthenticateParams { pin }`——**手写 `Debug` 脱敏 PIN**
- `PadPairParams { mac: Option<String>, timeout_seconds: Option<u32> }`——所有字段可选，`{"method":"pad.pair"}` 是日常调用
- `PadForgetParams { mac }`

## 五、结果类型（results）

### 5.1 更新结果

- `HelloResult { api_version, daemon_version: Option<Version>, revision: Option<String> }`——`revision` 始终序列化（含 `null`），线形状不依赖值
- `Phase` 枚举：`Idle`/`Preflight`/`Checking`/`Downloading`/`Verifying`/`Extracting`/`RunningPreHook`/`Swapping`/`RunningPostHook`/`Applying`/`HealthGate`/`Committing`/`RollingBack`
- `Progress { component, phase, percent: Option<u8>, detail: Option<String> }`
- `ComponentStatus`、`InstalledRelease`
- `CheckResult`：`UpToDate` / `Available`（含 `mandatory`）/ `Incompatible`
- `ApplyResult`：`Applied` / `AlreadyCurrent`（含 `stale: Vec<String>`，标记需重启的 unit）/ `DryRunPassed` / `RolledBack` / `Stuck`（首次安装未起来且无可回滚版本）

### 5.2 日志与 transcript

- `LogEntry { at, component, from, to, outcome: Outcome, run: Option<u64> }`
- `Outcome`：`Success` / `RolledBack { reason }` / `Aborted { reason }`
- `RunRecord { at, event: RunEvent }`（`#[serde(flatten)]`，使每行是扁平对象，便于 `cat`/`grep`）
- `RunEvent`：`Began`/`Phase`/`Manifest`/`Hook`/`Unit`/`Health`/`Note`/`Ended`/`Truncated`/`Unrecognised`（`#[serde(other)]`，未来版本的事件不破坏整个 transcript）
- `RunTranscript { run, component, events, available }`

### 5.3 机器人健康结果

`HealthResult` 是更新健康门控的核心：

- `healthy: bool`
- `degraded: bool`——当原因是板级属性（如无舵机电源）而非发布版本时为 true，门控可据此 commit 而非 rollback
- `reason: Option<String>`
- `battery: Option<Battery>`——**只报告不判断**，扁平电池不应导致 rollback（否则机器人永远无法更新）
- `motors: Option<MotorThermal>`（最热关节而非均值）
- `cpu_temp_c: Option<f64>`（SoC 温度，与电机温度区分）
- `control_loop: Option<LoopHealth>`（target_hz、achieved_hz、ticks、missed、last_tick_age_ms）
- `bus: BusHealth`（consecutive_errors、startup_failures）
- `imu: Option<ImuHealth>`

`ImuHealth` 用 `#[serde(default)]`（整个结构体），因为每个零值都是诚实的"无事报告"。这修复了一次真实回归：`consecutive_stale_blocks` 字段新增后，旧 `robotd` 发送的 `imu` 段缺此字段，常驻 `updaterd` 拒绝整个回复，将正在 50Hz 运行的机器人判为"unreachable"并回滚了好版本。

`ImuHealth::FROZEN_RUN = 25`（50Hz 下半秒），`frozen()` 方法判断朝向是否冻结。

### 5.4 机器人状态流

`RobotState`（`robot.state` notification）：

- `t`（启动后秒数，单调）
- `movement`（序列化名为 `move`，Rust 关键字）
- `head: [f64; 4]`
- `policy: String`（`walk`/`stand`/`held`）
- `safety: SafetyState`（`fallen`、`limp`、`gravity`、`gain`）
- `control_loop`（序列化名为 `loop`）
- `joints: Vec<f64>`、`targets: Vec<f64>`
- `odom: OdomState`（默认，旧 robotd 无此字段）
- `theremin: Option<ThereminState>`（乐器关闭时整段缺失）
- `chorale: Option<ChoraleState>`

`MoveState { requested, applied, limited_by: Vec<String> }`——报告被拒绝的内容（`limited_by` 为空时序列化省略）。

### 5.5 网络/系统/手柄/ToF 结果

- `NetStatusResult`、`Network`（含 `saved`）、`NetScanResult`、`ConnectResult`（`Connected`/`Failed { reason: ConnectFailure, detail }`）、`ForgetResult`
- `SystemInfoResult`（不含版本——`hello` 与 `update.status` 已回答不同问题）、`SetNameResult`、`PairingPinResult`、`RebootResult`（延迟重启，使调用可应答）
- `Pad`（paired/trusted/connected 分开报告）、`ServiceUnit`（从 `/proc/<pid>/exe` 解析真实运行版本）、`PadStatusResult`、`PadPairResult`、`PadForgetResult`
- `TofStreamResult`（`accepted` + `sensor` + `unavailable`，区分"已订阅"与"有传感器"）

### 5.6 手柄输入流（pad.input）

`pad_link` 模块常量：`NOTABLE_MS=100`、`DEADMAN_MS=500`（robotd 速度归零阈值）、`IDLE_MS=5000`（静止时 evdev 不发报告，不算卡顿）。

- `PadInputDevice`（name、node、unique、bus、vendor、product、axes、buttons），`over_bluetooth()` 方法
- `PadAxis`（code、name、min、max、flat、fuzz、value）
- `PadKey`（code、name、pressed）
- `PadReport`：`Attached { device }` / `Frame(PadFrame)` / `Detached { why }`
- `PadFrame`：`seq`、`at_us`（内核时间戳，非 padd 读取时间）、`since_us: Option<i64>`（**有符号**，因 NTP 时钟跳变可能为负）、`events: Vec<PadEvent>`、`after_drop`、`socket_dropped`
- `PadEvent`：`kind`、`code`、`value`、`name`，常量 `SYNCHRONIZATION=0`、`KEY=1`、`ABSOLUTE=3`

### 5.7 ToF 帧

`TofFrame { seq, at_us, rows, cols, distance_mm: Vec<i16>, status: Vec<u8> }`——毫米与 ST 原始状态（不用米，因 JSON 无 NaN，状态字节保留传感器三态回答）。

### 5.8 合唱（chorale）

`ChoraleBeacon` 是**唯一非 JSON 的线契约**，走 BLE 广播（两机器人无网络无共享时钟）：

字段：`piece: u8`、`beat: u8`、`register: u8`（音高中心量化，100-625Hz）、`id: u16`（16 位防碰撞，曾因 1 字节导致四鸭房间碰撞）、`roster: Vec<(u8, u16)>`（座位表，最多 4，防止两鸭唱同声部）。

`TAG = 0xC0` 区分于地址广播载荷。`to_bytes()`/`from_bytes()` 严格校验长度与布局。

`ChoraleAdvertise`、`ChoraleHeard { beacon, from, age_us }`（用年龄而非时间戳，因两进程不共享时钟）。

## 六、构建身份与发布机制

### 6.1 `BuildInfo` 与 `build_info!` 宏

`BuildInfo { version, revision: Option<&'static str>, built_at: Option<&'static str> }`。宏用 `env!("CARGO_PKG_VERSION")`、`option_env!("DUCK_REVISION")`、`option_env!("DUCK_BUILD_TIME")`，在调用点展开（函数会报告本 crate 版本）。

### 6.2 `release_from_path(path)`

从路径中 `releases/<version>/` 段解析版本。处理 ` (deleted)` 后缀（进程仍在运行但二进制已被删除），dev build 保留 prerelease 后缀。

### 6.3 `Identity` 与 `publish_identity`

守护进程自发布身份到 `/run/<service>/identity.json`（service、version、revision、built_at、exe、pid）。**自发布而非从 `/proc/<pid>/exe` 外部读取**，因为进程知道自己的版本/revision，且读 `/proc/self/exe` 无需特权。

`publish_identity` 永不致命（机器人不会因无法描述自己而不启动）。

### 6.4 `CameraStats` 与 `publish_camera_stats`

`mediad` 发布相机统计到 `/run/mediad/camera.json`（文件而非查询，因 mediad 不是请求-响应服务）。`fps` 在 tee 之前测量（项目已四次犯错把下游丢帧率当捕获率）。

### 6.5 `log_startup_identity!` 宏

守护进程启动第一行日志（`warn` 级别，survive `RUST_LOG=warn`）。宏避免引入 `tracing` 依赖。

## 七、test_support 模块

`every_call()` 返回 46 个 `Call` 变体的完整列表（`#[cfg(any(test, feature = "test-support"))]`）。强迫 `method`、`destination` 等穷举 match 保持完整——新变体破坏这些构建。

## 八、单元测试描述

| 测试 | 断言意图 |
|---|---|
| `a_release_path_names_its_version` | 路径中 `releases/0.4.0/` 解析出版本 |
| `a_dev_release_keeps_the_suffix_that_distinguishes_it` | dev build 保留 prerelease 后缀且比较为旧于 release |
| `a_deleted_binary_still_names_its_release` | ` (deleted)` 后缀仍解析出版本 |
| `a_binary_outside_the_layout_has_no_release` | 非布局路径返回 None |
| `an_identity_survives_being_published_and_read_back` | 身份发布/读取往返，通过 serde 双向 |
| `only_authenticate_has_no_service` | 仅 `system.authenticate` 无 destination（由传输层处理） |
| `subscriptions_are_the_only_thing_on_the_stream_lane` | Stream 车道只能是订阅类调用 |
| `every_call_covers_every_variant` | 46 个变体全在 `every_call` 中 |
| `every_call_has_a_distinct_method` | 方法名不重复 |
| `pairing_a_pad_needs_no_parameters` | `pad.pair` 无参数也能解析 |
| `every_call_round_trips_over_the_wire` | 每个 Call 线往返不变 |
| `a_call_serialises_as_jsonrpc` | JSON-RPC 字段正确，`from_dir` 未设置时不出现 |
| `from_dir_round_trips_and_is_omitted_when_unset` | `from_dir` 往返且未设置时省略 |
| `unknown_methods_and_bad_params_get_different_codes` | 未知方法 vs 坏参数不同错误码 |
| `an_unknown_params_member_is_refused_by_name` | 所有带参方法拒绝未知成员并指名 |
| `methods_without_params_accept_any_params_field` | 无参方法接受任意 params |
| `only_software_changing_calls_are_mutating` | 仅改变软件的调用为 mutating |
| `component_carrying_calls_expose_it` | 带组件的调用暴露组件 |
| `notifications_carry_no_id` | notification 无 id |
| `a_notification_parses_without_an_id_field` | 无 id 字段的 notification 可解析 |
| `responses_omit_the_half_they_do_not_use` | 响应不同时带 result 和 error |
| `robot_results_round_trip` | HealthResult reason 省略往返 |
| `pad_reports_round_trip_under_their_tag` | 三种 PadReport 带 tag 往返 |
| `a_clean_frame_carries_neither_alarm` | 干净帧不含 after_drop/socket_dropped/since_us |
| `a_usb_pad_is_not_on_the_radio` | USB 手柄不在蓝牙上 |
| `a_beacon_survives_the_advertisement` | ChoraleBeacon 字节往返，idle 状态，座位查找 |
| `a_roster_of_the_wrong_length_is_not_a_beacon` | 错误长度/计数的载荷不是 beacon，超过 4 截断为 4 |
| `an_address_payload_is_not_mistaken_for_a_beacon` | IPv4 地址载荷不被误判为 beacon |
| `a_quantised_register_is_good_enough_to_cast_from` | register 量化精度足够分声部，全种群不落在 clamp 上 |
| `the_theremin_block_is_absent_until_there_is_a_theremin` | theremin 关闭时字段缺失，静默时 note_hz 也省略 |
| `robot_state_uses_the_documented_field_names` | `move`/`loop` 序列化名正确（Rust 关键字） |
| `an_unlimited_command_omits_limited_by` | 无限制命令省略 limited_by |
| `health_without_the_degraded_field_is_not_degraded` | 缺 degraded 字段默认为 false |
| `degraded_is_omitted_when_false_and_present_when_true` | degraded 省略策略 |
| `an_imu_section_missing_its_newest_field_still_parses` | imu 段缺 `consecutive_stale_blocks` 仍解析（真实回归测试） |
| `a_state_frame_missing_odom_still_parses` | 缺 odom 的 state 帧解析为原点 |
| `a_bus_section_missing_a_counter_still_parses` | 缺总线计数器默认为 0 |
| `a_missing_battery_is_unknown_not_empty` | 缺 battery 是未知而非 0 伏 |
| `build_info_is_explicit_about_an_unknown_revision` | 本地 build 显示 "rev unknown, not a CI build" |
| `build_info_macro_reports_the_calling_crate` | 宏报告调用 crate 版本 |
| `a_wifi_key_is_redacted_from_debug_output` | psk 在 Debug 中脱敏 |
| `a_pairing_pin_is_redacted_from_debug_output` | PIN 在 Debug 中脱敏但仍序列化到线 |
| `a_wifi_key_still_serialises` | psk 仍序列化，开放网络省略字段 |
| `target_round_trips_in_every_form` | Target 五种形式线往返 |
| `a_run_record_is_one_flat_line` | RunRecord 是扁平单行 JSON |
| `an_unknown_run_event_keeps_the_rest_of_the_transcript` | 未知 RunEvent 解码为 Unrecognised |
| `a_log_entry_without_a_run_still_parses` | 旧 log entry 无 run 字段仍解析且不变成 null |
| `a_ref_with_a_slash_survives_the_wire` | 含斜杠的 ref 名原样往返 |
| `hello_result_round_trips_with_and_without_a_revision` | revision 为 None 时序列化为 `null`（线形状统一） |

## 九、关键摘要

1. **契约集中**：所有服务与客户端共用此 crate 的类型，方法与参数在 `Call` 枚举中配对，编译器保证不错配
2. **版本策略反转**：不在握手拒绝，而是在方法/参数层精确拒绝（`deny_unknown_fields` + `METHOD_NOT_FOUND`/`INVALID_PARAMS`），解决了"差异即拒绝"误伤可服务调用的问题
3. **车道分离**：按调用持有时长分 Prompt/Slow/Operation/Stream，避免单连接队列饥饿
4. **安全脱敏**：`NetConnectParams` 与 `AuthenticateParams` 手写 `Debug` 脱敏，线传输仍正常
5. **向前/向后兼容**：`#[serde(default)]` 结构体（ImuHealth、BusHealth）使旧发送者缺字段仍被解析；`RunEvent::Unrecognised` 使 transcript 可跨版本读取；`skip_serializing_if` 省略零值/空值保持线形状稳定
6. **诊断优先**：`RobotState` 报告 `requested`/`applied`/`limited_by`；`HealthResult` 携带电池/温度/循环/总线/IMU 全部测量供人工核查，不由门控自动判断电池
7. **身份自发布**：守护进程写 `/run/<service>/identity.json`，从 `/proc/self/exe` 解析真实运行版本（区分 symlink 指向与实际进程）
8. **BLE 合唱 beacon**：唯一非 JSON 契约，4 字节定长 + roster，无共享时钟靠 beat 到达同步
