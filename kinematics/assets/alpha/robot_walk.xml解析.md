# 解析：`kinematics/assets/alpha/robot_walk.xml`

## 这是什么

Microduck 机器人的 **MuJoCo MJCF 运动学模型**（XML，110 行），模型名 `microduck`，对应 alpha 版本的几何。它是纯运动学模型——**只有 body / joint / site / inertial 元素，没有 actuator（执行器）、sensor（传感器）、contact（接触）、asset（网格/贴图）**，用途是定义“14 个关节角 → 各特征点（site）位姿”的正向运动学；[`fk_alpha.json`](../../../kinematics/tests/fixtures/fk_alpha.json解析.md) 的真值就是针对这同一个文件由 MuJoCo 计算的。

## 文件头（第 1-3 行）

```xml
<?xml version='1.0' encoding='utf-8'?>
<mujoco model="microduck">
  <compiler angle="radian" autolimits="true" />
```

- `angle="radian"`：关节范围与初始角一律用**弧度**（不是 MJCF 默认的度）；
- `autolimits="true"`：`<joint>` 只需写 `range`，限位自动生效，无需逐关节写 `limited="true"`。

## 单位与总体结构

MuJoCo 默认 SI 单位：长度 **米**、质量 **千克**、角度 **弧度**。整个模型藏在一个自由浮动基座下：

```
worldbody
└── trunk_base（躯干，freejoint = 6 自由度）
    ├── imu_bno / imu（site）
    ├── yaw2roll → 左腿链（5 关节）…… left_foot site
    ├── neck → 头部链（4 关节）…… head_camera / mouth_tip / tof / head_imu site
    └── bearing_roll → 右腿链（5 关节）…… right_foot site
```

**14 个铰链关节 + 1 个自由关节（6 自由度）= 20 自由度**——14 这个数与策略的 14 维动作、与 `fk_alpha.json` 的 14 个关节名完全一致。

## 基座 trunk_base（第 6-12 行）

```xml
<body name="trunk_base" pos="0 0 0.12" quat="1 0 0 0">
  <freejoint name="trunk_base_freejoint" />
  <inertial pos="-0.0184191 6.58793e-06 0.00237545" mass="0.264385" fullinertia="..." />
  <site group="3" name="imu_bno" pos="-0.032 0.0140011 0.0430618" quat="1 0 -0 -0" />
  <site group="3" name="imu" pos="-0.021 6.53121e-05 -0.0148911" quat="1 -0 0 0" />
```

- 初始位置 z=0.12 m（站立躯干高度），四元数单位姿态；
- 质量 0.264 kg，`fullinertia` 六元组给出完整惯量（Ixx Iyy Izz + 惯量积）；
- 两个 IMU 标记点：`imu_bno`（BNO 系列 IMU 的位置）与 `imu`（板载 IMU），`group="3"` 表示默认不渲染（仅供计算/传感器引用）。

## 三条运动链逐链解析

### 左腿：yaw2roll → hip_l → left_upper_leg → leg → ankle_left（第 14-44 行）

| 层 | body | 关节 | 范围（弧度） | 范围（约值） | 质量 |
|---|---|---|---|---|---|
| 1 | `yaw2roll` | `left_hip_yaw` | -0.4363 ~ 0.5236 | **-25° ~ +30°** | 0.0235 |
| 2 | `hip_l` | `left_hip_roll` | -0.3840 ~ 0.3840 | **±22°** | 0.0052 |
| 3 | `left_upper_leg` | `left_hip_pitch` | -1.5708 ~ 1.5708 | **±90°** | 0.0432 |
| 4 | `leg`（小腿） | `left_knee` | -1.5708 ~ 1.5708 | **±90°** | 0.0216 |
| 5 | `ankle_left` | `left_ankle` | -1.5708 ~ 1.5708 | **±90°** | 0.0265 |

链末端：`<site name="left_foot">`（脚底参考点）。每个 body 用 `pos`（相对父系的偏移）+ `quat`（相对父系的旋转）逐级定位，关节统一是 `axis="0 0 1"` 的 hinge——**轴向的实际空间方向由各级 body 的 quat 旋转链决定**，所以局部都写 z 轴即可。

### 头部：neck → neck_pitch → yaw_roll_motion → bottom_head_shell（第 46-75 行）

| 层 | body | 关节 | 范围（弧度） | 范围（约值） | 质量 |
|---|---|---|---|---|---|
| 1 | `neck` | `neck_pitch` | -1.5708 ~ 1.0472 | **-90° ~ +60°** | 0.0368 |
| 2 | `neck_pitch` | `head_pitch` | -1.5708 ~ 1.5708 | **±90°** | 0.0057 |
| 3 | `yaw_roll_motion` | `head_yaw` | -2.9671 ~ 2.9671 | **±170°** | 0.0249 |
| 4 | `bottom_head_shell` | `head_roll` | -0.4363 ~ 0.4363 | **±25°** | 0.1726 |

头壳是全身最重的单部件（0.173 kg）。头壳上挂 4 个 site：`head_camera`（摄像头光心）、`mouth_tip`（喙尖）、`tof`（ToF 深度传感器）、`head_imu`（头部 IMU）。

### 右腿：bearing_roll → hip_l_2 → right_upper_leg → leg_2 → ankle_right（第 77-107 行）

左腿链的**镜像**：关节为 `right_hip_yaw/roll/pitch`、`right_knee`、`right_ankle`，body 命名带 `_2` 后缀（`hip_l_2`、`leg_2`），末端 site `right_foot`。一个有意的不对称细节：右髋偏航范围是 **-0.5236 ~ 0.4363（-30° ~ +25°）**，与左腿的 -25°~+30° 正好左右镜像。

## 惯性参数的意义

每个 body 的 `<inertial pos= mass= fullinertia=>`：

- `pos`：质心在该 body 坐标系的位置（注意多个肢体的质心**不**在关节轴上，如大腿 0.0073/0.0225/0.0125）；
- `mass`：部件质量（kg）；
- `fullinertia`：绕质心三轴的完整惯量张量六元组。这些数值通常由 CAD/URDF 导出，供动力学仿真使用；**纯运动学计算（本夹具的用途）只读 body/site 的几何与关节角，惯量不参与**，但保留它们使模型也能直接用于动力学。

## 与 JSON 夹具的对应

| 本文件 | `fk_alpha.json` |
|---|---|
| 14 个 hinge joint | `joint_names` 的 14 个名字（顺序：左腿 5、颈/头 4、右腿 5） |
| 8 个 site（含 `imu_bno`、`head_imu`） | `site_names` 只取其中 6 个：`head_camera`、`imu`、`left_foot`、`right_foot`、`mouth_tip`、`tof` |
| `range`（弧度） | 夹具样本角度均落在这些范围之内（如 `head_yaw` 实测 -2.94~2.65，未越 ±2.967） |
| freejoint（躯干） | 夹具不设自由关节角：位姿以躯干为基准表达，`imu` site 在 64 个样本中恒定为其 XML 偏移 (-0.021, 6.53e-05, -0.0149) 即证据 |
