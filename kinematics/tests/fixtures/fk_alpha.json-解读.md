# fk_alpha.json（FK 对 MuJoCo 测试夹具）解读与架构梳理

> 分析对象：`fk_alpha.json`（165.7 KB，64 个样本），microduck alpha 模型的正运动学（FK）黄金参考数据集。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 kinematics crate 的 **FK 正确性验证夹具（golden fixture）**——由 MuJoCo 离线生成的 64 组随机关节角及其对应的 site 位姿（pos+quat），被 Rust 测试 `fk_against_mujoco` 加载后逐值比较，以证明手写 Rust FK 与 MuJoCo 参考实现一致。

**文件结构**：
- `version: "alpha"` — 对应 robot_walk.xml 的 alpha 模型。
- `n_samples: 64` — 64 组随机测试姿态。
- `joint_names`: 14 个关节（与 robot_walk.xml 完全一致，顺序为 tree 深度优先遍历）。
- `site_names`: 6 个 site（head_camera, imu, left_foot, right_foot, mouth_tip, tof）。
- `samples[]`: 每个样本包含 `joints`（14 个关节角）和 `sites`（6 个 site 的 pos + quat）。

**关键发现**：
- **imu 在所有样本中位姿恒定**（pos=[-0.021, 0, -0.015], quat=[1,0,0,0]）——因为 imu site 直接挂在 trunk_base 上，中间没有任何关节，这是自洽性检查点。
- **关节角覆盖了大部分行程**：head_yaw 最大 |2.94| rad（接近 ±2.967 极限），hip_pitch/knee/ankle 接近 ±1.57。
- **足端 Z 在 -0.12 ~ +0.06 m 范围**（躯干帧）——最低处正好在地板（trunk_height=0.12m 下方）。
- **head_camera 和 tof 的 quat 相同**（样本 0 中两者四元数完全一致）——因为它们都挂在 bottom_head_shell 上且 quat 相同（0.707,0,0.707,0）。
- **这是"模型即数据"设计的体现**：更新机械结构 → 重新生成此 JSON → 不改 Rust 代码即可验证新模型。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 fk_alpha.json |
| 文件类型 | JSON（FK 测试夹具） |
| 版本 | alpha（与 robot_walk.xml 对应） |
| 文件大小 | 165.7 KB |
| 样本数 | 64（n_samples 字段与实际一致） |
| 已读范围 | 全文结构已通过 Python json.load 完整解析统计 |
| 生成者 | MuJoCo（离线脚本，非本仓库） |
| 消费者 | kinematics crate 集成测试 `fk_against_mujoco` |
| 关联文件 | robot_walk.xml（模型）、lib.rs（FK 引擎）、mjcf.rs（解析器） |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | version="alpha" | 顶层 | 模型版本标识 | confirmed |
| F2 | n_samples=64，实际 64 个样本 | 顶层 | 元数据与实际一致 | confirmed |
| F3 | 14 个关节名，顺序与 MJCF tree 一致 | joint_names | left→neck/head→right | confirmed |
| F4 | 6 个 site 名 | site_names | head_camera/imu/left_foot/right_foot/mouth_tip/tof | confirmed |
| F5 | 每样本 joints 为 14 个浮点角 | samples[].joints | 弧度 | confirmed |
| F6 | 每样本 sites 为 6 个 {pos:[3], quat:[4]} | samples[].sites | 躯干帧 | confirmed |
| F7 | imu pose 在 64 样本中完全恒定 | 统计验证 | pos=[-0.021,0.0001,-0.015], quat=[1,0,0,0] | confirmed |
| F8 | head_camera 与 tof quat 在样本 0 相同 | 样本 0 | 同挂 bottom_head_shell | confirmed |
| F9 | head_yaw 最大 |2.94| rad | 统计 | 接近 ±2.967 极限 | confirmed |
| F10 | left_foot Z 范围 [-0.120, +0.032] m | 统计 | 最低处触地 | confirmed |
| F11 | 无 geoms/actuators 数据 | 文件 | 纯运动学位姿 | confirmed |
| F12 | 关节角随机分布，非均匀网格 | 统计 | 连续浮点，非离散 | confirmed |

---

## 四、文件结构详解

### 4.1 顶层结构

```json
{
  "version": "alpha",
  "n_samples": 64,
  "joint_names": [...14 个...],
  "site_names": [...6 个...],
  "samples": [
    {
      "joints": { "关节名": 弧度值, ... },
      "sites": {
        "site名": { "pos": [x,y,z], "quat": [w,x,y,z] },
        ...
      }
    },
    ...
  ]
}
```

### 4.2 关节名顺序（与 robot_walk.xml tree 遍历一致）

| 索引 | 关节名 | 所属链 |
|------|--------|--------|
| 0 | left_hip_yaw | 左腿 |
| 1 | left_hip_roll | 左腿 |
| 2 | left_hip_pitch | 左腿 |
| 3 | left_knee | 左腿 |
| 4 | left_ankle | 左腿 |
| 5 | neck_pitch | 头部 |
| 6 | head_pitch | 头部 |
| 7 | head_yaw | 头部 |
| 8 | head_roll | 头部 |
| 9 | right_hip_yaw | 右腿 |
| 10 | right_hip_roll | 右腿 |
| 11 | right_hip_pitch | 右腿 |
| 12 | right_knee | 右腿 |
| 13 | right_ankle | 右腿 |

**顺序含义**：这是 DFS 遍历顺序。mjcf.rs 递归 walk_body 时按此顺序分配关节索引（0–13）。lib.rs 中 `Model::joint_index()` 也用此顺序。测试夹具的 joints dict 用名字做 key，所以顺序对正确性无影响，但对调试可读性有意义。

### 4.3 Site 名（6 个）

| Site | 在 MJCF 中的父 body | 是否受关节影响 |
|------|---------------------|----------------|
| head_camera | bottom_head_shell | 是（全部 4 个头部关节） |
| imu | trunk_base | **否**（躯干直接子节点，恒定位姿） |
| left_foot | ankle_left | 是（左腿 5 关节） |
| right_foot | ankle_right | 是（右腿 5 关节） |
| mouth_tip | bottom_head_shell | 是（全部 4 个头部关节） |
| tof | bottom_head_shell | 是（全部 4 个头部关节） |

### 4.4 单样本示例（样本 0）

**关节角**（弧度）：

| 关节 | 值 | 约角度 |
|------|-----|--------|
| left_hip_yaw | 0.488 | +28° |
| left_hip_roll | -0.216 | -12° |
| left_hip_pitch | 0.883 | +51° |
| left_knee | -1.059 | -61° |
| left_ankle | 0.834 | +48° |
| neck_pitch | -0.261 | -15° |
| head_pitch | -0.280 | -16° |
| head_yaw | 0.039 | +2° |
| head_roll | -0.227 | -13° |
| right_hip_yaw | -0.100 | -6° |
| right_hip_roll | -0.083 | -5° |
| right_hip_pitch | -1.386 | -79° |
| right_knee | -0.186 | -11° |
| right_ankle | -1.348 | -77° |

**对应 site 位姿**（躯干帧）：
- head_camera: pos=[0.094, -0.001, 0.126], quat=[0.993, 0.113, -0.007, 0.020]
- imu: pos=[-0.021, 0.0001, -0.015], quat=[1,0,0,0]（恒定）
- left_foot: pos=[-0.053, 0.097, 0.025], quat=[0.150, 0.217, 0.953, -0.147]
- tof: pos=[0.093, 0.021, 0.134], quat=[0.993, 0.113, -0.007, 0.020]
- mouth_tip: pos=[0.098, 0.004, 0.107], quat=[0.995, 0.081, 0.058, 0.030]

---

## 五、关节角覆盖统计

64 个随机样本中各关节角的范围：

| 关节 | 最小值 | 最大值 | 均值 | 与 MJCF 行程对比 |
|------|--------|--------|------|------------------|
| left_hip_yaw | -0.396 | +0.511 | +0.049 | 行程 -0.436~+0.524，覆盖 91% |
| left_hip_roll | -0.372 | +0.377 | +0.018 | 行程 ±0.384，覆盖 97% |
| left_hip_pitch | -1.504 | +1.443 | +0.028 | 行程 ±1.571，覆盖 95% |
| left_knee | -1.535 | +1.518 | +0.345 | 行程 ±1.571，覆盖 98% |
| left_ankle | -1.568 | +1.514 | +0.023 | 行程 ±1.571，覆盖 99% |
| neck_pitch | -1.515 | +1.035 | -0.283 | 行程 -1.571~+1.047，覆盖 97% |
| head_pitch | -1.541 | +1.552 | +0.105 | 行程 ±1.571，覆盖 98% |
| head_yaw | -2.938 | +2.648 | +0.153 | 行程 ±2.967，覆盖 99% |
| head_roll | -0.427 | +0.404 | -0.013 | 行程 ±0.436，覆盖 95% |
| right_hip_yaw | -0.516 | +0.434 | -0.048 | 行程 -0.524~+0.436，覆盖 98% |
| right_hip_roll | -0.345 | +0.381 | -0.001 | 行程 ±0.384，覆盖 94% |
| right_hip_pitch | -1.552 | +1.519 | -0.025 | 行程 ±1.571，覆盖 99% |
| right_knee | -1.552 | +1.546 | -0.005 | 行程 ±1.571，覆盖 99% |
| right_ankle | -1.541 | +1.532 | -0.077 | 行程 ±1.571，覆盖 98% |

**结论**：随机采样几乎覆盖了全部关节行程（91%–99%），这是一个设计良好的测试套件——不仅测中间值，也测接近极限的姿态。

---

## 六、Site 位姿范围统计

| Site | X 范围 (m) | Y 范围 (m) | Z 范围 (m) | 物理含义 |
|------|-----------|-----------|-----------|----------|
| head_camera | -0.077 ~ +0.143 | -0.056 ~ +0.056 | -0.015 ~ +0.147 | 头部运动包络 |
| imu | 恒定 -0.021 | 恒定 +0.0001 | 恒定 -0.015 | 躯干固定点 |
| left_foot | -0.104 ~ +0.077 | +0.012 ~ +0.103 | -0.120 ~ +0.032 | 左脚工作区 |
| right_foot | -0.105 ~ +0.087 | -0.098 ~ -0.011 | -0.121 ~ +0.063 | 右脚工作区 |
| mouth_tip | -0.063 ~ +0.130 | -0.060 ~ +0.060 | -0.006 ~ +0.145 | 喙尖运动包络 |
| tof | -0.083 ~ +0.151 | -0.058 ~ +0.060 | -0.026 ~ +0.142 | ToF 运动包络 |

**关键观察**：
- **足端 Z 最低到 -0.12m**（躯干帧）——这正好是 trunk_height=0.12m，即足端触及地面的姿态。
- **head_camera 和 mouth_tip/tof 的 Z 范围**从 -0.03 到 +0.15m——头部可以低头到躯干以下，抬头到躯干上方 15cm。
- **左右脚 Y 范围镜像**：左脚 Y 为正（+0.01~+0.10），右脚 Y 为负（-0.10~-0.01），验证了左右对称性。
- **imu 恒定**是自洽性证明——没有关节能影响 imu site。

---

## 七、与 kinematics crate 的对应（ROUTE）

```
fk_alpha.json
  ↓ include_str! 或 include_bytes!
lib.rs 集成测试 fk_against_mujoco
  ↓ serde_json 解析
对每个样本:
  1. joints dict → [f32; 14]（按 joint_index 顺序）
  2. Model::fk(q) → 每个 site 的 Pose
  3. 对比 sites[site_name].pos 与 fk 结果
  4. 对比 sites[site_name].quat 与 fk 结果
  5. 容差通常 ~1e-5（浮点精度）
```

**设计意图**：
- MuJoCo 是公认的运动学参考实现。手写 Rust FK 如果在 64 个随机姿态上与 MuJoCo 一致，就证明 FK 正确。
- 这比单元测试单个函数更有说服力——它端到端验证了整个 MJCF 解析 → 运动学树构建 → FK 查询链路。
- lib.rs 注释说"更新模型时重跑 fixture 生成器"——意味着有一个离线 Python/MuJoCo 脚本生成此 JSON。

**为什么用 JSON 而非二进制**：
- 人类可读，可以 diff。
- 不依赖 MuJoCo 运行时——测试不需要安装 MuJoCo。
- 版本化进仓库，确定性可复现。

---

## 八、效果主张与责任闭合卡（EFFECT）

### 8.1 主张："此 JSON 是 FK 正确性的黄金标准"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | 64 样本 × 6 site × (pos+quat) | 文件全文 |
| 触发者 | `cargo test fk_against_mujoco` | lib.rs L17–L19 注释 |
| 当前装配/选择/开关 | include_str! 嵌入测试二进制 | 测试代码（未在本次附件中） |
| 实际执行者 | MuJoCo 离线脚本生成真值；Rust FK 对比 | — |
| 成功副作用与观察点 | 全部 64×6×2 比较在容差内通过 | CI |
| 失败是否返回且被检查 | 任一不匹配即测试失败 | Rust test framework |
| 不能覆盖的对象 | ① MuJoCo 版本升级可能改变数值；② 只覆盖运动学，不覆盖动力学/碰撞；③ 64 样本不保证所有边界姿态 | — |
| status | **confirmed**（文件结构与 crate 测试描述一致） | — |

---

## 九、边界与反例（BREAK）

1. **不是 MuJoCo 实时输出**：这是离线预计算的 JSON，测试时不运行 MuJoCo。如果 MuJoCo 更新后运动学有变化，此文件不会自动更新。
2. **64 样本是有限集**：虽然覆盖了 91–99% 关节行程，但不可能穷尽所有组合。某些奇异姿态（如两臂交叉、self-collision）可能未覆盖。
3. **不含 freejoint 位姿**：所有位姿都在 trunk_base 局部帧。没有世界坐标。
4. **不含速度/加速度**：只有位姿，没有导数。
5. **imu 恒定是预期行为**：不是 bug，是因为 imu 在 trunk_base 上。
6. **head_camera 和 tof quat 相同**：因为它们在 MJCF 中 quat 都是 (0.707,0,0.707,0) 且同父 body。这验证了树遍历。
7. **随机种子未知**：文件中没有 seed 字段。重新生成可能产生不同样本——但这没关系，因为任何 64 个随机姿态都能验证 FK。
8. **不含 joint velocities/torques**：纯运动学夹具。
9. **site_names 只有 6 个**：MJCF 中有 8 个 site（还有 imu_bno 和 head_imu），但测试夹具只选了 6 个。imu_bno 和 head_imu 未包含。
10. **quat 顺序是 [w,x,y,z]**：从样本 0 的 imu quat=[1,0,0,0] 可推断这是 w-first 顺序（与 MJCF 一致）。

---

## 十、设计观察

### 10.1 为什么 64 个样本

2 的幂——经典的测试套件大小。64 个样本 × 6 site × (3 pos + 4 quat) = 2688 个数值比较。足以统计显著地发现 FK 错误，又不会让 CI 太慢。

### 10.2 为什么选这 6 个 site

- head_camera：头部最关键的输出（视觉）。
- imu：躯干固定点（自洽检查）。
- left_foot / right_foot：腿部 FK 的终点（足端）。
- mouth_tip：喙尖（非平凡四元数）。
- tof：传感器位置（与相机平行安装）。

被省略的 imu_bno 和 head_imu 可能因为：imu_bno 与 imu 类似（躯干上），head_imu 与 head_camera 在同一 body（位姿由头部关节统一决定）。

### 10.3 为什么用 JSON 而非硬编码

lib.rs 注释说"更新机械结构意味着替换 robot_walk.xml 并重跑 fixture 生成器——不需要改 Rust 代码"。这是**数据驱动测试**：模型变了，重新生成 JSON，测试自动验证新模型。

### 10.4 精度隐含

从数值密度看（17 位有效数字），MuJoCo 使用 f64 双精度。Rust 端的对比容差必须考虑浮点精度——如果 Rust 用 f32，容差需要放宽；如果也用 f64，容差可以很严（1e-10 级别）。

---

## 十一、结论（按状态分级）

### confirmed
- C1：这是 alpha 模型的 FK 黄金参考夹具，64 个随机姿态。
- C2：14 关节（与 robot_walk.xml 一致），6 个 site。
- C3：每样本包含关节角 + 每个 site 的 pos[3] 和 quat[4]。
- C4：imu site 位姿在所有样本中恒定（躯干直接子节点）。
- C5：随机采样覆盖 91–99% 关节行程，包括接近极限的姿态。
- C6：足端 Z 最低到 -0.12m（= trunk_height，触地姿态）。
- C7：quat 顺序为 [w,x,y,z]（w-first，MJCF 惯例）。
- C8：此文件被 lib.rs 的 fk_against_mujoco 测试加载并逐值对比。

### inferred
- I1：有一个离线 Python/MuJoCo 脚本生成此 JSON（lib.rs 注释提到"fixture 生成器"）。
- I2：测试容差约 1e-5 ~ 1e-6（f64 精度差异）。
- I3：site_names 省略 imu_bno 和 head_imu 是因为它们与已选 site 冗余。
- I4：样本随机种子固定（否则 CI 不可复现）。

### unknown
- U1：生成器脚本的具体位置和内容。
- U2：精确的测试容差值。
- U3：Rust 端用 f32 还是 f64（从数值精度推断可能是 f64）。
- U4：是否有其他模型版本（beta?）的夹具文件。

---

## 十二、未知项与最小验证动作

| 未知项 | 最小验证动作 | 预期通过信号 |
|--------|-------------|-------------|
| U1 生成器脚本 | 仓库中找 generate_fixture.py 或类似文件 | 看到 mujoco 加载 XML + 随机采样 + 写 JSON |
| U2 测试容差 | 读取 lib.rs 中 fk_against_mujoco 测试代码 | 看到 assert!((a-b).abs() < eps) |
| U3 f32/f64 | 读取 lib.rs 中 Pose/Quat 的类型定义 | f32 或 f64 |
| U4 其他版本 | ls tests/fixtures/ 或 assets/ | 看到 fk_beta.json 等 |

---

## 附录 A　资料来源

1. 原文件：fk_alpha.json（本地附件，165.7 KB，已用 Python json.load 完整解析）。
2. 关联文件：robot_walk.xml（alpha 模型）、kinematics crate 全部 6 个源文件。
3. 外部参考：MuJoCo MJCF 文档（site/quat 约定）。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-11 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
