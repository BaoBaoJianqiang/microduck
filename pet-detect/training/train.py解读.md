# `train.py` 解读

## 概述

宠物分类器的训练脚本。用 `uv run` 直接运行（脚本内联依赖声明）。

### 核心设计决策：特征提取由 Rust 二进制完成

> Features are extracted by the `pet-features` Rust binary so that the training features exactly match what the runtime computes — no risk of a Python/Rust log-mel discrepancy biasing the model.

特征由 `pet-features` Rust 二进制提取，使训练特征与运行时计算完全匹配——没有 Python/Rust log-mel 差异偏差模型的风险。

**为什么这很重要**：如果训练时用 Python 的 librosa 提取特征，推理时用 Rust 手写 log-mel，两者可能在浮点精度、窗函数、mel 滤波器设计上有微小差异。这些差异在训练时不存在，但在推理时会系统性偏移模型输入，降低准确率。用同一个 Rust 二进制提取训练特征，消除了这种风险。

---

## 运行方式

```bash
# 从仓库根目录运行：
cargo build --release -p pet-detect --bin pet-features
uv run training/train.py
```

脚本头部的 PEP 723 内联依赖声明：

```python
# /// script
# requires-python = ">=3.10"
# dependencies = [
#   "torch>=2.2",
#   "numpy",
#   "onnx",
#   "onnxscript",
# ]
# ///
```

`uv run --script` 自动创建临时虚拟环境并安装依赖，不需要手动管理。

---

## 路径常量

```python
REPO = Path(__file__).resolve().parent.parent          # pet-detect/
DATA = REPO / "data"                                    # data/
MODEL_OUT = REPO / "models" / "pet_detect.onnx"         # models/pet_detect.onnx
FEATURES_BIN = REPO.parent / "target" / "release" / "pet-features"  # ../target/release/pet-features
```

### 特征参数（必须与 src/lib.rs 一致）

```python
# Must match src/lib.rs.
N_MELS = 40
WINDOW_FRAMES = 100
BLOCK_BYTES = N_MELS * WINDOW_FRAMES * 4  # f32 LE = 16,000 bytes per window
```

**注释明确说"必须与 src/lib.rs 一致"**。如果 Rust 端改了特征参数，这里必须同步改。这是训练契约的另一半——Rust 端有测试固定，Python 端靠注释和 BLOCK_BYTES 校验。

---

## extract_blocks

```python
def extract_blocks(wav: Path) -> np.ndarray:
    """Run the Rust pet-features bin on `wav`, return [N, N_MELS, WINDOW_FRAMES] f32."""
    res = subprocess.run([str(FEATURES_BIN), str(wav)], capture_output=True, check=True)
    raw = res.stdout
    if len(raw) % BLOCK_BYTES != 0:
        raise RuntimeError(f"{wav}: feature byte count {len(raw)} not divisible by {BLOCK_BYTES}")
    n = len(raw) // BLOCK_BYTES
    arr = np.frombuffer(raw, dtype=np.float32).reshape(n, N_MELS, WINDOW_FRAMES).copy()
    return arr
```

### 流程

1. 运行 `pet-features <wav>`，Rust 二进制输出原始 f32 LE 特征到 stdout
2. 校验字节数是 BLOCK_BYTES 的整数倍
3. 用 `np.frombuffer` 零拷贝解释为 float32
4. reshape 为 `[N, N_MELS, WINDOW_FRAMES]`
5. `.copy()` 因为 frombuffer 返回只读数组

### 为什么用 stdout 而非文件

- 避免临时文件管理
- 二进制直接输出二进制，无文本编码开销
- `check=True` 让非零退出码抛异常

---

## load_class

```python
def load_class(label_dir: Path, label: int) -> tuple[np.ndarray, np.ndarray]:
    wavs = sorted(label_dir.glob("*.wav"))
    feats = [extract_blocks(w) for w in wavs]
    X = np.concatenate(feats, axis=0)
    y = np.full(X.shape[0], label, dtype=np.int64)
    return X, y
```

从 `data/normal/`（label=0）和 `data/petting/`（label=1）加载 WAV 文件，提取特征，拼接成数据集。

---

## TinyAudioCNN

```python
class TinyAudioCNN(nn.Module):
    def __init__(self) -> None:
        super().__init__()
        self.net = nn.Sequential(
            nn.Conv2d(1, 8, kernel_size=3, padding=1),
            nn.BatchNorm2d(8),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(8, 16, kernel_size=3, padding=1),
            nn.BatchNorm2d(16),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1),
            nn.Flatten(),
            nn.Linear(16, 2),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return torch.softmax(self.net(x), dim=1)
```

### 架构

| 层 | 输出形状 | 说明 |
|----|----------|------|
| Input | [B, 1, 40, 100] | log-mel 频谱图 |
| Conv2d(1→8, 3×3) | [B, 8, 40, 100] | 特征提取 |
| BatchNorm2d(8) | [B, 8, 40, 100] | 归一化 |
| ReLU | [B, 8, 40, 100] | 激活 |
| MaxPool2d(2) | [B, 8, 20, 50] | 下采样 |
| Conv2d(8→16, 3×3) | [B, 16, 20, 50] | 更深特征 |
| BatchNorm2d(16) | [B, 16, 20, 50] | 归一化 |
| ReLU | [B, 16, 20, 50] | 激活 |
| AdaptiveAvgPool2d(1) | [B, 16, 1, 1] | 全局平均池化 |
| Flatten | [B, 16] | 展平 |
| Linear(16→2) | [B, 2] | 分类 |
| Softmax | [B, 2] | 概率 |

### 为什么这么小

- 模型只有约 20 KB（8×3×3 + 16×8×3×3 + 16×2 ≈ 1,300 参数）
- 运行在 CPU 上（Radxa Zero 3W），不需要 GPU
- 任务简单（二分类：petting vs normal），小模型足够
- BatchNorm 层在推理时折叠为固定缩放/偏置，不增加运行时开销

---

## 训练流程

### 数据划分

```python
rng = np.random.default_rng(42)
idx = rng.permutation(X.shape[0])
X, y = X[idx], y[idx]
split = int(0.8 * X.shape[0])
Xtr, ytr = X[:split], y[:split]
Xva, yva = X[split:], y[split:]
```

固定随机种子 42，80/20 划分。

### 不做特征归一化

> Per-dataset mean/std normalization. Stored in the ONNX as a Sub/Div is overkill; since features.rs and detect.rs share lib.rs, we normalize inside training only — the model will learn batch-norm offsets to absorb it. Keep features unnormalized to avoid train/infer drift.

**不在训练时做特征归一化**：
- 把 mean/std 存到 ONNX 中作为 Sub/Div 层是过度设计
- BatchNorm 层会学习偏移来吸收分布差异
- 保持特征不归一化，避免训练/推理漂移

### 优化器和训练

```python
opt = torch.optim.Adam(model.parameters(), lr=1e-3)
# ...
for epoch in range(1000):
    # 训练循环
    # 验证
```

- Adam 优化器，学习率 1e-3
- 最多 1000 epoch（没有 early stopping，训练完为止）
- batch size 32

### 损失函数

```python
# CrossEntropy expects logits; our forward applies softmax, so use NLL on log(probs).
loss = nn.functional.nll_loss(torch.log(probs + 1e-8), yb)
```

因为 forward 已经应用了 softmax，不能直接用 CrossEntropyLoss（它期望 logits）。用 NLL loss on log(probs) 等效。`1e-8` 防止 log(0)。

---

## ONNX 导出

```python
torch.onnx.export(
    model,
    dummy,
    MODEL_OUT,
    input_names=["input"],
    output_names=["probs"],
    dynamic_axes={"input": {0: "batch"}, "probs": {0: "batch"}},
    opset_version=18,
)
```

- 输入名 `input`，输出名 `probs`
- batch 维度动态（训练时 batch=32，推理时 batch=1）
- opset 18

### 合并 sidecar 文件

> The new dynamo exporter writes weights to a sidecar `.onnx.data` file by default. Consolidate everything back into a single .onnx so the runtime only needs one artifact.

新版 PyTorch dynamo 导出器默认把权重写到 sidecar `.onnx.data` 文件。需要合并回单个 `.onnx`，让运行时只需要一个文件：

```python
sidecar = MODEL_OUT.with_suffix(".onnx.data")
if sidecar.exists():
    m = onnx.load(str(MODEL_OUT), load_external_data=True)
    onnx.save_model(m, str(MODEL_OUT), save_as_external_data=False)
    sidecar.unlink()
```

---

## 与其他文件的关系

- **`src/lib.rs`**：特征参数的权威来源（N_MELS=40、WINDOW_FRAMES=100）
- **`src/bin/pet-features.rs`**：Rust 特征提取二进制，训练脚本调用它
- **`data/normal/`**：正常音频 WAV 文件
- **`data/petting/`**：宠物抓挠音频 WAV 文件
- **`models/pet_detect.onnx`**：导出的模型
- **`src/worker.rs`**：运行时消费导出的模型

---

## 关键设计要点

### 1. 特征提取与运行时共享同一 Rust 二进制

这是最重要的设计决策。训练和推理用完全相同的 log-mel 特征路径，消除了 Python/Rust 实现差异导致的训练/推理漂移。

### 2. 模型极小

约 20 KB、~1,300 参数。适合 CPU 推理，Radxa Zero 3W 上每 250 ms 推理一次。

### 3. 不做特征归一化

BatchNorm 层吸收分布差异。保持特征不归一化避免训练/推理漂移。

### 4. ONNX 合并 sidecar

新版 PyTorch 导出器生成两个文件（.onnx + .onnx.data），训练脚本合并为单个文件，运行时只需要一个 artifact。

### 5. 固定随机种子

种子 42，80/20 划分可复现。

---

## 关键踩坑点总结

1. **必须先构建 pet-features 二进制**：`cargo build --release -p pet-detect --bin pet-features`。没有它训练脚本报错退出。

2. **特征参数必须与 Rust 端同步**：N_MELS=40、WINDOW_FRAMES=100。改了 Rust 端这里必须同步改，否则特征字节数校验失败。

3. **forward 已应用 softmax，用 NLL loss**：不能用 CrossEntropyLoss（期望 logits），用 `nll_loss(log(probs))`。

4. **ONNX sidecar 合并**：新版 PyTorch dynamo 导出器生成 `.onnx.data` sidecar，必须合并回单个 `.onnx`。

5. **动态 batch 维度**：导出时 batch 维度标记为动态，运行时可以 batch=1 推理。

6. **没有 early stopping**：训练固定 1000 epoch。小数据集+小模型不会过拟合到需要 early stopping 的程度。
#（注：内容由AI生成）
