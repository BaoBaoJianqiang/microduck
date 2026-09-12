# path_map.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\path_map.rs`

## 定位

`robotctl monitor` 中 3D 视图下方的俯视里程计轨迹图。视图答"机器人在做什么"，这答"它去过哪里"。面板大小不变，**世界缩放**（轨迹增长时缩小，全路径始终在框内）。用盲文（每单元 2×4 点）绘制。

## 约定

世界 +x（启动时朝向）向上，+y（左）向左——站在机器人起始姿态上方看到的样子。原点标 `+`，机器人 `●` 带朝向射线。

## 抽稀保形

- `FIRST_STEP=0.02m` — 机器人移动至少此距离才记录下一点
- `CAPACITY=2048` — 记录点上限
- 满时保留第 0 个和最后一个，中间每隔一个取一个，`min_step *= 2`——形状变薄但两端保留，可达距离翻倍（~40m 首次抽稀）

## 关键方法

- `observe(x, y, yaw)` — 喂一帧里程计估计
- `extent_m()` — 当前世界宽度（面板标题显示缩放级别）
- `bounds()` — 包含所有点+live 位置+始终原点的框
- `draw(area, buf)` — 等比缩放（两轴同 scale，不变形），居中，6% margin

## Grid — 盲文点阵

- `dots: Vec<u8>` — 每单元盲文位掩码（`U+2800 + mask`）
- `over: Vec<Option<(char,Color)>>` — 字符覆盖（`+`/`●` 等标记优先于轨迹）
- `line()` — Bresenham 画线，端点可远在屏外（`set` 裁剪），步数受面板限制
- `mark()` — 放置覆盖字符
- `paint()` — 逐单元：over 优先，否则盲文字符

## 测试

- `a_standing_robot_is_a_dot_in_the_middle` — 不动=中点，不放大进抖动
- `forward_is_up_and_the_zoom_follows_the_walk` — 向前走向上画，extent 跟随
- `a_long_walk_thins_but_keeps_its_ends` — 200k 步仍 < CAPACITY，两端保留
- `a_tiny_area_draws_nothing_and_survives` — 窄面板不 panic

## 关键摘要

path_map.rs 用盲文点阵绘制俯视里程计轨迹：面板固定大小、世界自动缩放；CAPACITY 上限+每隔一个抽稀保两端；等比缩放不变形；标记（原点+/机器人●+朝向射线）用字符覆盖优先于盲文轨迹；Bresenham 画线端点可屏外。
