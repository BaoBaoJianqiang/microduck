# pet-detect

一个微型音频分类器，能听到机器人的头部何时被抓挠（板载麦克风就在那里）。
约 20 KB 的 CNN，基于 40 波段 log-mel 窗口，亚毫秒级推理。

从 `apirrone/microduck_pet_detect` 移植；原型 `pet_worker.rs` 中的 arecord worker
和环境声哨兵也住在这里，所以所有监听麦克风的东西都在一个 crate 中。`robotd`
运行这个 worker（见 `deploy/robotd.toml` 中的 `[audio]`）并在宠物开始时发出咕咕声。

- **库**：`PettingDetector`（流式，滞后）和 `worker::PetHandle`
  （arecord 子进程 + 事件通道）。
- **`pet-detect`**（二进制）：将 `arecord -D plughw:aic3104,0 -f S16_LE -r 16000 -c 1 -t raw`
  管道输入它，并在你抓挠头部时观察概率。
- **`pet-features`**（二进制）：转储 WAV 的 log-mel 特征——训练/推理一致性契约的训练端。
- **`models/pet_detect.onnx`**：训练好的模型，作为 `models/pet_detect.onnx` 随发布版发布。

## 重新训练

`training/train.py`（PyTorch）**通过** `pet-features` 二进制提取特征——
机器人运行的同一条代码路径，所以不存在训练/推理漂移。在机器人上录制数据：

```bash
arecord -D plughw:aic3104,0 -f S16_LE -r 16000 -c 1 -d 30 /tmp/petting_01.wav
```

放入 `data/petting/` 和 `data/normal/`（行走、电机、环境——任何不是宠物的东西；
录音本身不在这里 vendored），然后：

```bash
cargo build --release -p pet-detect --bin pet-features
uv run --with torch --with onnx training/train.py
```

并提交刷新后的 `models/pet_detect.onnx`。
#（注：内容由AI生成）
