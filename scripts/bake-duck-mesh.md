# bake-duck-mesh.py

## 文件位置

`d:\microduck\scripts\bake-duck-mesh.py`

## 核心设计决策

该脚本将机器人的 MJCF 视觉模型烘焙（bake）为紧凑二进制文件 `robotctl/assets/duck.bin`，供 `robotctl monitor` 在终端上绘制实时 3D 视图使用。

- **为什么需要烘焙**：`robotctl` 作为单二进制发布到板端，没有 assets 目录；终端约为 100×100 像素显示，CAD 导出的 33 万三角形纯属浪费。因此离线将模型烘焙为数千三角形的紧凑二进制并提交。
- **格式版本机制**：头部 `FORMAT_VERSION = 1`，由 `robotctl/src/duck.rs` 读取，两者必须一致；版本不匹配时会明确报错而非绘制乱码。
- **顶点聚类而非边折叠**：在终端分辨率下只有轮廓能保留，聚类只需几行 numpy 即可实现，无需网格库。每个聚类的顶点取落入其中顶点的均值，使外壳向自身表面收缩而非向单元角收缩。
- **按部件尺寸分配三角形预算**：外壳比 22mm 轴承获得更多三角形。实例化网格（如 xl330 出现十几次）只简化一次，每个实例引用同一网格。
- **隐藏内部部件**：`HIDDEN` 集合包含被外壳包围的电池、PCB、支架等，从外部不可见，且在该简化级别其三角形会穿透外壳，丢弃后既更轻也更正确。

## 常量/类型/函数分析

### 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `FORMAT_VERSION` | `1` | 二进制格式版本号，头部写入，用于运行时兼容性检查 |
| `JOINT_NAMES` | 15 个关节名列表 | 复制自 `duck_ipc_proto::JOINT_NAMES`，运行时按存储的编号索引测量角度。`mouth` 是无 MJCF 关节的舵机（颚为固定 geom），不会出现在烘焙结果中 |
| `HIDDEN` | 6 个部件名集合 | `np_f970`、`pcb__raspberry_pi_zero_2_w`、`elec_rpi_robot_hat_pcb`、`power_support`、`banana_pcb_locker`、`motor_support` |

### 函数

| 函数 | 签名 | 功能 |
|---|---|---|
| `load_stl` | `(path: Path) -> np.ndarray` | 读取二进制 STL，返回 `(n, 3, 3)` float32 三角形顶点。验证文件长度 `84 + n * 50`，忽略法线（运行时重算） |
| `decimate` | `(tris, budget) -> (verts, faces)` | 在网格上聚类顶点直到三角形数不超过 `budget`。单元尺寸从 `diag/48.0` 起步，每次乘 1.3 直到满足预算。返回 `(v,3) f32` 顶点和 `(f,3) int` 面 |
| `flat_coords` | `(tris) -> np.ndarray` | 将三角形展平为 `(-1, 3)` float64 坐标，用于 `bincount` 加权求和 |
| `parse_floats` | `(text, default) -> np.ndarray` | 将空格分隔的浮点字符串解析为 float64 数组，`text` 为 None 时用 `default` |
| `quat_matrix` | `(q) -> np.ndarray` | MJCF 四元数 `(w x y z)` 转 3×3 旋转矩阵，先归一化 |
| `main` | `() -> None` | 入口：解析 MJCF、遍历 worldbody、烘焙网格/身体/部件、写入二进制、打印统计 |

### 二进制输出格式

头部：`struct.pack("<4sIHHH", b"DUCK", FORMAT_VERSION, n_meshes, n_bodies, n_parts)`

- 每个 mesh：`<HH`（顶点数、面数）+ 顶点 float32 + 面 uint16
- 每个 body：`<hh`（parent、joint）+ pos float32 + quat float32 + axis float32
- 每个 part：`<HHBBBB`（body、mesh、RGB、alpha=0）+ pos float32 + quat float32

## 关键要点总结

1. 该脚本是开发机工具，依赖 `python3 + numpy`，不在板端运行。
2. 重新运行时机：机器人模型变更时，命令为 `scripts/bake-duck-mesh.py ~/MISC/microduck_app/robot_assets/alpha/robot_walk.xml`。
3. `JOINT_NAMES` 是与 `duck_ipc_proto` 的契约副本，必须保持一致。
4. 三角形预算公式：`int(np.clip(40 + diag_mm * 3.5, 60, 420))`，针对约 100 像素高的显示和同时运行机器人的 CPU 调优。
5. 输出写入 `robotctl/assets/duck.bin`，统计输出网格数、身体数、部件数和实例化三角形总数。
