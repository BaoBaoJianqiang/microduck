# policy.rs 文件解析

## 文件位置

`d:\microduck\duck-control\src\policy.rs`

## 核心设计决策

`policy.rs` 加载并运行 ONNX 策略网络。行走与站立按速度命令模长选择（与 `microduck_runtime` 一致）；技能网络（坐站、地面拾取、左右踢、翻滚）由 `robotd` 的调度器按优先级显式选择。所有网络共享同一 61-D 观测布局，技能只是会话选择 + 命令块编码，从不产生新契约。

**一切在加载时验证，而非推理时。** 观测宽度错误、动作数错误、ONNX Runtime 缺失，都必须在机器人静止、可告知调用方原因时失败——而非 60 个 tick 后、步态中途失败。`robotd` 将加载失败转为"保持位姿并报告不健康"，使更新器回滚发布而非留下一个不会走的机器人。

## 常量与错误类型

- `DEFAULT_STANDING_THRESHOLD = 0.05` — 低于此速度模长切换站立策略
- `INTRA_THREADS = 1` — 每会话推理线程数（原型用 2，但四核 A55 上小网络同步开销大于并行收益）

### `PolicyError`
- `Load` — 加载 ONNX 文件失败
- `Shape` — 图形状不匹配（同时报告期望与实际形状）
- `Inference` — 推理失败
- `RuntimeMissing` — ONNX Runtime 不可加载（含搜索路径与详情）
- `RuntimePanic` — `ort` 内部 panic（含 panic 消息，因实际遇到的 panic 命名了两个解释问题的版本号）

## 关键函数

### `dylib_name() -> String`
复制 `ort` 自身逻辑：优先 `ORT_DYLIB_PATH` 环境变量，否则按平台返回 `onnxruntime.dll` / `libonnxruntime.dylib` / `libonnxruntime.so`。

### `ensure_runtime() -> Result<()>`
在调用 `ort` **之前**确认 ONNX Runtime 可加载。因为 `ort` 在 dylib 缺失时不返回错误，而是在 `setup_api` 内 `expect`，从任何 API 调用可达的懒加载路径中 abort 线程——在控制循环中意味着线程死亡、无 tick 落地、`robot.health` 永远报告"循环未完成一个周期"，守护进程看起来卡住而非说 ONNX Runtime 未安装。

用 `libloading` 探测（与 `ort` 内部同一加载器），成功则泄漏库句柄（OS 引用计数，`ort` 稍后会再次 dlopen 同一文件）。仅证明文件可加载，不证明 `ort` 不会 panic（实测 ONNX Runtime 1.20.1 加载成功但 `ort` 因版本检查 `expected >= '1.23.x', got '1.20.1'` panic）。

### `catching_ort_panics<T>(work) -> Result<T>`
用 `std::panic::catch_unwind` 包裹 `ort` 调用，将 panic 转为 `PolicyError::RuntimePanic`。`ort` 把某些初始化失败视为不可恢复而 panic 而非返回 `Err`。在控制线程中 panic 比错误更糟：线程死亡、无 tick、健康报告"循环未完成周期"（唯一点名不出原因的消息），更新器因无人可执行的原因回滚。

`AssertUnwindSafe` 因 `Session` 非 `UnwindSafe`。**`panic = "abort"` 会破坏此机制**——根 `Cargo.toml` 无 `[profile.release]`，默认 unwind 策略生效。

### `panic_message(payload) -> String`
提取 panic 消息：`&'static str` 或 `String`，否则返回占位文本。

## 策略网络

### `Net` 枚举
`Walk`、`Stand`、`SitStand`（twist vx 槽携带姿势标志 1=坐/0=站）、`GroundPick`（twist 槽携带 `[cos φ, sin φ, 0]`）、`KickLeft`、`KickRight`、`Roulade`（前滚翻，所有命令槽为零，切换即开始）。

### `PolicyPaths`
`walk` 必需，其余为可选能力（`None` 表示机器人无此能力）。

### `Policy`
持有各网络 `Session`、`standing_threshold`、`standing_disabled`（翻滚/跌倒恢复模式保留站立网络，命令模长不得选择它）。

- `load(paths, standing_threshold)` — 加载、验证、预热。每个会话用全零观测 `Observation::zeroed()` 预热（首次推理总是离群值——懒初始化、冷页、首访缺页，在 tick 1 支付会像控制循环错过截止时间）。
- `set_standing_disabled` — 保留站立网络。
- `will_stand(twist_magnitude)` — 是否选择站立策略（与 `infer` 分离，调用方需同一答案决定增益和动作缩放，问两次不得不一致）。
- `has_*` — 各能力是否加载。
- `infer(observation, net) -> [f32; 14]` — 指定网络推理，缺失可选网络回退行走（错误步态胜过死控制线程）。

### `open(path)` / `check_width` / `run`
- `open`：`Level3` 图优化、1 线程、从文件提交，然后 `check_width` 验证输入宽度 = `OBS_LEN`、输出宽度 = `ACTION_LEN`。
- `check_width`：检查单张量出口的尾部维度（批量维通常动态为 -1，只检查最后一维）。
- `run`：构造 `[1, OBS_LEN]` 输入张量，`session.run`，提取 `f32` 输出，校验长度后复制到 14 元数组。

## 单元测试描述

- `the_standing_threshold_matches_the_prototype`：阈值 = 0.05。
- `without_a_standing_policy_it_never_stands`：无站立策略时永不站立。
- `disabling_standing_beats_the_magnitude_rule`：`standing_disabled` 优先于模长规则。
- `a_panic_out_of_ort_becomes_an_error_that_keeps_its_message`：`ort` panic 转为 `RuntimePanic` 且保留版本号消息（1.23.x、1.20.1）。
- `the_catch_is_transparent_when_nothing_panics`：无 panic 时值直通。
- `an_unprintable_panic_payload_still_reports_something`：非 str/String panic 载荷仍产生原因并指向日志。

## 关键摘要

`policy.rs` 管理 ONNX 策略的加载、验证、预热与推理。核心防护：`ensure_runtime` 在调用 `ort` 前探测 dylib、`catching_ort_panics` 将 `ort` panic 转为可报告错误（避免控制线程死亡）。所有网络共享 61→14 契约，加载时校验形状。行走/站立按速度模长选择，技能网络由 `robotd` 显式调度。`panic = "abort"` 会破坏 panic 捕获机制。
