# `lib.rs`（duck-detector）解读

## 概述

`duck-detector` crate 的根模块，负责在这台 Microduck 的摄像头中寻找其他 Microduck。模型在 [`duck_detector`](https://github.com/pollen-robotics/duck_detector) 中训练，以 INT8 `.rknn` 格式到达：一个类别，320×320 输入，2100 个候选框输出。

这个 crate 是摄像头帧和边界框之间的三件事——**letterbox（缩放填充）、runtime（推理后端）、decode（后处理解码）**——加上 `duck-bench`（在真实板子上测量它们）。

### 最危险的失败模式：静默变差

**这里的一切都必须与模型训练方式一致**，而跨两个仓库强制执行这一点的只有这个注释和下面的数字：

- 帧被 letterbox（保持比例缩放+填充），而非拉伸，成正方形，用 114 灰色填充
- RGB，不是 BGR
- 图片是 `mediad` 已经转正的那个（`--rotate`，默认 90°），因为数据集就是通过它采集的

**弄错其中任何一个，检测器不会失败——它只是悄悄变差**，这是这个 crate 最暴露的失败模式。

---

## 关键常量

```rust
const STRIDE: usize = 5;        // 每个候选的输出：cx, cy, w, h, score
pub const PAD: u8 = 114;        // ultralytics 填充 letterbox 用的灰色
```

- `STRIDE=5`：检测头每个候选输出 5 个值（中心 x、中心 y、宽、高、置信度）
- `PAD=114`：ultralytics YOLO 训练时 letterbox 填充用的灰色值，校准和训练都看到了这个值，所以推理时必须用同一个值

---

## 核心数据结构

### `Detection` — 一个检测结果

```rust
pub struct Detection {
    pub score: f32,
    pub box_: [f32; 4],  // x0 y0 x1 y1，原始帧坐标
}
```

坐标是**输入帧的坐标**，不是 letterbox 的坐标——解码时已经映射回去了。

#### `bearing()` — 行为真正想要的一个数字

```rust
pub fn bearing(&self, frame_width: f32) -> f32 {
    let centre = (self.box_[0] + self.box_[2]) / 2.0;
    (centre / frame_width) * 2.0 - 1.0
}
```

鸭子在帧中的方位：-1 最左，0 正前方，1 最右。

**"转向它"需要的是方位，不是框**——这是行为真正消费的一个数字。

### `Letterbox` — 帧如何适配到模型的正方形

```rust
pub struct Letterbox {
    pub scale: f32,
    pub pad_x: f32,
    pub pad_y: f32,
}
```

记录缩放比例和填充量，以便检测结果可以从 letterbox 坐标映射回原始帧坐标。

### `Turn` — 摄像头安装方向

```rust
pub enum Turn {
    #[default]
    None,
    Right,  // 顺时针 90°：这台机器人的摄像头安装需要的
    Half,
    Left,
}
```

#### 为什么在这里旋转而非在 pipeline 中

**这是性能决定，不是品味。** 在 tee 之前放一个 `videoflip` 花了机器人 145% 的一个核：`mpph264enc` 把 UYVY→NV12 免费交给 SoC 的 2D 引擎，而 flip 的缓冲区是 RGA 拒绝的（`RGA_BLIT fail: Bad address`），所以 MPP 回退到软件转换每一帧——97°C，CPU 节流到 408 MHz，30fps 摄像头只出 8fps。

这个采样器已经在重采样到 320×320，所以在同一次遍历中做旋转**完全不花钱**。

#### `from_degrees()` — 从角度创建

```rust
pub fn from_degrees(degrees: u32) -> Option<Self>
```

只接受 0/90/180/270，其他返回 `None`——这是标志的写法（顺时针角度）。

#### `upright()` — 旋转后的尺寸

```rust
pub fn upright(self, width: usize, height: usize) -> (usize, usize)
```

90°/270° 交换宽高，0°/180° 不变。

#### `source()` — 逆映射

```rust
fn source(self, ux: usize, uy: usize, width: usize, height: usize) -> (usize, usize)
```

**正立图片的一个像素在摄像头拍的帧中的位置**。逆映射，因为采样器遍历输出并从输入拉取。写成一个函数以便四种情况在一个地方而非散布在循环中。

顺时针 90°：源 (x,y) → 正立 (h-1-y, x)，所以逆映射取正立 (ux,uy) 从源 (uy, h-1-ux)。

---

## 核心函数

### `letterbox_rgb()` — RGB 帧 letterbox

```rust
pub fn letterbox_rgb(
    frame: &[u8], width: usize, height: usize,
    size: usize, out: &mut Vec<u8>,
) -> Letterbox
```

保持比例缩放到正方形，用 114 灰色填充，输出 NHWC RGB 字节。

#### 最近邻插值（故意的）

> 这在 50Hz 控制循环旁边逐帧运行，输入是模糊的 720×1280 房间照片，双线性缩放花三倍代价只为把框移动一个像素。如果测量说精度值得，RGA 可以免费做。

#### 几何计算

```rust
let scale = (size as f32 / width as f32).min(size as f32 / height as f32);
let fitted_w = ((width as f32 * scale).round() as usize).max(1).min(size);
let fitted_h = ((height as f32 * scale).round() as usize).max(1).min(size);
let pad_x = (size - fitted_w) / 2;
let pad_y = (size - fitted_h) / 2;
```

取较小的缩放比例（保持比例），居中填充。

#### 源坐标计算

```rust
let source_y = (y * height) / fitted_h.max(1);
let source_x = (x * width) / fitted_w.max(1);
```

从适配后的尺寸计算，这样舍入不会在最后一行走出源的末尾。

### `letterbox_from_uyvy()` — UYVY 直接 letterbox（一次遍历）

```rust
pub fn letterbox_from_uyvy(
    uyvy: &[u8], width: usize, height: usize,
    size: usize, turn: Turn, out: &mut Vec<u8>,
) -> Letterbox
```

**UYVY 直接进入 letterboxed RGB 正方形——一次遍历，只处理存活的像素。**

#### 为什么这取代了"先转换再缩小"

> 这取代了先转换帧再缩小它，后者在机器人上一次 look 花了 407ms 中的 345ms。tee 携带 720×1280 4:2:2 因为那是编码器想要的；模型想要 320×320 RGB。转换全部 921,600 像素然后扔掉 89% 是九倍的算术换同样的答案，所以这在目标网格上采样源——102,400 像素，整数数学，无中间缓冲区。

#### 所有计算在正立坐标中

```rust
let (upright_w, upright_h) = turn.upright(width, height);
```

下面的一切都在*正立*坐标中——图片转正后的样子，这是模型训练的样子，也是检测必须报告的样子。旋转只在获取源像素的那一刻被撤销。

#### UYVY 像素读取

```rust
let pair = row + (source_x / 2) * 4;
let luma = uyvy[pair + 1 + 2 * (source_x & 1)] as i32 - 16;
let u = uyvy[pair] as i32 - 128;
let v = uyvy[pair + 2] as i32 - 128;
```

UYVY 布局：U Y0 V Y1，每 4 字节两个像素共享色度。亮度是该像素所在半字节的奇数字节。

最近邻，色度从对中取不插值：输入是被缩小四倍的模糊房间照片，边界框里没有东西能在那个精度下存活。

#### BT.601 有限范围定点转换

```rust
let r = (298 * luma + 409 * v + 128) >> 8;
let g = (298 * luma - 100 * u - 208 * v + 128) >> 8;
let b = (298 * luma + 516 * u + 128) >> 8;
```

BT.601 有限范围，定点——ISP 的约定，也是数据集标注的每张 JPEG 经过的约定。整数因为这是内层循环。

#### 短帧保护

```rust
if row + stride > uyvy.len() {
    continue;  // 到达时正在拆除的帧是短的，缺失的保持填充而非让守护进程因为一张图崩溃
}
```

### `decode()` — 候选解码 + NMS + 坐标映射

```rust
pub fn decode(raw: &[f32], letterbox: Letterbox, threshold: f32, iou_limit: f32) -> Vec<Detection>
```

超过阈值的候选，抑制，映射回原始帧。

#### 检测头不做任何抑制

> 2100 个候选意味着一只鸭子回来是二十个重叠框，否则这个 crate 的每个消费者都得知道这件事。阈值是*这个*量化模型的属性：INT8 输出张量携带自己的 scale，所以 float 模型上意味着 0.9 的值在这里不是 0.9。对着板子调。

#### 张量布局：平面而非交错（最容易读错的地方）

```rust
// 张量是 [1, 5, N]：所有 cx 值，然后所有 cy 值，依此类推——不是每框五个数。
for index in 0..candidates {
    let score = raw[4 * candidates + index];
    let cx = raw[index];
    let cy = raw[candidates + index];
    let w = raw[2 * candidates + index];
    let h = raw[3 * candidates + index];
```

**`[1, 5, N]` 是平面的**：先所有 cx，再所有 cy。当作交错读取会产生几乎合理的框——这是最糟糕的错误，因为它看起来像坏模型而非坏读取器。

#### 从 letterbox 映射回帧

```rust
let unpad = |value: f32, pad: f32| (value - pad) / letterbox.scale;
```

减去填充，除以缩放比例。

#### NMS（非极大值抑制）

```rust
found.sort_by(|a, b| b.score.total_cmp(&a.score));
for detection in found {
    if kept.iter().all(|other| iou(&detection.box_, &other.box_) < iou_limit) {
        kept.push(detection);
    }
}
```

按分数降序，与已保留的所有框比较 IoU，超过限制则丢弃。

#### `iou()` — 交并比

```rust
fn iou(a: &[f32; 4], b: &[f32; 4]) -> f32
```

标准 IoU 计算，并集为零时返回 0。

---

## 测试要点

文件包含 8 个测试，覆盖：

1. **`a_portrait_frame_is_padded_left_and_right`** — 竖屏帧左右填充，不拉伸。摄像头在守护进程 90° 旋转后是竖屏 720×1280，所以这是唯一实际发生的情况。填充错误会把每个框移动一个没人注意的常数，直到机器人去够一只不在那里的鸭子。

2. **`uyvy_letterboxes_in_one_pass`** — 一次遍历转换与它取代的两步一致，包括几何。验证通道没交换（V 高是红）。

3. **`a_short_uyvy_frame_does_not_panic`** — 短帧不 panic，剩余保持填充。

4. **`a_quarter_turn_happens_while_sampling`** — 90° 旋转交换轴，角落落在旋转应该的位置。这是取代花 145% 核的 `videoflip` 的算术，所以最好是对的：镜像或转置的图片仍然会检测到*某些东西*，在一个都没训练过的模型上。

5. **`a_turned_frame_puts_the_bright_row_on_the_right_side`** — 旋转在像素中可见，不只是算术。顶行黑底行白，顺时针旋转后底行变成左列。

6. **`the_head_is_planar_not_interleaved`** — 检测头是平面的而非交错的。最可能读错的地方。

7. **`overlapping_candidates_collapse_and_map_back_to_the_frame`** — 一只鸭子二十个框一个幸存者，填充在输出路上被撤销。验证坐标映射和 bearing 计算。

8. **`two_ducks_stay_two`** — 两只远离的鸭子保持两只，不管检测头关于每只重复多少次。

---

## 与其他模块的关系

- **`crate::onnx`**：ONNX Runtime CPU 后端，`Model::infer` 返回与 NPU 路径相同布局的原始头，所以 `decode` 不关心是哪个产生的
- **`crate::rknn`**：Rockchip NPU 后端，dlopen `librknnrt.so`
- **`mediad`**：提供摄像头帧，已经通过 `--rotate`（默认 90°）转正
- **`duck_detector`**（外部仓库）：模型训练，INT8 量化导出 `.rknn`
- **`duck-bench`**：在真实板子上测量 letterbox/runtime/decode 的性能

---

## 关键踩坑点总结

1. **静默变差是最危险的失败模式**：letterbox 填充值、RGB/BGR、旋转方向——弄错任何一个检测器不会报错，只是悄悄变差。跨两个仓库（训练和推理）强制执行一致性的只有注释和数字。

2. **张量是平面的 `[1,5,N]` 而非交错的**：当作交错读取产生几乎合理的框，看起来像坏模型而非坏读取器。这是最容易读错的地方。

3. **旋转在采样器中做而非在 pipeline 中**：`videoflip` 花 145% 核（RGA 拒绝 flip 缓冲区 → MPP 软件转换 → 97°C/408MHz/8fps），采样器已经在重采样所以同一次遍历做旋转免费。

4. **UYVY 直接 letterbox 取代"先转换再缩小"**：后者花 407ms 中的 345ms，转换全部 921,600 像素然后扔掉 89%。直接在目标网格采样源只要 102,400 像素。

5. **PAD=114 必须与训练一致**：ultralytics 用 114 灰色填充 letterbox，校准和训练都看到了这个值，推理时必须用同一个。

6. **阈值是量化模型的属性**：INT8 输出张量携带自己的 scale，float 模型上 0.9 的值在这里不是 0.9，必须对着板子调。

7. **短帧保护**：到达时正在拆除的帧是短的，缺失的保持填充而非让守护进程因为一张图崩溃。

8. **检测头不做抑制**：2100 候选 → 一只鸭子二十个重叠框，decode 必须做 NMS，否则每个消费者都得知道这件事。
#（注：内容由AI生成）
