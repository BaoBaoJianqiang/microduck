# mjcf.rs 源码解读与架构梳理

> 分析对象：`mjcf.rs`（228 行），kinematics crate 的最小 MJCF 读取器——只解析运动学树（body/hinge joint/site），忽略 geoms/inertials/actuators/sensors/assets/defaults。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：`mjcf.rs` 是一个**最小 MJCF（MuJoCo XML）解析器**，只提取 FK 需要的三样东西：body（刚体）、hinge joint（铰链关节）、site（站点）。geoms、inertials、actuators、sensors、assets、defaults 全部忽略。树以 `<body name="trunk_base">` 为根锚定到 identity，所以 crate 产生的每个 pose 都在躯干帧中——世界放置是估计器的事，不是模型的事。

**关键设计决策**：
- **只解析运动学树**：不构建 DOM，不保留无关元素，解析结果直接是 Tree（bodies + sites + trunk_pos）。
- **trunk_base 必须存在**：没有 trunk_base 的模型被拒绝（ParseError::NoTrunkBase），因为 trunk 帧是所有 FK 的根。
- **root 的 world pos 不进入 FK**：trunk_base 的 `pos` 是 MuJoCo 把机器人放到世界的位置，trunk-frame FK 不能继承。但其 Z 分量是躯干站立高度，被提取为 `trunk_pos[2]`。
- **关节轴归一化**：零轴回退到 +z（MJCF 默认），保持解析可用而非 NaN。
- **四元数归一化**：MJCF 的 quat 属性解析后调用 normalized()，坏四元数变为 identity。
- **无名 body 拒绝，无名 site 跳过**：body 必须有名（否则无法定位），site 无名则无法被查询所以丢弃。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 mjcf.rs |
| 文件类型 | Rust 私有模块（mod mjcf，lib.rs L22） |
| 所属 crate | kinematics |
| 行数 / 已读范围 | 228 行，全文已完整读取（L1–L228） |
| 主要证据 | 文件本身；lib.rs（调用 mjcf::parse）；math.rs（Pose/Quat） |
| 不可读 / 未提供 | roxmltree 库文档、完整 MJCF 规范、alpha MJCF 资产 |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 只解析 body/hinge joint/site，忽略其他 | L1–L7 | 模块文档 | confirmed |
| F2 | 树以 trunk_base 为根，锚定 identity | L5–L7, L94–L100 | 模块文档 + 实现 | confirmed |
| F3 | 四元数是 MJCF 顺序 [w,x,y,z] | L7 | 模块文档 | confirmed |
| F4 | ParseError 有 6 种变体 | L13–L36 | enum 定义 | confirmed |
| F5 | Body 有 parent/Option、rest Pose、joint Option | L38–L45 | 结构体 | confirmed |
| F6 | Joint 有 name/axis/range Option | L47–L53 | 结构体 | confirmed |
| F7 | Site 有 name/body index/rest Pose | L55–L59 | 结构体 | confirmed |
| F8 | Tree.bodies 按 tree 顺序，parent 总是先于 child | L61–L63 | 注释 + 实现 | confirmed |
| F9 | trunk_pos 的 Z 是躯干站立高度 | L65–L68 | 注释 | confirmed |
| F10 | parse() 用 roxmltree 解析 XML | L71–L77 | 实现 | confirmed |
| F11 | 找不到 worldbody 或 trunk_base 时返回对应错误 | L73–L86 | 实现 | confirmed |
| F12 | root body 锚定 identity，joint=None | L96–L100 | 实现 | confirmed |
| F13 | walk_body 递归遍历子 body，无名 body 报错 | L112–L145 | 实现 | confirmed |
| F14 | 关节轴默认 +z，解析后归一化 | L125 | 实现 | confirmed |
| F15 | 关节 range 解析为 Option<(f64,f64)> | L126 | 实现 | confirmed |
| F16 | collect_sites 跳过无名 site | L147–L166 | 实现 | confirmed |
| F17 | rest_pose 解析 pos（默认 origin）和 quat（默认 identity，归一化） | L170–L177 | 实现 | confirmed |
| F18 | parse_floats_attr 验证分量数，不匹配报 BadVector | L189–L217 | 实现 | confirmed |
| F19 | normalize 零轴回退 +z | L219–L227 | 实现 | confirmed |

---

## 四、源码逐段解读

### 4.1 模块文档（L1–L7）

明确范围：bodies、hinge joints、sites 是 FK 需要的；geoms、inertials、actuators、sensors、assets、defaults 忽略。树以 trunk_base 为根锚定 identity，所有 pose 在躯干帧中。四元数是 MJCF 顺序 [w,x,y,z]。

### 4.2 ParseError（L13–L36）

```rust
pub enum ParseError {
    Xml(#[from] roxmltree::Error),      // XML 解析错误
    NoWorldbody,                         // 缺少 <worldbody>
    NoTrunkBase,                         // 缺少 <body name="trunk_base">
    UnnamedBody(String),                 // body 无 name
    BadFloat { tag, attr, value },       // 浮点数解析失败
    BadVector { tag, attr, expected, got }, // 向量分量数不匹配
}
```

使用 `thiserror::Error` 派生。`Xml` 变体用 `#[from]` 自动转换 roxmltree 错误。错误信息包含 tag/attr/value，便于定位问题。

### 4.3 数据结构（L38–L69）

**Body**：parent 索引（root 为 None）、rest pose（父帧中）、joint（None = 焊接）。

**Joint**：name、axis（body 帧中单位旋转轴，MJCF 默认 +z）、range（[lo,hi] 弧度，None = 无限制）。

**Site**：name、body 索引、rest pose。

**Tree**：bodies（tree 顺序，parent 先于 child）、sites、trunk_pos（trunk_base 的 world pos，Z 是站立高度）。

### 4.4 parse（L71–L110）

1. `roxmltree::Document::parse(xml)` 解析 XML。
2. 找 `<worldbody>`，找不到报 NoWorldbody。
3. 在 worldbody 下找 `<body name="trunk_base">`，找不到报 NoTrunkBase。
4. 提取 trunk_pos = parse_vec3(trunk, "pos")。
5. 创建 root body（index 0）：parent=None, rest=IDENTITY, joint=None。
6. 收集 trunk 上的 sites。
7. 递归 walk_body 遍历 trunk 的子 body。

**关键**：root 的 rest 是 IDENTITY，不是 trunk 的 MJCF pos。trunk 的 world pos 只用于提取 trunk_height（Z 分量），不进入 FK 链。

### 4.5 walk_body（L112–L145）

1. 检查 body 有 name，无则报 UnnamedBody。
2. 找第一个 `<joint>` 子元素，解析为 Joint：
   - name（默认空字符串）
   - axis（默认 [0,0,1]，归一化）
   - range（解析 2 个浮点数，None = 无限制）
3. 创建 Body，push 到 tree.bodies，获得新 index。
4. 收集该 body 上的 sites。
5. 递归遍历子 body。

注意：只取第一个 joint。如果 body 有多个 joint（MuJoCo 支持），只有第一个被解析，其余被忽略。

### 4.6 collect_sites（L147–L166）

遍历 body 的子元素，找 `<site>`。无名 site 跳过（无法被查询）。有名 site 记录 name、body index、rest pose。

### 4.7 rest_pose（L170–L177）

解析 `pos`（默认 [0,0,0]）和 `quat`（默认 IDENTITY，解析后 normalized）。每个可放置的 MJCF 元素（body/site）都携带这对属性。

### 4.8 辅助函数（L179–L227）

- **element_children**：过滤出元素子节点（跳过文本/注释）。
- **parse_vec3**：解析 3 分量向量。
- **parse_floats_attr**：通用浮点向量解析，按空白分割，验证分量数。
- **normalize**：归一化 3D 向量，零向量回退 +z（MJCF 默认），保持解析可用而非 NaN。

---

## 五、解析流程（ROUTE）

```
MJCF XML 字符串
    → roxmltree::Document::parse
    → 找 worldbody → 找 trunk_base
    → 创建 root body (identity, no joint)
    → 收集 trunk sites
    → 递归 walk_body:
        检查 name → 解析 joint → 创建 Body → 收集 sites → 递归子 body
    → Tree { bodies, sites, trunk_pos }
    → lib.rs Model::parse 编译为扁平链
```

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："解析结果的所有 pose 在躯干帧中"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Tree 中所有 body/site 的 rest pose | L94–L100, L132–L136 |
| 触发者 | parse() | 外部 |
| 当前装配/选择/开关 | root 锚定 identity，子 body 的 rest 是相对于 parent 的 | L96–L100 |
| 实际执行者 | walk_body 递归 | — |
| 成功副作用与观察点 | FK 结果在 trunk 帧中 | lib.rs 测试 L234–L239 |
| 失败是否返回且被检查 | 无 trunk_base 时返回 NoTrunkBase | L79–L86 |
| 不能覆盖的对象 | ① trunk_base 的 world pos X/Y 被丢弃；② 如果 MJCF 有多个 trunk_base，只取第一个 | 实现限制 |
| status | **confirmed**（root identity + 相对 rest = trunk 帧） | — |

### 6.2 主张二："坏输入不产生 NaN"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | 零轴、坏四元数、坏浮点 | L173, L219–L227 |
| 触发者 | 解析包含坏属性的 MJCF | 外部 |
| 当前装配/选择/开关 | 零轴→+z，坏四元数→identity，坏浮点→Err | L173, L221–L224, L201–L205 |
| 实际执行者 | normalize() / normalized() / parse() | — |
| 成功副作用与观察点 | 解析成功，FK 结果有限 | 需测试 |
| 失败是否返回且被检查 | 坏浮点返回 Err(BadFloat)，不静默 | L201–L205 |
| 不能覆盖的对象 | ① 零轴回退 +z 可能掩盖模型错误；② 坏四元数回退 identity 可能掩盖模型错误 | 设计权衡 |
| status | **confirmed**（回退逻辑 confirmed；是否掩盖错误需诊断） | — |

---

## 七、边界与反例（BREAK）

1. **只取第一个 joint**：body 有多个 joint 时只有第一个被解析，其余静默忽略。MuJoCo 支持多关节 body，但这个解析器不支持。
2. **不支持非铰链关节**：slide/ball/free joint 被当作 hinge 解析（axis 无意义），或被忽略。
3. **无名 body 拒绝**：但 MuJoCo 允许无名 body（匿名中间帧）。这个解析器要求所有 body 有名。
4. **无名 site 静默跳过**：如果 MJCF 有无名 site，不会出现在结果中，也不会报错。
5. **不解析 defaults**：MJCF 的 `<default>` 节可以设置默认 pos/quat/axis/range，这个解析器不处理，所有默认值硬编码（pos=origin, quat=identity, axis=+z, range=None）。
6. **不解析 compiler 角度约定**：MJCF 的 `<compiler angle="degree"/>` 会让所有角度用度，但这个解析器假设弧度。
7. **trunk_pos 只提取 Z**：X/Y 被丢弃。如果调用方需要世界放置，必须自己解析。
8. **roxmltree 不支持 DTD/entity**：如果 MJCF 包含 DTD 声明或实体引用，可能解析失败。
9. **关节 name 默认为空字符串**：`j.attribute("name").unwrap_or("")`——无名关节不会报错，但会导致 joint_index 查找失败。
10. **range 不验证 lo <= hi**：如果 MJCF 写反了 range，解析器不检查，IK clamp 可能行为异常。
11. **不支持 euler/zaxis 旋转**：MJCF 允许用 euler 或 zaxis 替代 quat 指定姿态，这个解析器只认 quat。
12. **site 的 pos/quat 之外的属性被忽略**：site 可以有 size/rgba/type 等，全部忽略。

---

## 八、结论（按状态分级）

### confirmed
- C1：只解析 body/hinge joint/site，忽略其他 MJCF 元素。
- C2：树以 trunk_base 为根，锚定 identity，所有 pose 在躯干帧。
- C3：root 的 world pos 不进入 FK，Z 分量提取为 trunk_height。
- C4：ParseError 有 6 种变体，错误信息含 tag/attr/value。
- C5：关节轴默认 +z，解析后归一化，零轴回退 +z。
- C6：四元数解析后 normalized，坏四元数→identity。
- C7：无名 body 拒绝（Err），无名 site 跳过（静默）。
- C8：只取 body 的第一个 joint。
- C9：浮点向量解析验证分量数，不匹配报 BadVector。
- C10：Tree.bodies 按 tree 顺序，parent 先于 child。

### inferred
- I1：roxmltree 是零拷贝 XML 解析器，解析结果的生命周期绑定到输入字符串。
- I2：alpha MJCF 不包含 defaults 节（否则解析结果会错）。
- I3：alpha MJCF 所有角度用弧度（MJCF 默认）。

### unknown
- U1：alpha MJCF 是否包含多关节 body。
- U2：alpha MJCF 是否使用 euler/zaxis 而非 quat。
- U3：roxmltree 对大 MJCF 文件的解析性能。
- U4：是否有 MJCF 验证工具（xmllint 等）。

---

## 附录 A　资料来源

1. 原文件：mjcf.rs（本地附件，228 行，全文已读）。
2. 同 crate 文件：lib.rs（调用方）、math.rs（Pose/Quat）。
3. 外部参考：MuJoCo XML 参考、roxmltree 文档。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-10 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
