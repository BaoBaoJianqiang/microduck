# `Cargo.toml` 解读 — kinematics

> 文件路径：`kinematics/Cargo.toml`
> 角色：duck 的前向运动学（FK），由 MJCF（MuJoCo XML）本身驱动

---

## 一、包级文档注释（这个 crate 存在的理由）

```toml
# Forward kinematics for the duck, driven by the MJCF itself.
#
# The kinematic tree is parsed out of the same `robot_walk.xml` the policies are
# trained against, so a mechanical revision means updating one XML file, not
# transcribing joint offsets into Rust by hand. Absorbed from the
# `microduck_kinematics_rs` satellite repo, reworked for the control-loop hot
# path: joints are indices into a slice rather than names in a `HashMap`, and a
# site query walks its own precompiled chain instead of recomputing every body
# in the tree. Validated against MuJoCo's `mj_kinematics` by fixture.
```

这段注释是整个 crate 的设计宣言，逐句解读：

### 1. "由 MJCF 本身驱动"

前向运动学（FK）的运动树不是硬编码在 Rust 中的常量，而是从 `robot_walk.xml`（MuJoCo MJCF 格式）解析出来的。这意味着模型定义和运动学实现共享同一个事实来源（single source of truth）。

### 2. "机械修订意味着更新一个 XML 文件，而非手动转录关节偏移到 Rust"

这是核心设计动机。如果关节偏移（零位、方向、轴）硬编码在 Rust 中，每次机械修改（换关节、改零位、换连杆）都需要：
1. 修改 `robot_walk.xml`（仿真和训练用）
2. 手动找到 Rust 中对应的硬编码常量并修改
3. 手动验证两者一致

这两步之间必然漂移——仿真中改了，Rust 中忘了改，或者改了但不一致。从 XML 解析消除了第二步。

### 3. "从 `microduck_kinematics_rs` 卫星仓库吸收，为控制循环热路径重做"

这个 crate 原本是一个独立的卫星仓库（satellite repo），现在被吸收（absorbed）到主仓库中。吸收时做了关键优化：

- **关节是切片索引而非 `HashMap` 中的名称**：原来按名称查关节（`HashMap<String, Joint>`），每次 FK 都要字符串查找。改为编译期确定的切片索引（`joints[id as usize]`），零查找开销。
- **站点查询走自己的预编译链**：原来每次站点查询都重算整棵树中每个 body 的变换。改为预编译（precompile）每个站点的运动链——从该站点到根 body 的路径被缓存为索引数组，查询时只走这条链，不遍历整棵树。

这些优化是因为 FK 在控制循环的热路径上（50Hz，每帧都要算），任何额外开销都直接影响控制频率。

### 4. "通过 fixture 对 MuJoCo 的 `mj_kinematics` 验证"

正确性通过 fixture 测试保证：用 MuJoCo 的 `mj_kinematics`（参考实现）在一组标准姿态下计算所有 body 和站点的世界变换，然后用这些结果作为 fixture 断言自实现的 FK 输出完全一致。

这是一种"参考实现对拍"策略——不相信自己的数学，让 MuJoCo 当裁判。

---

## 二、[package] 元数据

```toml
[package]
name = "kinematics"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `kinematics` | crate 名 |
| `version` | workspace | 从工作区根继承 |
| `edition` | workspace | 从工作区根继承 |
| `rust-version` | workspace | 从工作区根继承 |
| `license` | workspace | 从工作区根继承 |

---

## 三、[dependencies] 运行时依赖

### 3.1 `roxmltree = "0.21"`

```toml
roxmltree = "0.21"
```

**只读 XML 树解析库。**

- 选择 `roxmltree` 而非 `quick-xml`/`xmltree`/`roxml` 等的理由：
  - **只读**：MJCF 是输入，不需要写回。`roxmltree` 构建不可变树后丢弃原始输入，内存效率高。
  - **无额外依赖**：`roxmltree` 自身零外部依赖（纯 Rust），不引入 `encoding_rs`、`libxml2` 等。
  - **API 简洁**：DOM 风格的树遍历（`descendants()`、`children()`、`attribute()`），适合一次性解析 MJCF 后构建内部表示。
  - **为什么不用 `quick-xml`**：`quick-xml` 是流式解析器（SAX 风格），需要自己维护栈状态。MJCF 解析是启动时一次性操作，DOM 风格更直接，且树不大（机器人关节数有限）。
  - **为什么不用 `serde_xml_rs`/`quick-xml`+serde**：MJCF 不是配置文件——它是仿真格式，包含惯性张量、关节轴、范围等需要特殊处理的字段。用 serde 反序列化会引入大量样板代码和字段映射错误的风险。

### 3.2 `thiserror.workspace = true`

```toml
thiserror.workspace = true
```

**错误类型 derive 宏。**

- 定义解析错误（XML 格式不对、关节缺失、惯性张量维度错误等）的枚举。
- `thiserror` 用于库级错误类型（`#[error("...")]`），`anyhow` 用于应用级错误传播。这个 crate 是库，所以用 `thiserror` 而非 `anyhow`。
- 从工作区继承版本，确保全仓库一致。

---

## 四、[dev-dependencies] 开发依赖

### 4.1 `serde = { workspace = true }`

```toml
serde = { workspace = true }
```

仅在测试中使用。用于序列化 fixture 数据（MuJoCo 参考输出）为 JSON 文件，以及反序列化断言。

### 4.2 `serde_json.workspace = true`

```toml
serde_json.workspace = true
```

仅在测试中使用。fixture 文件格式为 JSON，存储 MuJoCo `mj_kinematics` 的参考输出（各 body/站点在标准关节角度下的世界变换矩阵）。

**为什么 fixture 是 JSON 而非硬编码在 Rust 中**：
- fixture 数据量大（多个姿态 × 多个 body × 4×4 矩阵），硬编码在 Rust 中不可读。
- JSON 文件可以由 MuJoCo Python 脚本生成，人与机器人语言无关。
- `serde`/`serde_json` 仅在 `dev-dependencies` 中，不影响发布版二进制大小。

---

## 五、依赖关系图

```
kinematics (库)
├── roxmltree 0.21        — 只读 XML 树解析（MJCF 解析）
└── thiserror (workspace) — 错误类型

dev
├── serde (workspace)     — fixture 序列化
└── serde_json (workspace)— fixture JSON 读写
```

---

## 六、设计要点总结

### 6.1 最小依赖原则

这个 crate 只有两个运行时依赖（`roxmltree` + `thiserror`），没有：
- **没有 `nalgebra`/`glam`**：矩阵运算用原始 `[f64; 16]`（行优先 4×4）或手写的旋转/平移函数。FK 是热路径，引入大型线性代数库会增加编译时间和二进制大小，而 FK 只需要少数几种矩阵操作（乘、平移、旋转）。
- **没有 `mujoco` crate**：不链接 MuJoCo C 库。MuJoCo 只在测试时作为参考（通过 Python 脚本生成 fixture），运行时完全自包含。
- **没有 `log`/`tracing`**：FK 是纯计算，没有副作用，不需要日志。错误通过 `Result` 返回。
- **没有 `serde`**：运行时不需要序列化，只有测试需要。

### 6.2 启动时解析，运行时纯计算

- MJCF XML 在 `Model::from_xml()`（或类似入口）中一次性解析
- 解析后构建内部表示：预编译的运动链、关节索引表、站点链
- 运行时 `fk()`/`site_world()` 只做浮点运算，不做字符串查找、不做 XML 遍历

### 6.3 控制循环热路径优化

| 优化 | 原来 | 现在 |
|------|------|------|
| 关节查找 | `HashMap<String, Joint>` 字符串查找 | 切片索引 `joints[id]` |
| 站点查询 | 重算整棵树所有 body | 走预编译链（只走该站点到根的路径） |
| 矩阵运算 | 通用 `nalgebra` | 原始数组 + 手写函数 |

### 6.4 正确性保证

- **单一事实来源**：运动树从策略训练用的同一个 `robot_walk.xml` 解析，机械修改只改一处
- **MuJoCo 对拍**：fixture 测试用 MuJoCo `mj_kinematics` 的输出作为参考
- **零运行时依赖 MuJoCo**：发布版不链接 `libmujoco.so`，只在测试时需要（通过 Python 脚本预生成 fixture）

---

## 七、与其他 crate 的关系

- **`duck-control`**：消费这个 crate。控制循环在每个 tick 调用 FK 将关节角度转为 body/站点世界坐标，用于策略观测构建和安全检查。
- **`robotd`**：策略推理的宿主进程，也间接消费运动学（通过 duck-control）。
- **`robot_walk.xml`**：MJCF 模型文件，是策略训练（MuJoCo）和运动学解析的共享输入。
- **`microduck_kinematics_rs`**（已吸收）：原来的独立卫星仓库，现合并为主仓库内的 `kinematics` crate。
- **MuJoCo**：仅在开发/测试时使用（Python 脚本生成 fixture），运行时不需要。
#（注：内容由AI生成）
