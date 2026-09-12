# train.py 文件解析

## 文件位置

`d:\microduck\pet-detect\training\train.py`

## 核心设计决策

`train.py` 是挠头分类器的训练脚本（PyTorch），**最关键的设计是特征提取通过 `pet-features` Rust 二进制完成**，而非在 Python 中重新实现 log-mel。这保证训练特征与机器人运行时逐字节一致，无 Python/Rust 实现漂移。

运行方式（PEP 723 inline script metadata）：

```bash
cargo build --release -p pet-detect --bin pet-features
uv run training/train.py
```

## 常量与路径

- `N_MELS = 40`、`WINDOW_FRAMES = 100`：必须与 `src/lib.rs` 一致
- `BLOCK_BYTES = 40 * 100 * 4 = 16000`：每个窗口的 f32 LE 字节数
- `FEATURES_BIN = <repo>/../target/release/pet-features`：Rust 二进制路径
- `DATA = <repo>/data`、`MODEL_OUT = <repo>/models/pet_detect.onnx`

## 函数分析

### `extract_blocks(wav)`

调用 `pet-features <wav>`，捕获 stdout，校验字节数能被 `BLOCK_BYTES` 整除，reshape 为 `[N, N_MELS, WINDOW_FRAMES]` 的 float32 数组。

### `load_class(label_dir, label)`

读取 `label_dir` 下所有 `*.wav`，对每个调用 `extract_blocks`，拼接为 `X`，生成对应标签 `y`。打印窗口数与文件数。

### `TinyAudioCNN`

模型结构（约 20 KB）：

```
Conv2d(1→8, 3×3, pad=1) → BN → ReLU → MaxPool(2)
Conv2d(8→16, 3×3, pad=1) → BN → ReLU → AdaptiveAvgPool2d(1)
Flatten → Linear(16→2) → softmax(dim=1)
```

`forward` 直接输出 softmax 概率。

### `main`

1. 检查 `pet-features` 二进制存在
2. 加载 `data/normal`（标签 0）与 `data/petting`（标签 1）
3. 拼接并加 channel 维 → `[N, 1, 40, 100]`
4. 固定种子 42 打乱，80/20 划分
5. Adam（lr=1e-3）训练 1000 epoch，batch=32
6. 损失：`nll_loss(log(probs+1e-8))`（因 forward 已 softmax）
7. 每 epoch 打印训练损失与验证准确率
8. 导出 ONNX（opset 18，动态 batch 维）
9. 若 dynamo 导出器生成了 `.onnx.data` 侧车文件，用 `onnx` 库合并回单一 `.onnx` 并删除侧车，使运行时只需一个工件

## 关键设计说明

- **特征不归一化**。注释解释：把 mean/std 归一化写进 ONNX 的 Sub/Div 是过度工程；既然 `features.rs` 与 `detect.rs` 共享 `lib.rs`，归一化只在训练内做，模型会通过 BatchNorm 偏置吸收。保持特征未归一化可避免训练/推理漂移。
- **单一 ONNX 工件**。新 dynamo 导出器默认把权重写到 `.onnx.data` 侧车；脚本用 `onnx.load(..., load_external_data=True)` + `save_model(..., save_as_external_data=False)` 合并，保证发布只有一个 `pet_detect.onnx`。

## 关键摘要

`train.py` 训练一个约 20 KB 的二分类 CNN 检测挠头。核心原则是训练特征由 Rust `pet-features` 二进制提取，确保与推理侧逐字节一致。训练 1000 epoch 后导出单一 ONNX 文件（合并侧车权重），交付为 `models/pet_detect.onnx`。
