# `onnx.rs`（duck-detector）解读

## 概述

同一个检测器在 CPU 上的实现，用于 NPU 被关闭的板子。

### 为什么这个文件存在

RK3566 有 NPU，厂商内核有驱动——但在这台板子上，设备树把 `npu@fde40000` 标记为 `disabled`，Armbian 提供的唯一 overlay 是进一步禁用它的那个。启用它是一个 overlay 和一次重启，这是关于某人的机器人的决定，而非检测器的细节。所以检测器在四个 A55 核上运行，直到那发生，然后通过改一个配置值移到 NPU。

ONNX Runtime 已经在每个配置好的板子上——`setup-board.sh` 为 robotd 的策略安装它——`ort` 通过 dlopen 加载它，所以这在机器人上不增加新依赖。

---

## 核心数据结构

### `Model` — CPU 上的 YOLO 检测器

```rust
pub struct Model {
    session: Session,
    /// `[height, width, channels]`，如图声明的
    pub input: (usize, usize, usize),
}
```

- `session`：ONNX Runtime 会话
- `input`：输入形状，从图本身读取而非硬编码常量

---

## 核心函数

### `Model::open()` — 加载模型

```rust
pub fn open(path: &Path) -> Result<Self>
```

#### 构建会话

```rust
let session = Session::builder()
    .with_optimization_level(GraphOptimizationLevel::Level3)
    .with_intra_threads(2)
    .commit_from_file(path)?;
```

**两个线程，不是四个。** 另外两个属于 `robotd` 的控制循环和 GStreamer；一个花整个 SoC 来找 3 米外的鸭子的检测器，拿走了比它给出的更重要的东西。

#### 读取输入形状

```rust
let shape = session.inputs().first()
    .and_then(|input| input.dtype().tensor_shape().map(|dims| dims.to_vec()))
    .unwrap_or_default();
let input = match shape.as_slice() {
    [_, c, h, w] if *c == 3 => (*h as usize, *w as usize, *c as usize),
    other => bail!("expected a [1, 3, H, W] input, got {other:?}"),
};
```

NCHW，如导出的：`[1, 3, H, W]`。从模型本身读取形状，而非使用可能与它不一致的常量。

### `Model::infer()` — 一次推理

```rust
pub fn infer(&mut self, frame: &[u8], out: &mut Vec<f32>) -> Result<()>
```

一个 letterboxed RGB 帧进，原始头出——与 NPU 路径返回相同的布局，所以 `crate::decode` 不关心是哪个产生的。

#### 帧大小验证

```rust
let (height, width, channels) = self.input;
if frame.len() != height * width * channels {
    bail!("frame is {} bytes, the model wants {}", ...);
}
```

#### HWC 字节 → NCHW float，归一化

```rust
let mut planar = vec![0.0f32; frame.len()];
for y in 0..height {
    for x in 0..width {
        for c in 0..channels {
            planar[c * height * width + y * width + x] =
                frame[(y * width + x) * channels + c] as f32 / 255.0;
        }
    }
}
```

HWC 字节转 NCHW float，按导出期望的方式归一化（0..1）。NPU runtime 自己从烘焙到 `.rknn` 的 mean/std 做这件事；这里是我们自己做。

#### 构建输入张量并推理

```rust
let tensor = Tensor::from_array((
    [1_usize, channels, height, width],
    planar.into_boxed_slice(),
))?;
let outputs = self.session.run(ort::inputs!["images" => tensor])?;
let (_, data) = outputs[0].try_extract_tensor::<f32>()?;
out.clear();
out.extend_from_slice(data);
```

输入名称是 `"images"`——这是导出时图的输入名称，必须匹配。

输出提取为 f32，写入调用者提供的缓冲区。

---

## 与其他模块的关系

- **`crate::decode`**：后处理解码，`infer` 返回与 NPU 路径相同布局的原始头，所以 decode 不关心后端
- **`crate::rknn`**：NPU 后端，`Model` 接口与 onnx 后端对称（`open`/`infer`/`input`）
- **`ort` crate**：ONNX Runtime Rust 绑定，通过 dlopen 加载 `libonnxruntime.so`
- **`robotd`**：也使用 ONNX Runtime（策略推理），所以运行时已经在板子上
- **`setup-board.sh`**：安装 ONNX Runtime

---

## 关键踩坑点总结

1. **两个线程而非四个**：RK3566 有四个 A55 核，但另外两个属于 robotd 控制循环和 GStreamer。检测器花整个 SoC 来找鸭子是拿了比它给出的更重要的东西。

2. **NCHW 归一化是自己做的**：NPU runtime 从烘焙到 `.rknn` 的 mean/std 自己做归一化；CPU 路径必须手动做 HWC→NCHW 转置和 0..1 归一化。弄错这个检测器不会报错，只是悄悄变差。

3. **输入名称 `"images"` 必须匹配导出**：ONNX Runtime 按名称匹配输入张量，名称不匹配会运行时错误。这是从训练仓库导出时决定的。

4. **输入形状从模型读取而非硬编码**：从 `session.inputs()` 读取 `[1,3,H,W]`，而非使用可能与模型不一致的常量。不匹配时明确报错。

5. **dlopen 而非链接**：`ort` 通过 dlopen 加载 ONNX Runtime，所以交叉构建不需要目标架构的预编译库，笔记本上也能编译和测试（没有运行时的话推理会失败，但编译通过）。

6. **与 NPU 路径相同的输出布局**：`infer` 返回原始头（平面 `[1,5,N]`），与 rknn 后端完全一致，所以 `decode` 是后端无关的。切换后端只需要改配置值，不需要改后处理。
#（注：内容由AI生成）
