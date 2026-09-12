# mjcf.rs 文件解析

## 文件位置

`d:\microduck\kinematics\src\mjcf.rs`

## 核心设计决策

最小 MJCF 读取器：只要运动学树，别的都不要。

FK 需要的是 body、铰链关节（hinge joint）和 site；geom、inertial、actuator、sensor、asset、default 全部忽略。树以 `<body name="trunk_base">` 为根并锚定为 identity，因此本 crate 产生的每个 pose 都在 trunk 帧中——世界放置是估计器的事，不是模型的事。四元数是 MJCF 顺序 `[w, x, y, z]`。

## 错误类型 `ParseError`

- `Xml` — roxmltree 解析错误
- `NoWorldbody` — 缺少 `<worldbody>`
- `NoTrunkBase` — 缺少 `<body name="trunk_base">`
- `UnnamedBody(String)` — body 无 name 属性
- `BadFloat` — 浮点解析失败（含 tag/attr/value）
- `BadVector` — 向量分量数不对（含 expected/got）

## 内部类型

- `Body` — `parent: Option<usize>`（root 为 `None`）、`rest: Pose`（父帧中的 rest pose，root 为 identity）、`joint: Option<Joint>`
- `Joint` — `name`、`axis: [f64; 3]`（body 帧中单位旋转轴，MJCF 默认 +z）、`range: Option<(f64, f64)>`
- `Site` — `name`、`body: usize`、`rest: Pose`
- `Tree` — `bodies: Vec<Body>`（树顺序：父总在子前）、`sites: Vec<Site>`、`trunk_pos: [f64; 3]`（场景把 `trunk_base` 放入世界的位置，FK 忽略它，但其 Z 是躯干离地高度）

## 关键函数

- `parse(xml)` — 找 `worldbody` → 找 `trunk_base` → root 锚定 identity（不继承 MJCF `pos`）→ 收集 trunk 的 site → 递归 `walk_body` 子 body
- `walk_body(node, parent, tree)` — 要求 body 有 name；解析可选 joint（axis 默认 `[0,0,1]` 并归一化，range 可选）；入栈 body；收集 site；递归子 body
- `collect_sites(node, body, sites)` — 收集 `<site>`，无 name 的 site 跳过（无法被查询）
- `rest_pose(node)` — `pos`（默认原点）+ `quat`（默认 identity，解析后归一化）
- `parse_vec3` / `parse_floats_attr(node, attr, expected)` — 解析浮点向量属性，缺省返回 `None`，分量数错误报 `BadVector`
- `normalize(v)` — 单位化；零轴回退 `[0,0,1]`（MJCF 自身默认），保持解析可用且错误在 FK 中可见而非 NaN

## 关键摘要

`mjcf.rs` 是极简 MJCF 解析器，只提取 FK 所需的 body/hinge joint/site 三件套，以 `trunk_base` 为 identity 根。所有其他 MJCF 元素（geom、inertial 等）一律忽略。错误信息精确到 tag/attr/value，便于定位坏 XML。
