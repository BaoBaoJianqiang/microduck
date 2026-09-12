# README.md 文件解析

## 文件位置

`d:\microduck\pet-detect\README.md`

## 项目概述

`pet-detect` 是一个**微型音频分类器**，通过板载麦克风（位于头部）检测机器人头部是否被挠。模型是约 20 KB 的 CNN，在 40 频带 log-mel 窗口上推理，亚毫秒级。

移植自 `apirrone/microduck_pet_detect`（相同特征、相同模型、相同阈值）；原型 `pet_worker.rs` 中的 arecord worker 与环境音哨兵也合入本 crate，使所有监听麦克风的逻辑集中在一处。`robotd` 运行此 worker（见 `deploy/robotd.toml` 的 `[audio]`），挠头开始时会发出咕咕声。

## 组成

- **库**：`PettingDetector`（流式、带滞回）与 `worker::PetHandle`（arecord 子进程 + 事件通道）
- **`pet-detect` 二进制**：把 `arecord ... | pet-detect --model <onnx>` 管道接进来，边挠头边观察概率
- **`pet-features` 二进制**：导出 WAV 的 log-mel 特征——训练/推理一致性契约的训练侧
- **`models/pet_detect.onnx`**：训练好的模型，随发布作为 `models/pet_detect.onnx` 交付

## 重新训练

`training/train.py`（PyTorch）**通过 `pet-features` 二进制提取特征**——与机器人运行同一段代码，无训练/推理漂移。在机器人上录数据：

```bash
arecord -D plughw:aic3104,0 -f S16_LE -r 16000 -c 1 -d 30 /tmp/petting_01.wav
```

放入 `data/petting/` 和 `data/normal/`（走路、电机、环境——任何非挠头声音；录音本身不入库），然后：

```bash
cargo build --release -p pet-detect --bin pet-features
uv run --with torch --with onnx training/train.py
```

提交刷新后的 `models/pet_detect.onnx`。

## 关键摘要

README 说明了 `pet-detect` 的定位（头部挠痒检测）、组成（库 + 两个二进制 + onnx 模型）以及重新训练流程。最重要的设计原则是**训练/推理一致性**：训练脚本不自己实现 mel，而是调用 `pet-features` 二进制复用 `lib.rs` 的提取路径，杜绝 Python/Rust mel 实现差异。
