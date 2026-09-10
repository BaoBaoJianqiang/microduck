# `policy.rs` 解读

## 概述

`policy.rs` 是 `duck-control` crate 中负责 **ONNX 策略网络推理**的模块（约 510 行）。它把训练好的 `.onnx` 策略模型加载进 ONNX Runtime，并在每个控制 tick 上把 61 维观测向量喂进去、取回 14 维关节动作。

文件的核心设计意图有三条：

1. **走/站由速度命令幅度自动选择，技能网络由调度器显式点名。** 走路与站立是一对连续策略——速度命令幅度低于阈值就切到站立策略，这与 `microduck_runtime` 的做法完全一致；而 sit↔stand、捡拾、左右踢、前滚翻这些"技能"网络则由 `robotd` 里的技能调度器（拥有优先级规则）显式选择。所有网络共享同一套 61 维观测布局，所谓"技能"只是换一个 session 加一段命令位编码，从不引入新的观测契约。
2. **一切在加载时验证，而不是在推理时验证。** 观测宽度不对、动作数不对、ONNX Runtime 缺失——这些必须在机器人静止、调用方还能知道原因的时候就失败，而不是走到第 60 个 tick、正在迈腿时才崩。`robotd` 把加载失败转成"保持姿态并上报不健康"，于是更新器会回滚这次发布，而不是留下一只不会走路的机器人。
3. **把 `ort` 库的"野蛮失败"（panic / abort）收编为普通 `Result` 错误。** 控制线程里一次 panic 等于整个控制循环死掉、健康状态永远报"尚未完成一个周期"，却不指明原因。本模块用"先探 dylib + 再 catch_unwind"两层手段把这些情况变成可上报、可回滚的错误。

## 关键常量

```rust
/// Below this velocity magnitude the standing policy takes over. The prototype's value.
/// 速度幅度低于此值时切到站立策略。这是原型（prototype）沿用的数值。
pub const DEFAULT_STANDING_THRESHOLD: f64 = 0.05;

/// Inference threads per session.
/// 每个 session 的推理线程数。
const INTRA_THREADS: usize = 1;
```

- **`DEFAULT_STANDING_THRESHOLD = 0.05`**：走/站切换的速度幅度阈值。这个数字必须与调参时用的原型一致，否则机器人会在错误的速度上换步态。有专门测试钉住这个值。
- **`INTRA_THREADS = 1`**：刻意设为单线程。原型用 2 线程，但在四核 A55 上意味着控制线程阻塞在一个自己并不拥有的线程池上；对这么小的网络，线程池带来的同步开销超过并行收益。注释提醒：上板后应重新实测，不要盲信任何一个数字。

## 关键类型与函数

### `PolicyError` 错误枚举

```rust
#[derive(Debug, thiserror::Error)]
pub enum PolicyError {
    #[error("loading {path}: {source}")]
    Load { path: PathBuf, #[source] source: ort::Error },
    #[error("{path}: {what} is {got}, expected {expected}")]
    Shape { path: PathBuf, what: &'static str, expected: String, got: String },
    #[error("inference failed: {0}")]
    Inference(String),
    #[error("ONNX Runtime not loadable ({searched}): {detail}")]
    RuntimeMissing { searched: String, detail: String },
    #[error("ort panicked loading the policy: {detail}")]
    RuntimePanic { detail: String },
}
```

五个变体分别对应五类失败：

- **`Load`**：`commit_from_file` 本身失败（文件损坏/格式错）。
- **`Shape`**：bundle 与本编译版本不匹配。注释特别强调要把 `expected` 和 `got` 两个形状都报出来——"策略文件错了"和"守护进程错了"在没有形状对照时看起来一模一样。
- **`Inference`**：运行期推理错误（建输入、取输出、输出长度不对等）。
- **`RuntimeMissing`**：ONNX Runtime 没装，或不在查找路径上。这是**运维问题，由运维解决**（装库或设 `ORT_DYLIB_PATH`），不是策略 bundle 坏了，所以单独成类并把修复办法写进消息。
- **`RuntimePanic`**：`ort` 没有返回错误而是 panic 了（见 `catching_ort_panics`）。`detail` 携带 panic 消息——板子上真实遇到过的那次 panic 直接点名了两个版本号，那就是全部诊断信息。

### `dylib_name`：复刻 `ort` 的动态库查找逻辑

```rust
fn dylib_name() -> String {
    match std::env::var("ORT_DYLIB_PATH") {
        Ok(path) if !path.is_empty() => path,
        _ => {
            if cfg!(target_os = "windows") { "onnxruntime.dll".to_owned() }
            else if cfg!(any(target_os = "macos", target_os = "ios")) {
                "libonnxruntime.dylib".to_owned()
            } else { "libonnxruntime.so".to_owned() }
        }
    }
}
```

—— 先看环境变量 `ORT_DYLIB_PATH`，否则按目标平台挑 `dll` / `dylib` / `so`。这是为了让"探测"和"真正加载"用同一个文件名，避免探测 A、实际找 B。

### `ensure_runtime`：先用 libloading 探测 dylib

```rust
fn ensure_runtime() -> Result<(), PolicyError> {
    static PROBE: OnceLock<Result<(), String>> = OnceLock::new();
    let outcome = PROBE.get_or_init(|| {
        let name = dylib_name();
        match unsafe { libloading::Library::new(&name) } {
            Ok(library) => {
                // Leak it: `ort` will dlopen the same file moments later...
                // 故意泄漏：ort 稍后会再 dlopen 同一个文件，OS 对映射做引用计数，
                // 我们 drop 掉也无害但纯属折腾。
                std::mem::forget(library);
                Ok(())
            }
            Err(e) => Err(e.to_string()),
        }
    });
    outcome.clone().map_err(|detail| PolicyError::RuntimeMissing {
        searched: dylib_name(), detail,
    })
}
```

**为什么需要它**：`ort` 在 dylib 缺失时**不返回错误**，而是在 `setup_api` 内部 `expect` 掉——这个懒初始化路径任何 API 调用都可能触发，于是缺库直接 abort 掉碰到它的那个线程。在控制循环里这意味着线程死掉、永远没有 tick 落地、`robot.health` 永远报"循环尚未完成一个周期"：守护进程看起来卡死了，却不说"ONNX Runtime 没装"。先探测，把"缺库"变成一个普通错误、把运维的修法写进去。

**踩坑点（注释里自己承认的纠正）**：它**不能**阻止 `ort` 随后 panic。曾有一版注释声称可以，被一块跑 ONNX Runtime 1.20.1 的板子证伪了——库加载成功、探测通过，但 `ort` 在自己的版本检查上 panic（`expected version >= '1.23.x', but got '1.20.1'`）。探测只证明"文件能打开"，仅此而已；剩下的由 `catching_ort_panics` 兜底。

`OnceLock` 保证整个进程只探一次。

### `catching_ort_panics`：把 `ort` 的 panic 收编为错误

```rust
fn catching_ort_panics<T>(work: impl FnOnce() -> Result<T, PolicyError>) -> Result<T, PolicyError> {
    std::panic::catch_unwind(std::panic::AssertUnwindSafe(work)).unwrap_or_else(|payload| {
        Err(PolicyError::RuntimePanic { detail: panic_message(payload) })
    })
}
```

**为什么**：`ort` 把某些初始化失败当作不可恢复、直接 panic 而不是返回 `Err`。在控制线程里 panic 比错误更糟——线程死、无 tick、健康状态报一句不点名原因的话，守护进程却还活着、还在服务 socket，于是更新器为一个没人能下手的原因回滚了发布。而 `robotd` 本来就会优雅处理"策略加载失败"：保持姿态、按速率继续 tick、上报原因、被回滚。这个包装让 panic 也走同一条路。

**设计取舍**：

- 只包住 `ort` 的活，**不**包住整个 `Policy::load`，否则我们自己代码的真 bug 也会被误标成"策略不可用"。
- 被捕获的 panic 仍然跑完了 panic hook，所以 backtrace 无论如何都进了 journal。
- 必须用 `AssertUnwindSafe`，因为 `Session` 不是 `UnwindSafe`；这里声辩其安全性：成功后 sessions 被 move 进 `Policy`，失败则被 drop，catch 之后没有人再观察我们的东西。
- **`panic = "abort"` 会让这一切失效**。根 `Cargo.toml` 没有 `[profile.release]`，所以用默认 unwind 策略；哪天加了一个 profile 把 panic 改成 abort，这里会悄悄变回"死掉的控制线程"。这是个隐性地雷。

### `panic_message`：从 panic payload 里掏消息

```rust
fn panic_message(payload: Box<dyn std::any::Any + Send>) -> String {
    if let Some(s) = payload.downcast_ref::<&'static str>() { (*s).to_owned() }
    else if let Some(s) = payload.downcast_ref::<String>() { s.clone() }
    else { "panicked with no message; see the journal for the backtrace".to_owned() }
}
```

—— `panic!` 带字面量产生 `&'static str`，带参数产生 `String`，`ort` 两种都用。都掏不出来时给一句指向 journal 的兜底，而不是空字符串。

### `Net`：枚举七种网络

```rust
pub enum Net {
    Walk,
    Stand,
    /// Commanded sit↔stand: the twist `vx` slot carries a posture flag, 1 = sit, 0 = stand.
    /// 命令式坐↔站：twist 的 vx 槽携带姿态标志，1=坐，0=站。
    SitStand,
    /// Phase-scripted ground pick; the twist slots carry `[cos φ, sin φ, 0]`.
    /// 相位脚本化的地面捡拾；twist 槽携带 [cos φ, sin φ, 0]。
    GroundPick,
    KickLeft,
    KickRight,
    /// Episodic forward roll; trained with every command slot at zero...
    /// 一次性前滚翻；训练时所有命令槽置零，一切进来就立刻开始滚。
    Roulade,
}
```

共七种：走路、站立、坐↔站、捡拾、左踢、右踢、前滚翻。选哪个网络是调用方（`robotd` 的技能调度器）的决定，这个枚举只是它用来点名的手段。点名一个没加载的网络会**回退到走路而不是 panic**——但调度器应先查 `has_*`；这个兜底存在的意义是"竞态也杀不死控制线程"。

### `PolicyPaths`：策略文件路径

```rust
#[derive(Debug, Clone, Default)]
pub struct PolicyPaths {
    pub walk: PathBuf,
    pub stand: Option<PathBuf>,
    pub sitstand: Option<PathBuf>,
    pub ground_pick: Option<PathBuf>,
    pub kick_left: Option<PathBuf>,
    pub kick_right: Option<PathBuf>,
    pub roulade: Option<PathBuf>,
}
```

—— `walk` 是**唯一必填**的；其余全是 `Option`，是 `None` 就代表机器人根本没有这项能力。

### `Policy`：已加载的网络集合

```rust
pub struct Policy {
    walk: Session,
    stand: Option<Session>,
    sitstand: Option<Session>,
    ground_pick: Option<Session>,
    kick_left: Option<Session>,
    kick_right: Option<Session>,
    roulade: Option<Session>,
    standing_threshold: f64,
    standing_disabled: bool,
}
```

设计理由：**配置了路径但加载失败，就让整个加载失败**。策略是随发布一起发的，缺文件/坏文件就是 bundle 坏了，正确结果是"不健康、回滚"，而不是机器人悄悄丢了踢腿能力。

`standing_disabled` 字段：滚轮模式和跌倒恢复模式要**预留**站立网络（滚轮没有站立网络；跌倒恢复要留着它好爬起来），所以命令幅度在这些模式下绝不能自动选中它。

### `Policy::load`：加载、校验、预热

```rust
pub fn load(paths: &PolicyPaths, standing_threshold: f64) -> Result<Self, PolicyError> {
    ensure_runtime()?;
    catching_ort_panics(move || {
        let zero = Observation::zeroed();
        fn open_warm(path: &Path, zero: &Observation) -> Result<Session, PolicyError> {
            let mut session = open(path)?;
            run(&mut session, path, zero)?;   // 用零观测跑一次
            Ok(session)
        }
        fn open_opt(path: &Option<PathBuf>, zero: &Observation)
            -> Result<Option<Session>, PolicyError> {
            path.as_deref().map(|p| open_warm(p, zero)).transpose()
        }
        Ok(Self { walk: open_warm(&paths.walk, &zero)?, stand: open_opt(...)?, ... })
    })
}
```

**预热（warm-up）**：在控制循环第一次调用之前，先用全零观测跑一次推理。第一次推理永远是离群值——懒初始化、冷页、首触缺页——把这些开销付在 tick 1 上，看起来就像控制循环错过了 deadline。而且用 `load-dynamic` 链接时，ONNX Runtime 是否真的可用，不实际跑一次是不知道的。

整段 `ort` 调用（且只有这段）包在 `catching_ort_panics` 里。

### `will_stand` / `has_*`：站不站的判定与能力查询

```rust
pub fn will_stand(&self, twist_magnitude: f64) -> bool {
    self.stand.is_some()
        && !self.standing_disabled
        && twist_magnitude <= self.standing_threshold
}
```

三个条件缺一不可：站立策略**存在**、**未被禁用**、速度幅度**不超阈值**。

它被单独拆出来、与 `infer` 分开，是因为调用方还要用同一个答案去决定增益和动作缩放，问两次必须得到一致结果（不能这次判站、下次判走）。

一组 `has_standing` / `has_sitstand` / `has_ground_pick` / `has_roulade` / `has_kick(left)` 供调度器在点名网络前先查能力。

### `infer`：单次推理，缺失则回退走路

```rust
pub fn infer(&mut self, observation: &Observation, net: Net)
    -> Result<[f32; ACTION_LEN], PolicyError>
{
    let session = match net {
        Net::Walk => None,
        Net::Stand => self.stand.as_mut(),
        ...
    };
    let session = match session {
        Some(session) => session,
        None => &mut self.walk,   // 可选网络缺失 → 回退走路
    };
    run(session, Path::new("<loaded>"), observation)
}
```

调度器理应先查 `has_*` 再点名，真走到回退分支算 bug，但**用错步态也好过控制线程死掉**。

### `open` / `check_width`：打开时校验形状

```rust
fn open(path: &Path) -> Result<Session, PolicyError> {
    let session = Session::builder()
        .and_then(|b| b.with_optimization_level(GraphOptimizationLevel::Level3))
        .and_then(|b| b.with_intra_threads(INTRA_THREADS))
        .and_then(|b| b.commit_from_file(path))
        .map_err(|source| PolicyError::Load { path: path.to_owned(), source })?;
    check_width(path, "observation width", session.inputs(), OBS_LEN)?;
    check_width(path, "action count", session.outputs(), ACTION_LEN)?;
    Ok(session)
}
```

- 优化级别固定 **`GraphOptimizationLevel::Level3`**（全量图优化）。
- 每个 session 用 `INTRA_THREADS=1`。
- 打开即校验输入观测宽度 == `OBS_LEN`、输出动作数 == `ACTION_LEN`。

`check_width` 只看**最后一维**：第一维通常是动态 batch（`-1`），不检查；最后一维才编码着接口契约。张量类型不对、最后一维缺失或不符，都报 `Shape` 错误并把 expected/got 一起带上。

### `run`：执行一次推理

```rust
fn run(session: &mut Session, path: &Path, observation: &Observation)
    -> Result<[f32; ACTION_LEN], PolicyError>
{
    let input = Value::from_array(([1usize, OBS_LEN], observation.as_slice().to_vec()))
        .map_err(|e| PolicyError::Inference(format!("{}: building input: {e}", path.display())))?;
    let outputs = session.run(ort::inputs!["obs" => &input])
        .map_err(|e| PolicyError::Inference(format!("{}: {e}", path.display())))?;
    let value = outputs.values().next()
        .ok_or_else(|| PolicyError::Inference(format!("{}: no output", path.display())))?;
    let (_, data) = value.try_extract_tensor::<f32>()
        .map_err(|e| PolicyError::Inference(format!("{}: extracting output: {e}", path.display())))?;
    if data.len() != ACTION_LEN { /* 报 "N actions, expected ACTION_LEN" */ }
    let mut actions = [0.0f32; ACTION_LEN];
    actions.copy_from_slice(data);
    Ok(actions)
}
```

输入张量名硬编码为 `"obs"`，形状 `[1, OBS_LEN]`；输出取第一个 tensor、抽成 `f32` 并再次校验长度后拷进定长数组。

## 测试要点

测试模块不依赖真实 ONNX Runtime（`load` 被绕过，直接构造判定逻辑），保证"无论装没装 ORT 都能跑"的分支被覆盖：

1. **`the_standing_threshold_matches_the_prototype`**：钉住阈值 == 0.05，否则机器人会在与调参不同的速度上换步态。
2. **`without_a_standing_policy_it_never_stands`**：没有站立策略时绝不选站立；同时覆盖"有策略、零命令→站""有策略、行走命令→不站"。
3. **`disabling_standing_beats_the_magnitude_rule`**：`standing_disabled` 必须压过幅度规则——否则滚轮鸭在摇杆归零瞬间会切到一个为"还站在腿上"训练的网络。
4. **`a_panic_out_of_ort_becomes_an_error_that_keeps_its_message`**（panic 契约）：人为 panic 出一条 Radxa 真实产生的版本不匹配消息，断言它被收编成 `RuntimePanic`，且 `"1.23.x"`、`"1.20.1"` 两个版本号必须活在上报消息里——没有它们运维无从下手。测试会打印一次 panic 与 backtrace 提示，这是**有意为之**（板子上正是靠它把细节写进 journal），不算失败。
5. **`the_catch_is_transparent_when_nothing_panics`**：无事发生时包装必须透明放行，返回值原样穿过——否则"一切正常"的板子也会被误报成"策略不可用"。
6. **`an_unprintable_panic_payload_still_reports_something`**：payload 既非 `&str` 也非 `String`（如 `42u32`）时仍要给出非空原因、并指向 journal。

## 与其他模块的关系

- **依赖 `crate::obs`**：消费 `OBS_LEN`、`ACTION_LEN`、`Observation`。61 维观测布局是所有网络共享的唯一契约，本模块不定义它，只校验网络遵守它。
- **被 `robotd`（crate 外）使用**：`robotd` 持有 `Policy`，用 `Net` 枚举点名网络、用 `will_stand` 决定增益与动作缩放、用 `has_*` 做能力查询。优先级规则归 `robotd`，本模块只负责"按名推理 + 兜底回退"。
- **与 `safety` 的衔接**：`policy` 产出的 `[f32; ACTION_LEN]` 动作交由 `safety::Safety::apply` 做非有限拒绝、范围钳制与写入；本模块不碰电机，也不做安全过滤。
- **运行期依赖 `ort` + 系统动态库 `libonnxruntime.so`**（经 `libloading` 探测）。整个文件的错误设计都围绕"这套外部依赖在控制线程里最坏会怎样"展开。
#（注：内容由AI生成）
