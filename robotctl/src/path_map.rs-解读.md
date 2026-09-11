# path_map.rs（盲文轨迹地图）解读与架构梳理

> 分析对象：`path_map.rs`（354 行），monitor 中的顶视里程计轨迹图。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是顶视轨迹图——回答"机器人去了哪里"。它住在 3D 机器人视图下方：3D 视图回答"机器人在做什么"，这个回答"它去过哪里"。面板大小不变——而是**世界缩放**：轨迹变长时缩小视野，让整条路始终在画面里。用盲文字符画（每格 2×4 点）——这是终端能提供的最细线分辨率。

**坐标系约定**：
- 世界 +x（开机航向）→ 屏幕上方
- 世界 +y（机器人左侧）→ 屏幕左侧
- 原点标 `+`，机器人标 `●` 带黄色航向射线

**关键参数**：
- FIRST_STEP=0.02m：机器人至少移动多少才记录新点
- CAPACITY=2048：记录点上限
- MIN_SPAN_M=1.0：世界缩放下限（静止抖动不放大成涂鸦）
- 抽稀：满了之后保留首尾、隔点取、min_step 翻倍

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | 顶视图，世界自适应缩放 | L1-7 | confirmed |
| F2 | 盲文字符 2×4 dots/cell | L6, L175 | confirmed |
| F3 | +x 向上，+y 向左 | L9-11 | confirmed |
| F4 | 原点 +，机器人 ●+黄色射线 | L12, L127-138 | confirmed |
| F5 | FIRST_STEP=0.02m | L21 | confirmed |
| F6 | CAPACITY=2048 | L25 | confirmed |
| F7 | MIN_SPAN_M=1.0m（防静止抖动放大） | L29 | confirmed |
| F8 | 抽稀：保留首尾、隔点取、step 翻倍 | L61-72 | confirmed |
| F9 | Bresenham 画线 | L184-210 | confirmed |
| F10 | 盲文 BITS: 1-2-3-7 左列，4-5-6-8 右列 | L175 | confirmed |
| F11 | 航向射线：5/scale 面板步长 | L135-137 | confirmed |
| F12 | 6% 边距 | L108-110 | confirmed |
| F13 | 小区域 draw 不 panic | L98, L348-353 | confirmed |

---

## 三、数据结构

```
PathMap {
  points: Vec<(f64, f64)>,  // 轨迹点，旧→新，间距 ≥ min_step
  min_step: f64,              // 当前抽稀步长（初始 0.02m）
  here: Option<(x,y,yaw)>,    // 当前位置（每帧更新）
}
```

**observe(x, y, yaw) 逻辑**：
1. 记录 here（当前位置）
2. 与最后一个记录点距离 ≥ min_step？
   - 否 → return（不记录）
   - 是 → push 新点
3. 点数 ≥ CAPACITY？
   - 是 → 保留首尾 + 隔点取 → min_step *= 2

---

## 四、绘制管线

```
points[] + here
  ↓
bounds(): min/max（含原点和 here）
  ↓
scale: 等比缩放，整个 box 装下 + 6% 边距
  ↓
dot(x,y): 世界坐标 → 盲文格坐标
  ↓
Grid (2×4 dots/cell):
  ├─ windows(2) Bresenham 画线
  ├─ 原点: '+' DarkGray
  └─ 机器人: '●' Yellow + 航向射线
  ↓
paint: dots → U+2800+mask 盲文字符
```

---

## 五、关键设计决策

### 5.1 为什么盲文

每格 2×4 点——终端能提供的最高线分辨率。比单字符像素好 8 倍，轨迹曲线更平滑。

### 5.2 为什么世界缩放而非面板缩放

面板大小不变（固定在 3D 视图下方）。世界缩放：走远了自动缩小，近了自动放大。MIN_SPAN_M=1.0 防止站着不动时 IMU 抖动放大成涂鸦。

### 5.3 为什么抽稀保留首尾

保留首点（原点——你从哪来的）和尾点（最新位置——你在哪）。中间隔点取——形状变细但两端不丢。min_step 翻倍——长距离行走时分辨率自然降低。

### 5.4 为什么航向射线固定面板步长

`reach = 5.0 / scale`——5 个面板像素，不管缩放多少。这样航向射线在任何缩放级别都可见，不会随地图缩小成一个点。

---

## 六、结论

### confirmed
- C1：盲文轨迹图，2×4 dots/cell，世界自适应缩放。
- C2：抽稀 CAPACITY=2048，保留首尾，min_step 翻倍。
- C3：MIN_SPAN_M=1.0 防静止抖动。
- C4：+x 向上 +y 向左，原点 +，机器人 ●+黄射线。
- C5：Bresenham 画线，盲文 U+2800+mask。

### inferred
- I1：里程计数据来自 robot.state 流（x, y, yaw）。
- I2：与 3D 视图共享面板区域，路径图在下方。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
