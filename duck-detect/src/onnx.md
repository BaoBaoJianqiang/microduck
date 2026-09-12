# onnx.rs 文件解析

## 文件位置

`d:\microduck\duck-detect\src\onnx.rs`

## 核心设计决策

`onnx.rs` 提供**同一检测器的 CPU 版本**，用于 NPU 被关闭的板子。

RK3566 有 NPU，厂商内核有驱动——但在这块板上设备树把 `npu@fde40000` 标为 `disabled`，Armbian 提供的唯一 overlay 还是进一步禁用它。启用需要 overlay + 重启，这是关于某人的机器人的决定而非检测器的细节。所以检测器先在 4 个 A55 核上运行，改一个配置值即可移到 NPU。

ONNX Runtime 已在每块已部署的板上（`setup-board.sh` 为 `robotd` 的策略安装它），`ort` 用 dlopen 加载，所以不增加机器人上的新依赖。

## 类型分析

### `Model`
CPU 上的 YOLO 检测器：
- `session: Session` — ONNX Runtime 会话
- `input: (usize, usize, usize)` — 图声明的 `[height, width, channels]`

## 方法分析

### `open(path) -> Result<Self>`
加载并验证模型：
- 图优化 `Level3`
- **2 个线程，非 4 个**：另外两个核属于 `robotd` 的控制循环和 GStreamer；一个为了找 3m 外的鸭子而占满整个 SoC 的检测器，拿走了比它给出的更重要的东西
- 从文件提交会话
- 读取输入形状，期望 `[1, 3, H, W]`（NCHW，导出时格式），否则 `bail!`

### `infer(frame, out) -> Result<()>`
一个 letterboxed RGB 帧进，raw head 出——与 NPU 路径返回相同布局，`crate::decode` 不关心是谁产生的：
- 校验帧长度 = `height * width * channels`
- HWC 字节转 NCHW float，归一化到 0..1（NPU runtime 从 `.rknn` 中烘焙的 mean/std 自己做，此处自己做）
- 构造 `[1, channels, height, width]` 张量，输入名 `images`
- 运行推理，提取第一个输出的 `f32` 张量，写入 `out`

## 关键摘要

`onnx.rs` 是 NPU 不可用时的 CPU 回退路径。用 `ort` dlopen ONNX Runtime（板上已为 `robotd` 安装）。关键决策：仅用 2 个线程（留 2 核给控制循环和 GStreamer），NCHW 输入布局，0..1 归一化。输出布局与 NPU 路径完全一致，使 `decode` 通用。
