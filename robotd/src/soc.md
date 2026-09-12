# soc.rs 文件解析

## 文件位置

`d:\microduck\robotd\src\soc.rs`

## 核心设计决策

### 1. 故意放在 `robotd` 而非 `duck-control`

板级温度（CPU/GPU/NPU/DDR 热区）与电机总线无关，是 `sysfs` 读取。它必须在电机总线不工作时仍然可用——而那恰恰是最需要知道温度的时候（伺服过载、控制循环卡死）。放在 `duck-control` 会错误地把它绑定到 `RobotIo` 总线语义上。

### 2. 无 `cfg` 门控

不在 Linux 上、或 Linux 内核没有 thermal sysfs 时，函数通过「找不到文件」自然返回 `None`，而不是编译失败。笔记本开发构建会报告「无板级温度」并如实说明，而非拒绝编译。

## 常量

| 名称 | 值 | 含义 |
|---|---|---|
| `THERMAL_ROOT` | `/sys/class/thermal` | 内核暴露所有热传感器的目录 |
| `MILLI_PER_DEGREE` | `1000.0` | 内核 thermal sysfs 以毫摄氏度为单位 |

## 函数

### `hottest_zone_c() -> Option<f64>`

返回板上最热热区的摄氏温度。

设计要点：

- **取所有热区的最大值，而非只取 CPU**：Radxa Zero 3 暴露 `soc-thermal` 与 `gpu-thermal`；其他板还可能有 NPU、DDR 热区。取最热者是「板是否过热」的保守答案，且不会静默漏掉真正在升温的那个区（按名取某一个区会在传感器布线不同时出错）。
- **每次采样都 `read_dir`**：1 Hz 下仅数微秒，缓存路径列表会在驱动加载时出现新热区时失效。
- **丢弃 ≤ 0°C 的读数**：运行中的板读到 0°C 或负值是传感器故障，而非真实温度；跳过而非参与平均。
- **永不返回 `Some(0.0)`**：与电池电压同理，`0` 是「未读到」的哨兵值。

## 单元测试

- `a_reading_is_plausible_or_absent`：在 Linux CI 上必须找到宿主机热区；在 macOS 上返回 `None` 而非 panic。断言的是「形状」而非具体温度（测试无法预知机器温度），温度须落在 `[1.0, 150.0]` °C。
- `millidegrees_convert_to_degrees`：固定 `47123 / 1000 == 47.123`。**毫度→度的换算是本模块最大风险**——忘记除以 1000 会把 47°C 变成 47000，轻松通过任何「是否过热」阈值。

## 关键摘要

`hottest_zone_c` 是无总线依赖的板级温度探针：遍历 `/sys/class/thermal` 全部 `thermal_zone*`，取最高有效值，丢弃故障读数。它在电机总线挂掉时仍然工作，这是它存在于 `robotd` 而非 `duck-control` 的根本理由。
