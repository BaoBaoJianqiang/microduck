# duck.rs（终端 3D 机器人渲染器）解读与架构梳理

> 分析对象：`duck.rs`（952 行），robotctl monitor 中的 3D 机器人视图——在终端中用 z-buffered 光栅化器渲染烘焙的机器人网格。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是终端里的 3D 机器人视图。一个关节表只告诉你"每个舵机离目标多远"，但它看不出"腿折反了、头戳地了、机器人侧躺了"——姿势画出来立刻就明白了。此模块做这件事：用训练时的同一视觉模型，按线上测得的关节角摆姿势，按 IMU 投影重力倾斜，渲染到终端字符。

**关键设计**：
- **烘焙网格**：`scripts/bake-duck-mesh.py` 从 MJCF 烘焙，编译进二进制（`include_bytes!("../assets/duck.bin")`）。机器人保持单二进制无资产目录，板上不解析 CAD。
- **简化**：~330k CAD 三角形抽稀到几千个——终端能表达的量。
- **自写光栅化**：z-buffered、正交投影、平面着色。无 GPU、无依赖——一帧就是板上 CPU 上几毫秒的算术。
- **半块字符**：每个终端格画两个像素（上下半块 `▀▄`），像素变方。
- **帧率节流**：~12fps（80ms），中间帧 blit 缓存像素——监控器 50Hz 重绘，但渲染不能拖慢它观察的控制循环。

**模型结构**：15 个 body（躯干+两腿各5+头4），>30 个 part（网格实例），FORMAT_VERSION=1。

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | 烘焙网格 duck.bin 嵌入二进制 | L24 | confirmed |
| F2 | FORMAT_VERSION=1，blob 不匹配则拒绝解析 | L28, L124 | confirmed |
| F3 | 相机仰角 0.32 rad | L32 | confirmed |
| F4 | LOOK_AT_Z=0.15, WINDOW=0.36m | L38-39 | confirmed |
| F5 | 15 个 body（测试断言） | L736 | confirmed |
| F6 | 自写 3x3 旋转+平移 Pose | L191-224 | confirmed |
| F7 | Rodrigues 旋转（axis_pose） | L252-264 | confirmed |
| F8 | 重力→躯干姿态：最小旋转将重力对齐到下 | L292-310 | confirmed |
| F9 | 无 yaw（重力不可观测 yaw） | L287-288 | confirmed |
| F10 | 渲染节流 80ms，中间帧 blit | L319, L434-440 | confirmed |
| F11 | 姿态 key 量化到 ~0.3°（传感器噪声不触发重渲染） | L707-711 | confirmed |
| F12 | ToF marker：十字形 blob，z-tested | L556-585 | confirmed |
| F13 | MARKER_REACH=0.45m，更远的不画 | L333 | confirmed |
| F14 | 自动缩放：远点立即 zoom out，5 秒安静后 zoom in | L389-396 | confirmed |
| F15 | 地面网格：圆盘形（非方形），z=0 | L590-619 | confirmed |
| F16 | 双面着色（抽稀不保证绕序） | L531-533 | confirmed |
| F17 | PPM dump 测试（DUCK_DUMP 环境变量） | L833-881 | confirmed |
| F18 | 帧成本基准测试 | L901-950 | confirmed |

---

## 三、渲染管线

```
关节角 [f64; 15] + 重力 [f64; 3]
  ↓
attitude(gravity) → 躯干旋转（最小旋转对齐重力）
  ↓
FK: 每个 body 从父 body 累积 Pose（left-to-right pass）
  ├─ trunk: attitude(gravity)
  └─ 其他: parent_then(body_quat, body_pos) + joint_axis * angle
  ↓
每个 part: body_pose_then(part_quat, part_pos) → 顶点到世界
  ↓
相机: azimuth + elevation → 正交投影
  ↓
三角形光栅化: edge function + z-buffer + 双面平面着色
  ↓
ToF marker: 十字 blob，z-tested
  ↓
地面网格: 圆盘采样
  ↓
blit: 像素→字符（▀▄ 半块）
```

---

## 四、关键设计决策

### 4.1 为什么烘焙而非运行时解析 MJCF

- 板上不解析 CAD——节省内存和启动时间。
- 烘焙脚本在 app 仓库中，需要 numpy 和完整 MJCF——CI 没有这些。
- blob 提交到仓库，FORMAT_VERSION 同步升级——不匹配就拒绝解析而非画错。

### 4.2 为什么重力决定姿态而非完整 IMU 四元数

重力方向只能确定 roll+pitch，yaw 不可观测。所以：
- 躯干姿态 = 最小旋转将测量重力对齐到正下方。
- yaw 留给相机方位角（用户旋转相机观察）。
- 全零重力（旧 robotd）→ 直立（IDENTITY）。

### 4.3 为什么 80ms 节流

监控器以 50Hz 重绘。如果每帧都重新光栅化：
- 板上 CPU 也在跑 50Hz 控制循环。
- 一个"测量自己所测之物"的工具不应成为负担。
- 80ms ≈ 12fps——人眼够了，中间帧 blit 缓存。
- 相机旋转立即重渲染——手按 `[` 时视图必须立即动。

### 4.4 为什么姿态量化到 0.3°

传感器噪声让每个测量都微抖。如果不量化，缓存永远不命中——每个新帧都是"新姿势"，全部重光栅化。量化到 ~0.3°：噪声范围内不重绘。

---

## 五、结论

### confirmed
- C1：终端 3D 机器人视图，自写 z-buffer 光栅化器。
- C2：烘焙网格嵌入二进制，FORMAT_VERSION=1。
- C3：15 body，>30 part，~330k→几千三角形。
- C4：重力驱动 roll/pitch，yaw 由用户相机方位控制。
- C5：80ms 渲染节流 + 姿态量化 + blit 缓存。
- C6：ToF marker z-tested，MARKER_REACH=0.45m。
- C7：自动缩放迟滞：立即 out，5 秒安静后 in。
- C8：半块字符（▀▄）双像素/格。

### inferred
- I1：与 monitor.rs 集成，joints 和 gravity 来自 robot.state 流。
- I2：bake-duck-mesh.py 在 app 仓库。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
