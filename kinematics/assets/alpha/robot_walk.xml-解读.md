# robot_walk.xml（alpha MJCF 模型）解读与架构梳理

> 分析对象：`robot_walk.xml`（110 行），microduck 机器人的 alpha 运动学模型——被 kinematics crate 通过 `include_str!` 嵌入二进制。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 microduck 机器人的 **alpha 版本 MuJoCo MJCF 运动学模型**，定义了双足行走机器人的完整刚体树：1 个自由躯干关节 + 14 个铰链关节（每腿 5 个 + 头部 4 个）、8 个命名 site（传感器和端点）、15 个刚体（含惯性参数）。文件被 kinematics crate 通过 `include_str!("../assets/alpha/robot_walk.xml")` 嵌入，在 `Model::alpha()` 中 `LazyLock` 解析一次。

**关键参数**：
- **站立高度**：trunk_base 在 world pos Z=0.12m——这就是 `Model::trunk_height_m()` 返回的值，tof.rs 地板过滤用它。
- **总质量**：约 0.744 kg（躯干 0.264 kg + 头部 0.240 kg + 双腿各 ~0.120 kg）。
- **head_yaw 行程**：±2.967 rad = **±170°**——这正是 head.rs 凝视 IK 测试中正后方目标 clamp 在 yaw 极限的来源。
- **角度单位**：`compiler angle="radian"`，所有 range 属性用弧度——与 mjcf.rs 解析器假设一致。
- **自动限位**：`autolimits="true"`——无 `<equality>` 或 `<tendon>` 限制。
- **所有铰链轴都是 +z**：物理轴方向通过 body 的四元数旋转实现，这是 MuJoCo 常见模式。

**与 kinematics crate 的对应关系**：
- `trunk_base` → mjcf.rs 的 NoTrunkBase 检查、root identity 锚定。
- 14 个关节 → lib.rs `num_joints()`、head.rs "alpha model has 14 joints"。
- `head_camera` → head.rs `camera_in_trunk_cv2`。
- `tof` site → head.rs `tof_in_trunk`（优先用此 site 而非借用相机）。
- `left_foot`/`right_foot` → lib.rs 完整性测试要求的 site。
- `trunk_base` pos Z=0.12 → `trunk_height_m`。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 robot_walk.xml |
| 文件类型 | MuJoCo MJCF XML 模型文件 |
| 版本/模型 | alpha（`model="microduck"`） |
| 行数 / 已读范围 | 110 行，全文已完整读取（L1–L110） |
| 嵌入位置 | kinematics crate: `include_str!("../assets/alpha/robot_walk.xml")` |
| 主要证据 | 文件本身；kinematics crate 全部 6 个源文件 |
| 不可读 / 未提供 | 对应 geoms（视觉碰撞体）、actuators、sensors、training scene（mjlab_microduck） |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | model="microduck"，compiler angle="radian" autolimits="true" | L2–L3 | 头部属性 | confirmed |
| F2 | trunk_base pos="0 0 0.12" quat="1 0 0 0" | L6 | 站立高度 12cm | confirmed |
| F3 | trunk_base 有 freejoint | L7 | 6 自由度浮动基 | confirmed |
| F4 | 躯干质量 0.264kg，惯性矩阵完整 | L8 | inertial | confirmed |
| F5 | 两个 IMU site：imu_bno 和 imu | L10, L12 | group=3 | confirmed |
| F6 | 左腿链：yaw2roll→hip_l→left_upper_leg→leg→ankle_left→left_foot | L14–L44 | 5 关节 | confirmed |
| F7 | 右腿链：bearing_roll→hip_l_2→right_upper_leg→leg_2→ankle_right→right_foot | L77–L107 | 5 关节（镜像） | confirmed |
| F8 | 头链：neck→neck_pitch→yaw_roll_motion→bottom_head_shell | L46–L75 | 4 关节 | confirmed |
| F9 | 14 个铰链关节，全部 axis="0 0 1" | 全文 | 轴方向靠 body quat | confirmed |
| F10 | head_yaw range=±2.967 rad=±170° | L58 | 非对称（数值上对称） | confirmed |
| F11 | neck_pitch range=-1.57~1.047 rad（不对称） | L48 | 低头多抬头少 | confirmed |
| F12 | head_roll range=±0.436 rad=±25° | L63 | 小范围 | confirmed |
| F13 | head_camera site pos="0.01175 0 -0.0735" | L66 | 喙前方 7.35cm | confirmed |
| F14 | tof site pos="0.0143 0.0225 -0.0735" | L69 | 偏右 2.25cm（X？） | confirmed |
| F15 | mouth_tip site 非单位四元数 | L67 | 喙尖方向 | confirmed |
| F16 | head_imu site | L71 | 头部 IMU | confirmed |
| F17 | left_foot site 在 ankle_left 下 | L39 | 足部接触点 | confirmed |
| F18 | 总质量 ~0.744 kg | 计算 | 15 个 inertial 之和 | confirmed |
| F19 | 无 geoms、无 actuators、无 sensors | 全文 | 仅运动学+惯性 | confirmed |
| F20 | bottom_head_shell 质量 0.173 kg（头部主体） | L64 | 最大单刚体 | confirmed |

---

## 四、模型结构详解

### 4.1 编译器设置（L3）

```xml
<compiler angle="radian" autolimits="true" />
```

- `angle="radian"`：所有角度属性用弧度。**这与 mjcf.rs 解析器假设一致**——mjcf.rs 不做度→弧度转换。
- `autolimits="true"`：自动设置关节限位，不显式写的 range 为 ±π。本模型所有关节都显式写了 range。

### 4.2 躯干（L6–L12）

```xml
<body name="trunk_base" pos="0 0 0.12" quat="1 0 0 0">
  <freejoint name="trunk_base_freejoint" />
  <inertial ... mass="0.264385" ... />
  <site name="imu_bno" pos="-0.032 0.014 0.043" .../>
  <site name="imu" pos="-0.021 0.00007 -0.0149" .../>
</body>
```

- **pos Z=0.12**：这就是 `trunk_height_m`。训练世界中地板在躯干帧 Z=-0.12。
- **freejoint**：躯干在世界中自由浮动（6 DOF）。kinematics crate 忽略 freejoint——它只做躯干帧内的 FK。
- **两个 IMU site**：`imu_bno`（BNO055？）和 `imu`，位置不同，可能是主备或不同帧约定。
- 躯干质量 264g，是最大的单一刚体。

### 4.3 左腿链（L14–L44）

```
yaw2roll (pos 0.006, 0.0175, -0.005, quat 0,-.707,-.707,0)
  └─ left_hip_yaw    range -0.436~0.524 rad
     └─ hip_l (pos 0, 0.0165, 0.0125, quat .707,-.707,0,0)
        └─ left_hip_roll  range ±0.384 rad (±22°)
           └─ left_upper_leg (pos 0.025, 0, -0.0185, quat .5,-.5,.5,-.5)
              └─ left_hip_pitch range ±1.571 rad (±90°)
                 └─ leg (pos 0.022, 0.0358, -0.004, quat 0,.707,.707,0)
                    └─ left_knee  range ±1.571 rad (±90°)
                       └─ ankle_left (pos 0, 0.042, -0.026, quat 0,1,0,0)
                          └─ left_ankle range ±1.571 rad (±90°)
                             └─ left_foot site (pos 0, -0.0238, -0.0141)
```

**关节行程**：
| 关节 | range (rad) | 约 |
|------|-------------|-----|
| left_hip_yaw | -0.436 ~ 0.524 | -25° ~ +30°（不对称！） |
| left_hip_roll | ±0.384 | ±22° |
| left_hip_pitch | ±1.571 | ±90° |
| left_knee | ±1.571 | ±90° |
| left_ankle | ±1.571 | ±90° |

注意 hip_yaw 不对称（-25°/+30°），右腿是镜像（-30°/+25°）——这反映了髋关节机械结构的不对称。

### 4.4 右腿链（L77–L107）

右腿是左腿的镜像：
- `bearing_roll` 对应 `yaw2roll`，pos y=-0.0175。
- right_hip_yaw range = -0.524 ~ 0.436（左腿的镜像）。
- 其余 body 名和关节名用 right_ 前缀。
- 惯性参数与左腿对称（符号相反）。

### 4.5 头部链（L46–L75）

```
neck (pos 0.026, 0.0145, 0.0324, quat 0,0,-.707,.707)
  └─ neck_pitch  range -1.571~1.047 rad (-90°~+60°)
     └─ neck_pitch body (pos 0, -0.05, 0, quat 0,1,0,0)
        └─ head_pitch range ±1.571 rad (±90°)
           └─ yaw_roll_motion (pos 0, 0.0187, -0.0145, quat 0,0,-.707,-.707)
              └─ head_yaw range ±2.967 rad (±170°)
                 └─ bottom_head_shell (pos -0.0179, 0, 0.0145, quat .707,0,-.707,0)
                    └─ head_roll range ±0.436 rad (±25°)
                       ├─ head_camera site (pos 0.0118, 0, -0.0735)
                       ├─ mouth_tip site (pos -0.0083, 0, -0.0777)
                       ├─ tof site (pos 0.0143, 0.0225, -0.0735)
                       └─ head_imu site (pos 0.0115, 0.0002, -0.0513)
```

**关键发现**：
- **head_yaw ±170°**：这就是 head.rs 中 `look_at([-1,0,0], 0)` 测试期望 clamp 在 yaw 极限的来源。机器人几乎可以正后方看，但不能完全 180°。
- **neck_pitch 不对称**：-90° ~ +60°（低头比抬头多 30°）——这是机械结构限制。
- **head_roll ±25°**：小范围，保持头部水平。
- **bottom_head_shell 质量 0.173 kg**：占总质量 23%——头部是最重的单刚体（可能容纳了电池和主板）。
- **head_camera 和 tof 在同一 Z=-0.0735**：两者平行安装，tof 偏 X=+0.014, Y=+0.0225（约 2.25cm 横向偏移）。
- **mouth_tip**：喙尖位置，非单位四元数——喙不是直的，有偏航。

### 4.6 关节汇总（14 个）

| # | 关节名 | 链 | range (rad) | 约 |
|---|--------|-----|-------------|-----|
| 1 | left_hip_yaw | 左 | -0.436~0.524 | -25°~+30° |
| 2 | left_hip_roll | 左 | ±0.384 | ±22° |
| 3 | left_hip_pitch | 左 | ±1.571 | ±90° |
| 4 | left_knee | 左 | ±1.571 | ±90° |
| 5 | left_ankle | 左 | ±1.571 | ±90° |
| 6 | neck_pitch | 头 | -1.571~1.047 | -90°~+60° |
| 7 | head_pitch | 头 | ±1.571 | ±90° |
| 8 | head_yaw | 头 | ±2.967 | ±170° |
| 9 | head_roll | 头 | ±0.436 | ±25° |
| 10 | right_hip_yaw | 右 | -0.524~0.436 | -30°~+25° |
| 11 | right_hip_roll | 右 | ±0.384 | ±22° |
| 12 | right_hip_pitch | 右 | ±1.571 | ±90° |
| 13 | right_knee | 右 | ±1.571 | ±90° |
| 14 | right_ankle | 右 | ±1.571 | ±90° |

### 4.7 Site 汇总（8 个命名 site）

| Site | 父 body | pos (m) | 用途 |
|------|---------|---------|------|
| imu_bno | trunk_base | -0.032, 0.014, 0.043 | IMU（BNO055？） |
| imu | trunk_base | -0.021, 0, -0.015 | IMU（另一帧？） |
| left_foot | ankle_left | 0, -0.024, -0.014 | 左脚接触点 |
| head_camera | bottom_head_shell | 0.012, 0, -0.074 | 头部相机 |
| mouth_tip | bottom_head_shell | -0.008, 0, -0.078 | 喙尖 |
| tof | bottom_head_shell | 0.014, 0.023, -0.074 | ToF 传感器 |
| head_imu | bottom_head_shell | 0.012, 0, -0.051 | 头部 IMU |
| right_foot | ankle_right | 0, 0.024, -0.014 | 右脚接触点 |

### 4.8 质量分布

| 刚体 | 质量 (kg) | 占比 |
|------|-----------|------|
| trunk_base | 0.264 | 35.5% |
| bottom_head_shell | 0.173 | 23.2% |
| left_upper_leg | 0.043 | 5.8% |
| right_upper_leg | 0.043 | 5.8% |
| neck | 0.037 | 4.9% |
| ankle_left | 0.026 | 3.6% |
| ankle_right | 0.026 | 3.6% |
| yaw_roll_motion | 0.025 | 3.3% |
| yaw2roll / bearing_roll | 各 0.024 | 各 3.2% |
| leg / leg_2 | 各 0.022 | 各 2.9% |
| neck_pitch body | 0.006 | 0.8% |
| hip_l / hip_l_2 | 各 0.005 | 各 0.7% |
| **总计** | **~0.744** | 100% |

重心分布：躯干（35.5%）+ 头部（23.2%）= 58.7% 在上半身。这是一个上身较重的双足机器人——需要仔细平衡才能行走。

---

## 五、与 kinematics crate 的对应（ROUTE）

```
robot_walk.xml
  → mjcf.rs:parse()
    找 worldbody → 找 trunk_base
    递归 walk_body → 15 个 Body, 14 个 Joint, 8 个 Site
    trunk_pos = [0, 0, 0.12] → trunk_height = 0.12
  → lib.rs:Model::parse()
    关节按 tree 顺序索引（14 个）
    每 site 扁平 root→site 链
  → Model::alpha() LazyLock 单例
    → head.rs: HeadFk::new 找 head_camera, tof, 4 头部关节
    → tof.rs: Reprojector 用 trunk_height_m = 0.12
    → hand.rs: 用 ROWS=8, COLS=8
```

**关键对应验证**：
- head.rs 测试要求 `left_foot`/`right_foot`/`head_camera`/`tof` site → 全部存在（F10, F14, F17）。
- head.rs 测试要求 `left_hip_yaw`/`right_ankle`/`neck_pitch`/`head_roll` 关节 → 全部存在。
- head.rs "alpha model has 14 joints" → 确认 14 个（F9）。
- head_yaw range ±2.967 → look_at clamp 测试期望 ±170°。
- trunk_height = 0.12 → tof.rs 地板过滤的站立高度。

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："kinematics crate 嵌入的就是这个模型"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Model::alpha() 返回的运动学树 | lib.rs L36, L125–L129 |
| 触发者 | 首次调用 alpha() | LazyLock |
| 当前装配/选择/开关 | include_str!("../assets/alpha/robot_walk.xml") | lib.rs L36 |
| 实际执行者 | Model::parse(mjcf::parse(xml)) | — |
| 成功副作用与观察点 | 14 关节、8 site、trunk_height=0.12 | 本文件全文 |
| 失败是否返回且被检查 | CI fk_against_mujoco 测试覆盖 | lib.rs L17–L19 |
| 不能覆盖的对象 | ① 如果构建时 assets 路径不对，include_str! 编译失败；② 非 alpha 模型不由此文件定义 | 构建时 |
| status | **confirmed**（文件内容与 crate 引用完全对应） | — |

### 6.2 主张二："head_yaw ±170° 决定了凝视 IK 的可达包络"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | head_yaw 关节行程 | L58, head.rs L58 |
| 触发者 | look_at 求解目标 | head.rs L118 |
| 当前装配/选择/开关 | range="-2.967 2.967" rad | L58 |
| 实际执行者 | Levenberg 阻尼 + clamp | head.rs L183–L184 |
| 成功副作用与观察点 | 正后方目标 clamp 在 yaw 极限 | head.rs L299–L313 测试 |
| 失败是否返回且被检查 | clamped=true 报告未命中 | head.rs L188 |
| 不能覆盖的对象 | ① neck_pitch 和 head_pitch 也限制可达范围；② 舵机实际限位可能比 MJCF 更紧 | 机械限制 |
| status | **confirmed**（±170° 数值与 head.rs 测试期望一致） | — |

---

## 七、边界与反例（BREAK）

1. **无 geoms**：本文件只定义运动学树和惯性，没有视觉/碰撞几何体。实际碰撞检测需要另一个文件或 MuJoCo 生成的 geom。
2. **无 actuators**：没有定义伺服电机。训练策略的 actuator 定义在 mjlab_microduck 场景中。
3. **无 sensors**：没有定义 IMU/接触传感器。site 只是参考点，不自动产生传感器数据。
4. **freejoint 被忽略**：kinematics crate 不解析 freejoint（只解析 hinge）。躯干在世界中的位姿由估计器提供。
5. **关节轴全是 +z**：所有铰链轴写为 `axis="0 0 1"`，物理方向靠 body 的四元数旋转。如果 mjcf.rs 的 normalize 改变了轴，关节方向可能错误——但 mjcf.rs 只归一化不重定向。
6. **两个 IMU site**：`imu_bno` 和 `imu` 位置不同。kinematics crate 不区分它们，调用方需要知道用哪个。
7. **mouth_tip 非单位四元数**：pos="0.658 -0.020 0.752 0.023"，归一化后约为 [0.66, -0.02, 0.75, 0.02]。这是喙的倾斜方向。
8. **trunk_base pos Z=0.12**：这是训练场景中的放置高度。实际机器人站立时可能不同（取决于足端尺寸和舵机零点）。
9. **不对称 hip_yaw range**：左腿 -25°/+30°，右腿 -30°/+25°。这不是 bug，是镜像——但容易误读。
10. **no actuators means no torque limits**：MJCF 中没有 gear/ctrlrange 等 actuator 参数。实际舵机限制在别处定义。
11. **autolimits="true"**：虽然写了所有 range，但 autolimits 可能与显式 range 交互。本模型所有关节都有显式 range，所以 autolimits 不影响。
12. **惯性参数是完整六维**：fullinertia 给出 6 个惯性张量元素（Ixx, Iyy, Izz, Ixy, Ixz, Iyz），不是对角近似。

---

## 八、设计观察与工程含义

### 8.1 为什么是 alpha

这个模型是训练行走策略的 MuJoCo 场景的运动学子集。注释说"更新机械结构意味着替换这个文件并重跑 fixture 生成器——不需要改 Rust 代码"。这是一个数据驱动设计：模型是数据，不是代码。

### 8.2 为什么总质量 0.744 kg

这是一个非常小的机器人——大约 3/4 公斤。对照：一个人的头约 5kg，这里头部 0.173kg。按比例，头部占总质量 23%，说明头部容纳了主要电子设备（主板、电池、相机、ToF）。

### 8.3 为什么 head_yaw 是 ±170° 而非 ±180°

±170°（差 10°）是机械设计——舵机和结构件无法做到真正的 180°。head.rs 的 look_at 测试精确 pin 了这个极限。

### 8.4 为什么所有铰链轴都是 +z

MuJoCo 惯例：用 body 的四元数定位来旋转关节轴，而非在 joint 标签中写复杂轴。这使得树的视觉表示更清晰——每个 body 的 quat 编码了它在父帧中的方向。

### 8.5 为什么 trunk_height=0.12m

trunk_base pos Z=0.12 意味着训练场景中机器人从 12cm 高度掉到地板上。这个高度成为了 `trunk_height_m()`——tof.rs 地板过滤的基线。如果实际机器人的站立高度不同，需要覆盖 `Posture.trunk_height_m`。

---

## 九、结论（按状态分级）

### confirmed
- C1：这是 microduck alpha 的 MuJoCo MJCF 模型，110 行。
- C2：14 个铰链关节（每腿 5 + 头部 4），全部 axis=+z，方向靠 body quat。
- C3：8 个命名 site：imu_bno, imu, left_foot, head_camera, mouth_tip, tof, head_imu, right_foot。
- C4：trunk_base pos Z=0.12 → trunk_height_m=0.12。
- C5：head_yaw range=±2.967 rad=±170°。
- C6：总质量 ~0.744 kg，躯干 0.264 kg，头部 0.173 kg。
- C7：compiler angle="radian"，与 mjcf.rs 解析器一致。
- C8：无 geoms/actuators/sensors——纯运动学+惯性模型。
- C9：freejoint 被 kinematics crate 忽略。
- C10：左腿和右腿是镜像（关节名用 left_/right_ 前缀，range 对称翻转）。

### inferred
- I1：imu_bno 指 BNO055 IMU（Bosch 9 轴）。
- I2：bottom_head_shell 容纳主板和电池（质量最大的头部刚体）。
- I3：tof site 的 Y=+0.0225 表示 ToF 传感器在相机左侧 2.25cm（在 bottom_head_shell 帧中）。
- I4：训练策略在 mjlab_microduck 场景中，包含 actuators/geoms/sensors。

### unknown
- U1：训练场景的完整 MJCF（含 geoms/actuators）。
- U2：实际舵机的限位是否与 MJCF range 一致。
- U3：mouth_tip 的用途（喙尖接触？特雷门琴？）。
- U4：两个 IMU site（imu_bno vs imu）的区别和用途。
- U5：实际站立高度是否为 0.12m，还是足端尺寸使其不同。

---

## 十、未知项与最小验证动作

| 未知项 | 最小验证动作 | 预期通过信号 |
|--------|-------------|-------------|
| U1 完整训练场景 | 读取 mjlab_microduck 场景文件 | 看到 geoms/actuators/sensors |
| U2 舵机限位 | 查阅硬件文档或舵机 datasheet | 对比 MJCF range |
| U3 mouth_tip 用途 | grep -rn 'mouth_tip' src/ | 找到使用处 |
| U4 IMU 区别 | grep -rn 'imu_bno\|"imu"' src/ | 确认主备关系 |
| U5 实际站立高度 | 板上测量或 robotd 日志 | 对比 0.12m |
| FK 正确性 | cargo test --test fk_against_mujoco | 64 随机姿态通过 |

---

## 附录 A　资料来源

1. 原文件：robot_walk.xml（本地附件，110 行，全文已读）。
2. 关联文件：kinematics crate 全部 6 个源文件（lib.rs/math.rs/mjcf.rs/head.rs/tof.rs/hand.rs）。
3. 外部参考：MuJoCo MJCF XML 参考文档。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-11 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
