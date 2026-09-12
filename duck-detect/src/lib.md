# lib.rs 文件解析

## 文件位置

`d:\microduck\duck-detect\src\lib.rs`

## 核心设计决策

`duck-detect` 在 NPU 上从摄像头画面中寻找其他 Microduck。模型在 [duck_detector](https://github.com/pollen-robotics/duck_detector) 中训练，以 INT8 `.rknn` 形式到达此处：单类、320×320 输入、2100 个候选框输出。本 crate 是摄像头帧到边界框之间的三件事——letterbox（缩放进正方形）、runtime、decode（解码）——加上 `duck-bench` 在真实板上测量它们。

**一切都必须与模型训练方式一致**，跨两个仓库没有任何机制强制，只有本注释和下面的数字：
- 帧做 letterbox（缩放适配 + 填充）而非拉伸成正方形，用 114 灰填充
- RGB，非 BGR
- 画面是 `mediad` 已转正的（`--rotate`，默认 90°），因为数据集就是这样采集的

搞错其中任何一项，检测器不会失败——只会悄悄变差，这是本 crate 最暴露的失败模式。

## 常量

- `STRIDE = 5` — head 每个候选输出：cx, cy, w, h, score
- `PAD = 114` — ultralytics 做 letterbox 时的填充灰值（校准和训练所见的值）

## 类型分析

### `Detection`
一个检测结果，坐标在*输入帧*而非 letterbox 中：
- `score: f32`
- `box_: [f32; 4]` — 像素 `x0 y0 x1 y1`，原始帧坐标
- `width()` / `height()` — 宽高
- `bearing(frame_width) -> f32` — 鸭子在画面中的方位：-1 最左、0 正前、1 最右。行为真正想要的一个数——"转向它"需要方位而非框

### `Letterbox`
帧如何拟合进模型正方形：`scale`、`pad_x`、`pad_y`，用于把检测映射回原帧。

### `Turn`
摄像头安装方向，采样器需把画面转多少：
- `None` / `Right`（顺时针 90°，本机器人摄像头安装所需）/ `Half` / `Left`
- `from_degrees(degrees)` — 从角度（顺时针）构造
- `upright(width, height)` — 转正后的尺寸（90°/270° 交换轴）
- `source(ux, uy, w, h)` — 逆映射：正立像素在原始帧中的位置（采样器遍历输出、从输入取像素）

## 函数分析

### `letterbox_rgb(frame, width, height, size, out) -> Letterbox`
缩放适配并填充为正方形，写入 NHWC RGB 字节。**故意用最近邻**：这在 50Hz 控制循环旁逐帧运行，输入是模糊的 720×1280 房间照片，双线性缩放代价是三倍只为把框移动一个像素。

### `letterbox_from_uyvy(uyvy, width, height, size, turn, out) -> Letterbox`
**UYVY 直接转 letterboxed RGB 正方形——一次遍历，且只采样存活像素。** 这取代了"先转帧再缩放"的方案，后者在机器人上一次查找耗时 407ms 中占 345ms。tee 携带 720×1280 4:2:2（编码器需要），模型要 320×320 RGB。转换全部 921,600 像素再丢弃 89% 是九倍算术换同一答案，所以此处改为在目标网格上采样源——102,400 像素、整数运算、无中间缓冲。

所有计算在*正立*坐标中进行，仅在取源像素时撤销转向。UYVY→RGB 用 BT.601 有限范围定点整数（ISP 约定，数据集标注的每张 JPEG 都经过它）。短帧（teardown 中到达）缺失部分留为填充而非 panic。

### `decode(raw, letterbox, threshold, iou_limit) -> Vec<Detection>`
超过阈值的候选，经 NMS 抑制，映射回原帧。**head 不做任何抑制**：2100 个候选意味着一只鸭子回来时是 20 个重叠框，否则每个消费者都得知道这点。阈值是*本模型*（量化后）的属性：INT8 输出张量自带 scale，float 模型的 0.9 在这里不是 0.9。

**张量布局是 `[1, 5, N]`（planar）**：所有 cx，然后所有 cy……而非每框五个数。按交错读会得到几乎合理的框，是最坏的错误类型。读分数 `raw[4*candidates + index]`。NMS：按分数降序，保留与所有已保留框 IoU < `iou_limit` 的。

### `iou(a, b) -> f32`
交并比。

## 单元测试描述

- `a_portrait_frame_is_padded_left_and_right`：竖屏帧（4×8）放进 8×8 正方形，左右各 2 列填充，无拉伸。
- `uyvy_letterboxes_in_one_pass`：UYVY 一次遍历的几何与 RGB 一致，通道不互换（V 高为红）。
- `a_short_uyvy_frame_does_not_panic`：短帧不 panic，留填充。
- `a_quarter_turn_happens_while_sampling`：90°/270° 转向交换轴；通过逆映射验证角落位置（取代了耗费 145% 核心的 `videoflip`）。
- `a_turned_frame_puts_the_bright_row_on_the_right_side`：转向在像素中可见（底行变左列）。
- `the_head_is_planar_not_interleaved`：head 是 planar 布局 `[1,5,N]`，验证两个候选的解码结果。
- `overlapping_candidates_collapse_and_map_back_to_the_frame`：5 个近同候选坍缩为 1 个，并正确反 pad 回原帧坐标。
- `two_ducks_stay_two`：两只远离的鸭子保持两个检测。

## 关键摘要

`lib.rs` 是检测器的前处理（letterbox）与后处理（decode）核心。关键设计：UYVY→RGB letterbox 一次遍历（省 345ms）、转向在采样时完成（省 145% 核心）、head 张量为 planar `[1,5,N]` 布局、NMS 在 decode 中完成、检测坐标映射回原帧。所有预处理约定（114 灰填充、RGB、BT.601、最近邻）必须与训练一致，否则检测器悄悄变差。
