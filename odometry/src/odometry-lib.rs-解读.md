# odometry lib.rs（足部接触里程计）解读与架构梳理

> 分析对象：`lib.rs`（349 行），microduck 机器人的接触式里程计模块——从腿部 FK 和 IMU 估计机器人在世界坐标系中的位置和航向。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `Odometry` 模块——基于足部接触点锚定的世界位置估计器。核心思想：**任意时刻有一只脚的一个角是机器人与地面的接触点**。将该点锚定到世界（平地 Z=0），用 IMU 定向躯干，再通过正运动学（FK）反推躯干位置。当另一只脚的某个角低于锚点时（迈步落地），锚点转移到新角——估计不跳变。

**来源**：原型运行时的接触式里程计，源自 Rhoban 人形机器人 `model_service`。两个改进：
1. 腿部运动学链来自 kinematics crate 的 MJCF 模型（单一事实源），而非手写段表。
2. 每只脚的链每 tick 只算一次（4 个角变换结果），原型是每角重走整条腿链（9 次→2 次）。

**关键参数**：
- **足半长**：0.0270 m（前后）
- **足半宽**：0.0206 m（左右）
- **切换裕度**：-0.010 m（候选角需低于世界 Z=-1cm 才能竞争锚点）
- **切换确认 tick**：2 个（50Hz 下 40ms，姿态相位内，过单 tick 毛刺）
- **航向**：IMU 积分 yaw（无磁力计，世界帧是开机时朝向的相对帧）

**与之前分析的关联**：
- 依赖 `kinematics::{Model, Pose, Quat, SiteId}`——我们分析的 FK 引擎。
- 依赖 `duck_ipc_proto::JOINT_NAMES`——14 个关节名的顺序。
- 用 `Model::alpha()` LazyLock 单例——与 kinematics crate 相同的模型。
- alpha only（v1/v1.5 几何留在原型中）。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 lib.rs |
| 行数 | 349 行 |
| 模块类型 | lib（结构体 + 方法 + 测试） |
| 依赖 | duck_ipc_proto, kinematics |
| 核心类型 | Odometry |
| 调用频率 | 50Hz 控制循环每 tick 一次 |
| 已读范围 | 全文 349 行，完整阅读 |
| 版本 | alpha only |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 接触式里程计，源自 Rhoban model_service | L1–L10 | 模块文档 | confirmed |
| F2 | 锚点：一只脚的一个角贴地（世界 Z=0） | L5–L9 | 核心假设 | confirmed |
| F3 | 新角低于锚点→迈步→锚点转移 | L9–L10 | 连续不跳变 | confirmed |
| F4 | FK 链来自 kinematics MJCF 模型 | L13–L14 | 单一事实源 | confirmed |
| F5 | 性能优化：9 次链评估→2 次 | L15–L17 | 每脚一次，4 角变换结果 | confirmed |
| F6 | 航向来自 IMU 积分 yaw，无磁力计 | L19–L21 | 相对世界帧 | confirmed |
| F7 | alpha only | L23 | v1/v1.5 在原型中 | confirmed |
| F8 | SOLE_HALF_LEN=0.0270, SOLE_HALF_WIDTH=0.0206 | L31–L32 | v1.5 bbox 占位值 | confirmed |
| F9 | SWITCH_MARGIN=-0.010 | L37 | FK/IMU 噪声裕度 | confirmed |
| F10 | SWITCH_CONFIRM_TICKS=2 (40ms) | L42 | 防单 tick 毛刺 | confirmed |
| F11 | joint_map: JOINT_NAMES→model index | L52, L81 | mouth(index 9) 映射为 None | confirmed |
| F12 | pending/pending_ticks 实现时间确认 | L64–L65, L135–L151 | 候选需保持 2 tick | confirmed |
| F13 | needs_init: 首次更新锚定到启动姿态 | L67–L68, L119–L123 | 躯干从 (0,0) 开始 | confirmed |
| F14 | reproject: 躯干 = 锚点 XY - 旋转后的接触向量 | L183–L191 | Z = -contact[2] | confirmed |
| F15 | lowest_corner: 检查 8 个角（2 脚×4 角） | L195–L220 | 找最低且低于阈值的角 | confirmed |
| F16 | 测试：静止不动不漂移 | L238–L249 | 200 tick，<1e-9 | confirmed |
| F17 | 测试：mouth(index 9) 不影响里程计 | L262–L274 | 张嘴不移动 | confirmed |
| F18 | 测试：单 tick 毛刺不切换锚点 | L278–L303 | 保持 2 tick 才切 | confirmed |
| F19 | 测试：摆动腿连续平移 | L308–L347 | 每步 <2cm，总位移 >5mm | confirmed |
| F20 | angles 是 f64 而非 f32 | L54 | 高精度 FK | confirmed |

---

## 四、算法详解

### 4.1 核心思想

```
世界帧（开机时朝向为 yaw=0）
  │
  ├─ 锚点 (anchor)：某只脚的某个角，世界 Z=0，世界 XY 固定
  │    └─ 这个角是当前与地面接触的点
  │
  ├─ IMU 四元数：躯干在世界中的方向
  │
  └─ FK：从躯干到锚点的变换 → 躯干位置 = 锚点 - 旋转(接触向量)
```

### 4.2 每 tick 流程（update 方法）

```
1. 关节角从 JOINT_NAMES 顺序映射到 model 顺序（mouth 跳过）
2. IMU 四元数归一化
3. 双脚 site_pose 各算一次（2 次 FK）
4. 如果首次：锚定 anchor_xy 到当前脚的世界 XY
5. reproject：计算躯干位置
6. lowest_corner：找最低的脚角落
   ├─ 无候选：清空 pending
   ├─ 新候选：开始计数（pending_ticks=1）
   ├─ 同一只脚候选：计数+1
   └─ 计数≥2：切换锚点到新角，重新 reproject
7. yaw = IMU 四元数的 yaw 分量
```

### 4.3 四个角（CORNER）

在脚 site 坐标系中，四个角为：

| 角 | X (前后) | Y (左右) | Z |
|----|----------|----------|-----|
| +X+Y | +0.0270 | +0.0206 | 0 |
| +X-Y | +0.0270 | -0.0206 | 0 |
| -X+Y | -0.0270 | +0.0206 | 0 |
| -X-Y | -0.0270 | -0.0206 | 0 |

这些角通过 FK 变换到躯干帧，再用 IMU 旋转到世界帧。世界 Z 最低的角竞争锚点。

### 4.4 锚点切换的连续性

切换锚点时，新角的世界 XY 已经由当前 FK+IMU 计算得出（L148）。这意味着：
- 新角的世界位置已经是"对的"（它确实在地面上）。
- 躯干位置通过 reproject 重新计算，不会跳变。
- 旧锚点的 XY 被新角的 XY 替代。

**关键**：如果直接把锚点 XY 设为新角位置但不考虑当前 FK，会跳变。这里新角的 XY 是"它当前在世界中的位置"——也就是它已经落地的位置。

### 4.5 reproject 数学

```
contact_in_trunk = feet[anchor_foot].transform_point(anchor_local)
                  = 从躯干到锚点的向量（躯干帧）
contact = rot.rotate(contact_in_trunk)
         = 从躯干到锚点的向量（世界帧）
position.x = anchor_xy[0] - contact[0]
position.y = anchor_xy[1] - contact[1]
position.z = -contact[2]
```

**含义**：锚点在世界 (anchor_xy, 0)，躯干在锚点"反方向"——如果接触点在躯干前方 5cm，躯干就在锚点后方 5cm。Z 取负是因为 contact[2] 是躯干到脚底的向下距离，躯干高度 = -contact[2]。

---

## 五、Odometry 结构体

| 字段 | 类型 | 用途 |
|------|------|------|
| model | &'static Model | kinematics 模型（alpha 单例） |
| feet | [SiteId; 2] | left_foot, right_foot |
| joint_map | [Option\<usize\>; N] | JOINT_NAMES→model index（mouth=None） |
| angles | Vec\<f64\> | model 顺序的关节角（复用 buffer） |
| anchor_foot | usize | 当前锚定脚（0=左, 1=右） |
| anchor_local | [f64; 3] | 锚点在脚 site 帧中的位置 |
| anchor_xy | [f64; 2] | 锚点世界 XY |
| position | [f64; 3] | 躯干世界位置（输出） |
| yaw | f64 | 躯干航向（输出） |
| pending | Option<(usize, [f64;3], [f64;2])> | 候选锚点 |
| pending_ticks | u32 | 候选保持 tick 数 |
| needs_init | bool | 首次更新标志 |

---

## 六、测试分析

### 6.1 standing_still_stays_at_the_origin

- 200 个 tick 静止站立。
- 断言：x,y < 1e-9（无漂移），z > 0.02m（站立高度），yaw=0。
- **关键**：这证明算法没有积分漂移——静止时不产生虚假位移。

### 6.2 yaw_follows_the_imu

- IMU 持续给出 yaw=0.3 rad 的四元数。
- 断言：odo.yaw() ≈ 0.3。
- **关键**：航向直接取自 IMU，不积分（避免积分漂移）。

### 6.3 the_mouth_moves_no_odometry

- 关节 9（mouth）设为 42.0（一个假值）。
- 与不设 mouth 的结果比较——位置必须相同。
- **关键**：joint_map 将 mouth 映射为 None，不写入 angles buffer。这测试了映射的正确性。

### 6.4 the_anchor_switches_only_after_the_claim_holds

- 先稳定 10 tick。
- roll 躯干 0.35 rad（让另一只脚更低）**一个 tick**——锚点不切换。
- 然后恢复正常一个 tick。
- 断言：锚点仍在原来的脚。
- 再 roll 5 个 tick——锚点切换到另一只脚。
- **关键**：2 tick 确认机制防 FK/IMU 毛刺。

### 6.5 stance_leg_motion_translates_the_trunk_continuously

- 同时缓慢摆动左右 hip_pitch（0→0.2 rad，50 步）。
- 每 tick 断言位移 < 2cm（不跳变）。
- 总位移 > 5mm（确实移动了）。
- 所有值 finite。
- **关键**：摆动支撑腿（脚不动，躯干移动）时里程计连续，不 teleport。

---

## 七、效果主张与责任闭合卡（EFFECT）

### 7.1 主张："里程计输出无漂移且连续"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | position() 和 yaw() 输出 | L160–L167 |
| 触发者 | 50Hz 控制循环每 tick 调用 update | L104 |
| 当前装配/选择/开关 | 锚点固定角 + IMU 定向 | L5–L10 |
| 实际执行者 | reproject() | L183–L191 |
| 成功副作用与观察点 | 静止不漂移（<1e-9），连续无跳变（<2cm/tick） | 测试 L238, L308 |
| 失败是否返回且被检查 | 无 error 类型——纯计算，不返回 Result | — |
| 重试/重放来源 | SWITCH_CONFIRM_TICKS=2 防毛刺 | L42, L142 |
| 不能覆盖的对象 | ① 滑动（脚在地面滑）→锚点假设失败；② IMU 漂移→航向漂移；③ 跳跃（双脚离地）→无锚点 | — |
| status | **confirmed**（测试覆盖静止/切换/连续） | — |

---

## 八、边界与反例（BREAK）

1. **平地假设**：世界 Z=0 是平面。上坡/台阶会导致高度误差。
2. **无磁力计**：航向是 IMU 积分 yaw（实际上是直接读 IMU 四元数的 yaw 分量），长时间会漂移。
3. **脚滑不检测**：如果脚在地面上滑动（不迈步），锚点假设失效——估计会漂移。
4. **双脚离地（跳跃）**：没有脚在地面时，lowest_corner 可能返回 None，锚点不更新，但 reproject 仍在旧锚点上算——位置冻结。
5. **足尺寸是占位值**：SOLE_HALF_LEN/WIDTH 来自 v1.5 的 bbox，alpha 的 `sole_left.stl` 尚未测量。实际值可能不同。
6. **无加速度计融合**：纯运动学+IMU 方向，不融合加速度计。
7. **Z 高度 = -contact[2]**：假设锚点在 Z=0。如果脚有厚度或鞋垫，高度有偏置。
8. **f64 精度**：angles 是 f64，kinematics crate 内部也用 f64——精度足够。
9. **mouth 关节**：JOINT_NAMES 中 index 9 是 mouth（喙？），不影响腿部 FK。joint_map 跳过它。
10. **无初始化对齐**：needs_init 只在首次更新时锚定 XY，不做地面接触检测——假设启动时脚已在地面。
11. **SWITCH_MARGIN=-0.010**：候选角需要比锚点低 1cm 才竞争。这防止了"两只脚同时在地面"时的抖动——只有明显更低的角才切换。
12. **pending 只跟踪一个候选**：如果两只脚同时有角低于阈值，pending 只跟踪最后遇到的。最低角由 lowest_corner 保证。
13. **无速度估计**：此模块只输出位置和航向，不输出速度。速度由调用方差分或从其他来源获取。
14. **无回退**：没有"如果 IMU 不可用怎么办"的分支。假设 IMU 始终提供有效四元数。

---

## 九、设计观察

### 9.1 为什么用接触式而非轮式/光学里程计

这是双足机器人——没有轮子。视觉里程计需要相机和计算（在板上太重）。接触式里程计只需要关节角和 IMU——两者都已经在控制循环中。

### 9.2 为什么从 9 次链评估优化到 2 次

原型代码为每个角落重走整条腿链（4 角 × 2 脚 + 当前锚点 = 9 次）。优化后：每只脚算一次 site_pose，4 个角通过 `transform_point` 变换——这是简单的 3x3 旋转+平移，不需要重走关节链。在 50Hz 下，这从 9 次 FK 降到 2 次。

### 9.3 为什么 SWITCH_CONFIRM_TICKS=2

在 50Hz 控制循环下，2 tick = 40ms。这个时间窗口：
- 远短于一个支撑相位（通常几百 ms）——实际迈步会被检测到。
- 远长于单 tick 毛刺——FK 数值抖动或 IMU 噪声不会导致锚点抖动。

### 9.4 为什么航向直接读 IMU 而不积分

注释说"Heading is whatever the IMU's integrated yaw says"——但代码是 `self.yaw = rot.yaw()`，直接从当前四元数取 yaw。这意味着 IMU 内部已经做了积分（可能是 AHRS 融合算法），此模块只读结果。无磁力计意味着长时间航向会漂移——但对于相对运动消费者（"我往哪走了"），这足够了。

### 9.5 为什么 needs_init

如果首次更新时不锚定 anchor_xy，躯干位置会是"脚当前位置"而非 (0,0)。这意味着机器人启动时不在原点。needs_init 在第一个 tick 把锚定脚的世界 XY 作为锚点，使躯干从 (0,0) 开始。

---

## 十、与其他 crate 的关联

```
odometry
  ├── kinematics::{Model, Pose, Quat, SiteId}
  │    ├── Model::alpha()  ← robot_walk.xml（我们分析过）
  │    ├── site_pose()     ← 编译型 FK（lib.rs）
  │    ├── transform_point ← Pose（math.rs）
  │    └── Quat::yaw()     ← 四元数（math.rs）
  └── duck_ipc_proto::JOINT_NAMES
       └── 14 个关节名顺序（与 robot_walk.xml 一致）
```

**被谁调用**：
- robotd 或控制循环每 tick 调用 `Odometry::update()`。
- position() 和 yaw() 提供给导航/遥测。
- anchor_xy() 和 anchor_foot() 用于遥测显示。

---

## 十一、结论（按状态分级）

### confirmed
- C1：这是接触式足部里程计，从腿 FK + IMU 估计世界位置。
- C2：锚点是一只脚的一个角（世界 Z=0），切换需保持 2 tick（40ms）。
- C3：每 tick 算 2 次 site_pose（每脚一次），4 角通过 transform_point 变换。
- C4：航向直接读 IMU 四元数的 yaw 分量，无磁力计。
- C5：足尺寸 0.0270×0.0206 m（v1.5 占位值，alpha STL 未测）。
- C6：joint_map 将 mouth（index 9）映射为 None，不影响 FK。
- C7：5 个测试覆盖：静止不漂移、yaw 跟随 IMU、mouth 无影响、切换确认、连续平移。
- C8：f64 精度，复用 angles buffer 零分配。
- C9：alpha only。
- C10：无 Result 类型——纯计算，假设输入有效。

### inferred
- I1：被 robotd 或控制循环以 50Hz 调用。
- I2：IMU 四元数来自 BNO055（imu_bno site），已做 AHRS 融合。
- I3：足尺寸后续会被 alpha STL 测量替换。
- I4：position() 用于导航/避障，yaw() 用于方向控制。

### unknown
- U1：滑动时的行为（无脚滑检测）。
- U2：跳跃/双脚离地时的行为。
- U3：上坡/不平地面的高度误差。
- U4：IMU 长时间漂移量。
- U5：完整 crate 名（odometry?）和 Cargo.toml。

---

## 十二、未知项与最小验证动作

| 未知项 | 最小验证动作 | 预期通过信号 |
|--------|-------------|-------------|
| U1 脚滑 | 人工测试：机器人原地蹭脚 | 位置缓慢漂移（已知局限） |
| U2 跳跃 | 读取调用方代码 | 看到无锚点时的处理 |
| U3 不平地面 | 读取 tof.rs 地板过滤 | 知道是否有高度校正 |
| U4 IMU 漂移 | 板上长时间静置 | 看 yaw 是否漂移 |
| U5 crate 名 | 读取 Cargo.toml | 看到 [package] name |

---

## 附录 A　资料来源

1. 原文件：odometry/lib.rs（本地附件，349 行，全文已读）。
2. 关联文件：kinematics crate 全部 6 个源文件、robot_walk.xml、duck-ipc-proto。
3. 外部参考：Rhoban 人形机器人 model_service。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-11 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
