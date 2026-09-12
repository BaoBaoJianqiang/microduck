# worker.rs 文件解析

## 文件位置

`d:\microduck\pet-detect\src\worker.rs`

## 核心设计决策

`worker` 模块把麦克风变成后台工作线程：`arecord` 子进程 → 宠物检测分类器 + 环境音哨兵。一切分析麦克风的逻辑共享这一条采集流。

核心设计要点：

1. **用 `arecord` 子进程而非进程内 ALSA 绑定**。不引入新原生依赖，与独立 `pet-detect` 二进制相同模式；采集设备是单客户端的，所以所有分析必须共享这一条流。

2. **分类器在调用者线程构建，不在线程内**。模型或 ONNX Runtime 缺失的错误必须让调用者看到，而不是 worker 静默死亡。构建包在 `catch_unwind` 里——`ort` 在不可恢复失败时 panic，缺少 libonnxruntime 应读作"无麦克风 worker"，而不是守护进程崩溃。

3. **环境音哨兵是纯 RMS 启发式，不用 ML**。区分短暂瞬态（`Noise`：拍手/砰/关门，≤~0.38 s）与持续发声（`Voice`：说话/对鸭叫，~3 s 内）；更长的连续噪声（吸尘器/音乐）被自适应底噪吸收，不产生事件。

4. **哨兵在挠头期间抑制**。头麦对挠头非常响（接近接触麦），所以分类器报告 petting 时哨兵抑制事件，外加 1 秒余辉。petting 状态从 `detector.is_petting()` 读取而非本地跟踪——因为 `arecord` 可能在一次 Start..End 会话内重启，分类器由 `worker_loop` 持有能跨重启存活，本地副本会回到 false 导致哨兵把挠头噪声当成 `Voice`。

5. **重启退避是真问题**。原代码只在 `arecord` 二进制缺失时睡眠；但"arecord 存在但 codec 不存在"（DKMS 构建失败的软失败）会让 arecord 每次 spawn 立即退出，worker 会以 CPU 全速 fork/exec 直到守护进程结束。改为指数退避（250 ms 起，30 s 封顶），连续 5 次失败后降级为 debug 日志。

## 常量分析

- `SENTRY_FRAME = 512`：哨兵帧长（32 ms @16 kHz）
- `RESTART_BACKOFF_MIN = 250 ms` / `RESTART_BACKOFF_MAX = 30 s`：重启退避范围
- `RESTART_HEALTHY = 5 s`：采集存活超过此时间视为健康，退避重置
- `RESTART_QUIET_AFTER = 5`：连续失败超过此次数后日志降为 debug

## 类型与函数分析

### `SoundEvent`

`{ Noise, Voice }`，环境音事件。

### `SoundSentry`

RMS 包络哨兵。状态字段：`floor`（自适应底噪，τ≈6 s）、`frame_acc`/`frame_n`（帧累加）、`in_event`/`event_frames`/`quiet_frames`（事件状态机）、`cooldown_frames`（事件间冷却 ≥1 s）、`petting_hold_frames`（挠头抑制余辉 31 帧≈1 s）、`event_peak`（调参日志用）、`floor_log_frames`（每 60 s 打印底噪）。

`push` 逐样本累加方和，凑满 `SENTRY_FRAME` 就计算 RMS 调 `frame`。`frame` 中：
- 非事件帧以 `0.995*floor + 0.005*rms` 缓慢抬升底噪
- `on_thresh = max(floor*6, 0.002)` / `off_thresh = max(floor*3, 0.0012)`——绝对下限防止近零底噪的 6 倍在静音中误触发
- 事件超过 94 帧（~3 s）视为连续噪声，吸收进底噪不发事件
- 事件结束按持续帧数分 `Noise`（≤12 帧≈0.38 s）或 `Voice`

### `PetConfig`

`alsa_device`（默认 `plughw:aic3104,0`）、`model_path`（默认 `/opt/robot/daemon/current/models/pet_detect.onnx`）、`enter_threshold`、`exit_threshold`。

### `PetHandle`

worker 句柄。`spawn` 在调用线程构建分类器（带 panic 捕获），spawn `pet-worker` 线程，返回带两个 `mpsc` 接收端的句柄。`try_recv_event`/`try_recv_sound` 非阻塞取事件（每控制 tick 一次）。`shutdown` 设标志并 join 线程。

### `worker_loop`

主循环：spawn arecord → `pump` 读取并喂给分类器+哨兵 → 失败则退避。存活超过 5 s 重置失败计数。退避睡眠被切成 100 ms 片，使 `shutdown()` 能快速 join（否则 30 s 睡眠=30 s 退不出）。

### `spawn_arecord`

`arecord -D <device> -f S16_LE -r 16000 -c 1 -t raw`，stdout 管道化，stderr 丢弃。

### `pump`

从 arecord stdout 读 4096 字节块，按 2 字节切片转 i16 LE → f32 → `detector.push_samples` → 发送 petting 事件 → 喂哨兵发 sound 事件。奇数字节视为错误（i16 流不可能奇数）。EOF 返回错误触发重启。

## 关键摘要

`worker.rs` 把单客户端麦克风变成可靠的后台事件源：arecord 子进程产出 PCM，分类器做挠头检测（带滞回），同一流上跑 RMS 哨兵产出 Noise/Voice 事件（挠头期间抑制）。重点在健壮性——分类器构建带 panic 捕获、arecord 重启带指数退避（防止无 codec 时 CPU 跑满）、退避睡眠分片使 shutdown 能及时 join。
