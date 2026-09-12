# duck.rs 文件解析

## 文件位置

`d:\microduck\robotctl\src\duck.rs`

## 定位

`robotctl monitor` 的 3D 机器人实况视图。关节表只能说"每个舵机离目标多远"，无法说机器人长什么样——腿折错方向、头砸向地面、机器人侧躺都只是数字，画出来才一目了然。

## 模型

`static ASSET: &[u8] = include_bytes!("../assets/duck.bin")` — 由 `scripts/bake-duck-mesh.py` 从 app 仓库 MJCF 烘焙并提交（CI 无 numpy）。~330k CAD 三角形抽稀到几千，编译进二进制无资产目录，板子不解析 CAD。

`FORMAT_VERSION = 1` — 与 bake 脚本同步 bump，错时代 blob 拒绝解析而非画垃圾肢体。

## 渲染管线

1. **正向运动学**：body 按 parent 先于 child 排序，单次左到右 pass；trunk 由 IMU 投影重力姿态化（`attitude()`），关节由 `axis_pose()` 旋转
2. **顶点变世界坐标**：每 part 网格顶点变世界，找 floor（最低点）
3. **相机**：方位角(`azimuth`)+仰角(`ELEVATION=0.32`)，正交投影（100 像素下透视无意义）
4. **光栅化**：z-buffer 三角形填充（edge function），双面光照（抽稀不保 winding），地面网格点采样
5. **blit**：每单元两像素（`▀` 半块字符）使像素方正；未点亮用终端自身背景色

## 性能：缓存优先

`RASTER_INTERVAL = 80ms` — 机器人 50Hz 状态流，但视图每帧几毫秒 CPU（与控制循环共享），所以运动重光栅化 ~12fps，中间帧 blit 缓存。

- `pose_key()` — 关节量化到 ~0.3°、重力 0.01、marker 厘米级，防传感器噪声击败缓存
- `cached: (pose_key, camera_key, instant)` — 相机改变立即重绘（`[`/`]` 绕转需即时反馈）

## 缩放迟滞

`settle_zoom()` — 需要更大视野时**立即**放大，仅在 `ZOOM_HOLD=5s` 无远 marker 后才缩回。量化到 `ZOOM_STEP=0.05m` bins 防抖动。`MARKER_REACH=0.45m` 外的 marker 不画（不缩机器人去框远处墙）。

## ToF marker 叠加

`Marker{at, rgb}` — trunk 框架接触点，经 trunk body 完整 pose 变世界（非 bare attitude），z-tested 与网格同缓冲，画为十字形 blob。

## 测试

- `the_committed_blob_parses` — 15 body + >30 part
- `a_standing_robot_renders` — 默认姿态画 >200 cell
- `the_zoom_backs_out_instantly_and_creeps_back_in` — 5s 迟滞
- `a_tof_marker_paints_its_colour_and_a_far_one_is_dropped` — 近点上色、远点丢弃、消失点消失
- `frame_cost` (bench) — 板上测每帧成本：raster/+64点/pose-change/cached-blit

## 关键摘要

duck.rs 在终端用 half-block 字符渲染机器人 3D 视图：烘焙模型编译入二进制；正向运动学+z-buffer 光栅化；80ms 缓存节流防与控制循环抢 CPU；pose_key 量化防噪声击败缓存；缩放迟滞（立即放大/5s 后缩回）；ToF 接触点以十字标记叠加。
