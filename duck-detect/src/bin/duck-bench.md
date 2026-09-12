# duck-bench.rs 文件解析

## 文件位置

`d:\microduck\duck-detect\src\bin\duck-bench.rs`

## 核心设计决策

`duck-bench` 回答三个问题，按重要性排序：

1. **它能运行吗？** NPU runtime 加载失败、为其他平台构建的模型、或驱动旧于 runtime，都在此失败而非在守护进程中失败。
2. **它还能看到鸭子吗？** 量化是检测器停止工作的地方，一个运行但什么都检测不到的模型与一个正常工作的模型看起来完全一样。所以报告每帧检测数，而非仅毫秒数。
3. **它代价多少？** 延迟百分位和 CPU 消耗——因为把它放 NPU 上的理由是不打扰 `robotd` 的 50Hz 循环，"NPU 做的"是需检验而非假设的断言。

读 JPEG 而非打开摄像头：`mediad` 持有摄像头，采集会话的帧已是正确的东西，而需要停守护进程的基准没人会跑第二次。

## 命令行参数（`Args`）

| 参数 | 默认 | 说明 |
|---|---|---|
| `--model` | （必填） | 量化模型 `.rknn` |
| `--frames` | （必填） | JPEG 目录 |
| `--warmup` | 5 | 跳过此数量计时运行，避免首次调用开销 |
| `--passes` | 3 | 整组运行次数，小会话上求稳定百分位 |
| `--threshold` | 0.35 | 检测阈值。**针对量化模型调**：其输出张量自带 scale，float 模型的 0.5 不是此模型的 0.5 |
| `--hz` | 2.0 | 每秒推理数。**默认限速，非礼貌**：全速跑把 Radxa Zero 3 带到 95°C，CPU 节流到 408MHz，全速跑报告的是已过热板子的数字。2Hz 是检测器实际运行速率。`--hz 0` 取消限速 |
| `--verbose` | false | 每帧打印一行 |

## 函数分析

### `cpu_seconds() -> Result<f64>`
本进程已用 CPU 秒数，从 `/proc/self/stat` 读 user+system。实测而非用工具采样，数字属于本进程。`comm` 字段可含空格和括号，从最后一个 `)` 后计数字段，utime/stime 是完整行的第 14/15 字段（comm 后第 11/12）。

### `libc_sysconf_clk_tck() -> i64`
`sysconf(_SC_CLK_TCK)`，不引入 libc 依赖只为一个常量（此处都是 100），但询问而非假设。`_SC_CLK_TCK` 在 Linux 上是 2。

### `soc_temperature() -> Option<f64>`
SoC 温度（°C），扫描 `/sys/class/thermal/thermal_zone{0..8}` 找 `soc-thermal`。报告是因为它是结果：检测器 2Hz 时便宜，全速时烤板，只测速度的基准会说后半段免费。

### `percentile(sorted, fraction) -> Duration`
从已排序数组取百分位。

### `jpegs(directory) -> Result<Vec<PathBuf>>`
列出目录中 `.jpg`/`.jpeg` 文件并排序。

### `main() -> Result<()>`
1. 初始化 tracing
2. 解析参数，列出 JPEG
3. 打开模型，打印 runtime/driver 版本、模型形状、输出数、帧数/轮数
4. 校验正方形 RGB 模型
5. **预先解码所有 JPEG**（JPEG 解码不是被测对象，放进计时循环会掩盖推理）
6. 预热：对第一帧做 letterbox + infer + decode
7. 主循环：`passes` 轮，每轮每帧：
   - 限速（在工作前睡，慢推理吃掉自己的时隙而非推后整个运行——与控制循环 tick 同形）
   - letterbox RGB
   - 计时 infer + decode
   - 第一轮统计检测数、有鸭子的帧数、verbose 输出
8. 报告：
   - 延迟 p50/p95/p99/max（ms）
   - 限速速率 或 全速吞吐量
   - 每帧 CPU ms、占单核百分比（在限速速率下的代价，即与 50Hz 控制循环共存的关键数字）
   - SoC 温度
   - 总检测数、有鸭子的帧数
   - 若零检测：提示可能是量化模型阈值不对，先试 `--threshold 0.2`

## 关键摘要

`duck-bench` 是检测器的真实板基准工具。回答三问：能运行、能看到鸭子、代价多少。关键设计：限速（默认 2Hz，避免过热节流导致数字失真）、预解码 JPEG 排除其开销、报告检测数（量化易让检测器静默失效）、CPU 每帧 ms + 单核占比（判断能否与 50Hz 控制循环共存）。零检测时提示先调阈值而非判定模型坏。
