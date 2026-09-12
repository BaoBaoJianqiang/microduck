# detect.rs 文件解析

## 文件位置

`d:\microduck\pet-detect\src\bin\detect.rs`

## 核心设计决策

`pet-detect` 是**实时挠头检测的命令行前端**：从 stdin 读取 16-bit signed LE 单声道 16 kHz PCM，每次推理在 stdout 输出一行 `<ts_ms>\t<p>\t<state>`。

用法：

```bash
arecord -D plughw:aic3104,0 -f S16_LE -r 16000 -c 1 -t raw | pet-detect --model <onnx>
```

设计要点：

1. **stdin/stdout 纯管道接口**。不直接碰 ALSA，通过 `arecord` 喂数据，便于在开发机上回放录音调试。

2. **概率与状态同时输出**。`p` 是当前推理的正类概率，`state` 是 `petting`/`normal`，方便现场调阈值。

3. **事件走 stderr**。`PettingEvent` 打到 stderr，不污染 stdout 的 TSV 数据流。

## 命令行参数

- `--model`：ONNX 模型路径，默认 `/opt/robot/daemon/current/models/pet_detect.onnx`
- `--stride`：推理窗口间隔（样本数），默认 `WINDOW_SAMPLES/4` ≈ 250 ms
- `--threshold`：进入阈值，默认 0.95
- `--exit-threshold`：退出阈值（滞回），默认 0.85

## 主循环

1. `PettingDetector::new` 加载模型
2. 锁定 stdin，循环读 4096 字节块
3. 奇数字节报错（i16 流必须偶数）
4. 按 2 字节切片转 i16 LE → f32
5. `detector.push_samples` 返回事件序列与最新概率
6. 有推理则打印 `ts_ms \t p \t state`
7. 事件打到 stderr
8. EOF 退出

## 关键摘要

`detect.rs` 是 `PettingDetector` 的最薄 CLI 封装，通过 stdin 管道消费 `arecord` 的原始 PCM，输出可观测的概率/状态时间序列，是调试阈值与验证模型的主要工具。
