# lib.rs 文件解析

**文件位置**：`d:\microduck\robotd-params\src\lib.rs`

## 核心设计决策

`robotd` 的启动参数文件——schema、默认值与验证的唯一真相源。从 `robotd/src/params.rs` 抽成独立 crate，使 `robotctl configure` 能直接用真实类型/默认值/验证编辑文件，而不是维护一个会漂移的副本。`robotd` 把它重新导出为 `crate::params`，自身代码不变。

**关键设计**：
1. **一个文件而非一墙 CLI 标志**——原型曾长到 142 个 flag，多数是变体/死技能/死传感器。
2. **只读一次，不热重载**——任何改动都需重启 robotd，这对工具是负荷性的：编辑器永远不必问哪些 key 是生效的。
3. **位于 `releases/<ver>/` 之外**——使它在更新与回滚中幸存：这是逐机器人配置，不是随发布的默认值。
4. **独立 crate 的唯一消费者是 `robotctl configure`**——`registry` 是每个 key 的机器可读索引，完整性由遍历 `Params` 序列化结果的测试强制。

## 常量

- `RELEASE_DIR = "/opt/robot/daemon/current"`——发布挂载点；策略路径默认在此下。
- `DEFAULT_PATH = "/etc/robot/robotd.toml"`——预置机器人的配置位置。
- `QUALITY_LABELS = ["1080p30","720p30","720p15","360p30"]`——编辑器循环顺序与文件使用的字符串，单一列表使注册表、文件、`Quality` 枚举三者不脱节。
- `CONGESTION_LABELS = ["disabled","homegrown","gcc"]`——`webrtcsink` 自身的属性昵称，注意是 `gcc` 不是 `googcc`。
- `BITRATE_MIN = 100_000`、`BITRATE_MAX = 20_000_000`——`media.bitrate` 的接受带宽（bits/s）。下限不是品味：`bitrate = 2000` 是把千比特写成比特的笔误，2 kb/s 流根本出不了画面。

## 类型

### `Params`（顶层 struct）
10 个 section：`bus`、`control`、`update_gate`、`policy`、`safety`、`audio`、`theremin`、`chorale`、`media`、`detect`。全部 `deny_unknown_fields` + `default`。

### `Quality` 枚举
视频模式作为一个名称而非四个数字。**帧尺寸、帧率、匹配码率要么一起动要么不动**——四把独立的 key 会让每种错误组合都可表达（含采集路径根本产不出的组合），而启动失败的 pipeline 会连带 WebRTC 控制通道一起没。
- 四档：`1080p30`（满帧，最可能跑不到 30fps）、`720p30`（默认，`mediad` 所有测量值所在档）、`720p15`（链路扛不住 30 时的半帧率）、`360p30`（小而省）。
- 每档都是 16:9（传感器自身宽高比），每维都是 8 的倍数（ISP scaler 与编码器宏块都想要）。
- `default_bitrate` 按像素速率缩放：720p30=2 Mb/s（`mediad` 一直用的实测值），其余按像素份额等比，人工取整。
- 1080p30 不做缩放（传感器已钉在 1920x1080 模式），未测量的是采集路径与编码器能否在 2.25× 像素下保住 30 fps。

### `CongestionControl` 枚举
`mediad` 决定实际发送码率的方式。**这既是网络设置也是 CPU 设置**：`rtpgccbwe` 是进程内最大的单核消耗者（7.6% 对 `v4l2src` 的 0.3%），因为它按包工作而采集按 DMABuf 句柄工作。
- `Disabled`：不适应，`bitrate` 即固定码率。
- `Homegrown`：`webrtcsink` 自家启发式，比估计器便宜但更钝。
- `Gcc`（默认）：Google Congestion Control，`webrtcsink` 自己的默认——命名它是为了当插件默认改变时不连带改变每台机器人的发送速率。

### `MediaParams` `[media]`
`camera`、`quality`、`bitrate`（可选，未设跟随 quality）、`congestion_control`。原先是 `mediad.service` 的 CLI flag，发布安装器重写该 unit 文件，唯一支持改 flag 的方式是 systemd drop-in——把它们放进 `robotctl configure` 已编辑的文件是本 section 存在的理由。改后需 `systemctl restart mediad`（不是 robotd）。

### `DetectParams` `[detect]`
**由 mediad 读取而非 robotd**（首例）——帧在 mediad 的 tee 上，感知应挨着传感器。仍放在此文件因为一个机器人只有一处开关集合。
- `enabled`：默认关。检测器要消耗发布里的一个模型、~50ms CPU/帧与一些热量。
- `model`：`.rknn` 走 NPU、`.onnx` 走 CPU；缺失用发布自带模型，NPU 优先。
- `hz`：默认 2.0。**2Hz 是散热数字不是偏好**——Radxa Zero 3 满速达 95°C、CPU 节流到 408MHz。
- `threshold`：0.35。量化模型输出张量自带 scale，0.9 在此不指 0.9——发布模型的 INT8 分数饱和在约 1.4，在板上调校而非从训练继承。
- `models()`：返回尝试列表（最优在前）。是列表而非单选，因为 NPU 是否可用本文件无法知道；NPU 不可用时落到 `.onnx` 让板子仍能看见，而不是记一条警告就永远不做事。显式 `model` 则只试它。

### `ChoraleParams` `[chorale]`
`accept` 默认 **false**——这是整个 section 的核心。合唱不仅发声还动嘴和头：机器人因别的机器人进房而开始动作，是在别人客厅里做没人要求的动作。关闭也意味着**不可见**（不上广播），而非可见地拒绝。

### `ThereminParams` `[theremin]`
ToF 特雷门琴配置。关键是 `statuses`——ST 文档把 5 和 9 标为"range valid"，但只信这两个会在约 30cm 后看不见手（移动的手返回 4 或 13 的 *consistency failed*，距离对音高完全可用）。默认值来自 `kinematics::hand::Config::default()`。
- `hold_ms`：保持时长，防闪烁区把音符切成碎石。

### `AudioParams` `[audio]`
`enabled`、`device`（`plughw:aic3104`，TLV320AIC3104 codec）、`bank`（`/var/lib/robot/sounds`，由 `sounds ensure-bank` 从 SoC 序列号渲染）、`greet`（启动叫一声，无头板上这是"robotd 在跑"的听觉信号）、`pet_detect`（`Option<bool>`，默认关——原型按模式启用导致每次偶然摸头都咕咕叫，很烦人）、`pet_model`（`"none"` 字面量禁用）、`pet_enter_threshold=0.95`/`pet_exit_threshold=0.85`（滞回）。
- `capture_device()`：在播放设备后加子设备 0，但若已含逗号则原样返回——否则 `plughw:aic3104,0` 会变成 `plughw:aic3104,0,0`，没有卡应答，worker 陷入重启循环。
- `pet_detect_resolved(_mode)`：始终关，除非显式请求。保留 mode 参数以便一行改回。

### `Mode` 枚举
`walk`（默认）或 `roller`。同一机器人两种人格：腿或滚球。改变策略与调优，所以是一个开关而非六条操作者需保持一致的路径。切换 = 编辑 + 重启 robotd。

### `PolicyParams` `[policy]`
策略加载与调优。`enabled=false` 即 slice 1 行为：跑循环、保持位姿、保持健康——与"想要策略但加载失败"不同，后者是不健康。
- 7 个策略槽：`walk`、`stand`、`sitstand`、`ground_pick`、`kick_left`、`kick_right`、`roulade`——均 `Option<PathBuf>`，缺失走发布目录默认，`"none"` 字面量禁用该槽。
- 调优：`action_scale`（未设按模式：walk 0.9 / roller 0.8）、`standing_action_scale=1.0`、`standing_gain_ratio=0.8`、`gain=200`、`head_lowpass=0.5`（训练时的值，必须匹配否则迁移退化）、`legs_lowpass=0.7`、`ground_pick_period`（walk 4.0s / roller 3.0s）、`ground_pick_action_scale`、`ground_pick_gain_ratio=1.0`、`kick_duration=0.5`、`roulade_duration=1.0`、`roulade_action_scale=1.0`、`roulade_gain_ratio=1.0`、`voltage_adapt=false`、`nominal_voltage=7.4`。

### `ResolvedPolicy`
`policy.resolved()` 的产物——所有缺失字段按模式默认填好，下游无需再问"walk 还是 roller"。

### `SafetyParams` `[safety]`
- `fall_gravity_z=-0.5`：投影重力 z 高于此即视为倒下（直立约 -1.0，侧躺近 0）。
- `fall_debounce_ms=200`：去抖，避免重脚步被判为跌倒。
- `deadman_ms=500`：意图超龄后速度归零（停止，不是瘫软）。
- `gain_limp=50`：limp-fall 时降到的增益。
- `battery_empty_shutdown=true`：电池 EMA 到达 6.6V 时坐下关机，EMA ~10s 不会被负载骤降触发。
- `limp_fall=true`（默认开，因为已在机器人上验证）：跌倒时瘫软→软着地→摆回站立位→交给站立策略。
- `limp_fall_tilt_z=-0.90`（约 26° 倾斜，正常走路达不到）、`limp_fall_predict_z=-0.5`、`limp_fall_lookahead_ms=300`、`limp_fall_debounce_ms=60`（3 tick，长于脚步脉冲、短于留给瘫软的大部分下落）、`limp_fall_still_rate=1.0 rad/s`、`limp_fall_still_ms=200`、`limp_fall_max_ms=1500`（硬上限，被人抱着的机器人不会永远瘫软）、`limp_fall_pose_ms=600`（摆回时长）、`limp_fall_pose_gain=160`（摆回增益，软站立而非瘫软）。

### `Bus` `[bus]`
`port`：`/dev/ttyS2`（Radxa Zero 3W 把舵机与 IMU 板接到此）。

### `Control` `[control]`
- `hz=50`：控制循环率（原型在 Pi Zero 2W 上选的，应在 Radxa 上重推）。
- `cmd_alpha=0.2`、`head_alpha=0.2`：速度命令与头部目标的逐 tick EMA，原型的 `--cmd-alpha`。

### `UpdateGate` `[update_gate]`（原 `[health]`，彻底改名不设别名）
决定 `healthy` 的阈值——即更新是否被保留。**不是 `robot.health` 报告的一切**（电池/电机温度/循环计数都不得进入 verdict）。
- `min_achieved_hz=45.0`：低于此即不健康（目标率的 90%）。
- `stall_periods=25`（500ms）：无 tick 的周期数超过此即视为**卡死**。刻意远离周期——3 周期=60ms 会被调度抖动触发而回滚好版本。
- `max_consecutive_errors=10`：连续总线读失败容忍数。

### `ParamsError`
`Read`、`Parse`、`Rate`（control.hz 须在 1..1000）、`Bitrate`（须在 BITRATE_MIN..MAX，单位是 bits）。

## 函数

- `is_none_sentinel(path)`：判断路径是否等于 `"none"`（忽略大小写）——禁用可选策略槽的字面量。
- `Params::load(path, explicit)`：
  - 默认路径文件缺失不报错（未预置板应起在默认值上），显式指定则必须存在。
  - **严格解析优先**：本构建完全理解的文件一遍过 serde。
  - 失败时走 `without_unknown_keys` 宽松路径：丢掉本构建不认识的 key 并在 `warn` 命名它们，剩余部分仍解析失败则报真实错误（代价是丢失位置信息，但比把刚被宣布无害的 key 报成启动失败原因好）。
  - 然后 `validate`。
- `Params::validate`：拒绝 0 或 >1000 的 hz；拒绝越界 bitrate（在此检查而非 mediad，让 `robotctl configure` 直接拒绝写）。
- `Params::period()`：`1/hz`。
- `without_unknown_keys(text)`：
  - 用 registry 判断 section/key 是否存在；未知 section 作为整体报告（`[chorale]` 是一个决定），未知 key 单独报告。
  - 返回 `Some((reparsed, ignored))` 或 `None`（没有未知 key，调用方应报原始严格错误）。
  - 设计理由：未知 key 曾是致命的，但一个跑分支的机器人有 `[chorale]`，更新到无此功能的 main 会导致 robotd 起不来、连续 4 次回滚——为一段什么都不做的配置付出过大代价。
  - `deny_unknown_fields` 仍保留在 struct 上——是 registry 完整性测试的基础，也是 backstop。
- `PolicyParams::resolved()`：按 mode 解析所有 `Option` 字段为 `ResolvedPolicy`。walk 与 roller 的差异：策略文件名不同、`stand` 在 roller 不加载（原型加载了但跳过所有站立转换，所以从不运行——不加载是同一机器人但少了死 session）、`action_scale`/`ground_pick_period`/`ground_pick_action_scale` 不同。
- `MediaParams::bitrate_resolved()`：`bitrate.unwrap_or(quality.default_bitrate())`。
- `AudioParams::capture_device()`、`pet_model_resolved()`、`pet_detect_resolved()`。
- `ThereminParams::hand()`：转为 `kinematics::hand::Config`。

## 单元测试

覆盖：捕获设备不重复子设备；缺失默认文件回退默认值；显式指定缺失文件报错；缺失 section 走默认；quality 标签与枚举双向 round-trip；congestion 标签 round-trip 且默认 gcc；未设 bitrate 跟随 quality；千比特 bitrate 被拒绝；默认值等于 mediad.section 存在前的 flag 默认；`deploy/robotd.toml` 与内置默认一致；walk 模式解析为原型 alpha 配置；命令平滑匹配原型 `--cmd-alpha`/`--head-alpha`；roller 模式一行 `mode="roller"` 复现原型 roller preset；`"none"` 与 `1.0` 是关闭开关；typo 被忽略并命名且真实 key 不动；旧 `[health]` section 按名忽略；其他分支的 section 不阻止机器人启动；错误类型值仍被拒绝且带位置；未知 section 旁的真实错误命名真实错误而非 section；语法错误仍是语法错误；干净文件无报告；不可能的率在启动时拒绝。

## 关键摘要

这是整个系统的配置单一真相源。两大工程成就：(1) `registry` + 遍历 serde 序列化的测试让编辑器不可能静默不知道某个 key；(2) 未知 key 从致命改为 `warn` 命名+忽略，避免跨分支更新时机器人因一段无害配置起不来。每节默认值都明确标注来自原型实测值，`deploy/robotd.toml` 由测试钉死与默认一致。
