# `fk_against_mujoco.rs` 解读

## 概述

这是 `kinematics` crate 的集成测试，用 MuJoCo 的 `mj_kinematics` 作为 FK（前向运动学）的**地面真值**，验证自实现的运动学计算是否与 MuJoCo 完全一致。

### 测试哲学：不相信自己的数学

Fixture 是 64 个随机关节配置，每个配置记录 MuJoCo 计算的每个 site 的位姿。fixture 由 Python 脚本（`scripts/gen_fixtures.py`）通过 `uv` 运行，对同一个 `robot_walk.xml` 生成。

**当 MJCF 改变时重新生成 fixture**——一致性是精确的（1e-6），任何真实偏差都会响亮失败。

---

## Fixture 数据结构

```rust
#[derive(Deserialize)]
struct Fixture {
    samples: Vec<Sample>,
}

#[derive(Deserialize)]
struct Sample {
    joints: HashMap<String, f64>,
    sites: HashMap<String, SitePose>,
}

#[derive(Deserialize)]
struct SitePose {
    pos: [f64; 3],
    quat: [f64; 4],
}
```

- **`Fixture`**：顶层，包含多个采样
- **`Sample`**：一个关节配置 + 所有 site 的期望位姿
  - `joints`：关节名 → 角度（HashMap，因为 fixture JSON 中关节名是字符串键）
  - `sites`：site 名 → 期望位姿
- **`SitePose`**：位置（3 向量）+ 四元数（4 分量，wxyz）

**为什么用 `HashMap<String, f64>` 而非数组**：fixture JSON 由 Python/MuJoCo 生成，关节名是字符串键。测试需要按名查找关节索引（`model.joint_index(name)`），这也验证了关节名映射的正确性。

---

## 测试主体

### `alpha_matches_mujoco_on_every_site_of_64_random_poses`

```rust
const POS_TOL: f64 = 1e-6; // metres
const QUAT_TOL: f64 = 1e-6; // unitless
```

#### 容差

- **位置容差 1e-6 米**（1 微米）：非常严格，意味着自实现 FK 与 MuJoCo 的差异在浮点误差范围内
- **四元数容差 1e-6**：同样严格

这个容差不是工程容差（不需要这样精确），而是**验证"自实现 FK 与 MuJoCo 使用相同的数学"**。如果运动学树解析正确、关节变换正确，差异应该只有浮点舍入误差，远小于 1e-6。

#### 加载 fixture 和模型

```rust
let fixture: Fixture =
    serde_json::from_str(include_str!("fixtures/fk_alpha.json")).expect("fixture parses");
let model = Model::alpha();
```

- **`include_str!`**：fixture JSON 在编译时嵌入二进制。测试不需要外部文件，`cargo test` 自包含。
- **`Model::alpha()`**：构建 alpha 变体机器人的运动学模型，从嵌入的 MJCF XML 解析。

#### 遍历每个采样

```rust
for (i, sample) in fixture.samples.iter().enumerate() {
    let mut angles = vec![0.0; model.num_joints()];
    for (name, &angle) in &sample.joints {
        let idx = model.joint_index(name)
            .unwrap_or_else(|| panic!("fixture joint {name:?} missing from model"));
        angles[idx] = angle;
    }
```

将 fixture 的关节名→角度映射转换为模型的关节索引→角度数组。如果 fixture 中的关节在模型中不存在，panic（说明 MJCF 不一致）。

#### 遍历每个 site 并比较

```rust
for (site_name, expected) in &sample.sites {
    let site = model.site(site_name)
        .unwrap_or_else(|| panic!("fixture site {site_name:?} missing from model"));
    let got = model.site_pose(site, &angles);
```

同样按名查找 site，如果不存在则 panic。调用 `model.site_pose(site, &angles)` 计算自实现的 FK。

#### 位置比较

```rust
let pos_err = expected.pos.iter().zip(got.pos)
    .map(|(a, b)| (a - b).abs())
    .fold(0.0, f64::max);
assert!(pos_err < POS_TOL, "sample #{i} site {site_name:?}: pos off by {pos_err} ...");
```

取三个分量绝对误差的最大值（无穷范数），而非均方根——因为最坏情况分量必须在容差内。

#### 四元数比较（符号歧义处理）

```rust
// q 和 -q 是同一个旋转；接受 MuJoCo 选择的任一符号。
let got_q = got.quat.wxyz();
let err = |sign: f64| {
    expected.quat.iter().zip(got_q)
        .map(|(a, b)| (a - sign * b).abs())
        .fold(0.0, f64::max)
};
let quat_err = err(1.0).min(err(-1.0));
```

**关键细节：四元数双覆盖（double cover）。** 四元数 `q` 和 `-q` 表示完全相同的旋转。MuJoCo 的 FK 实现可能返回 `q`，自实现可能返回 `-q`（或反之），这取决于内部计算顺序。

如果直接逐分量比较，`q` 和 `-q` 的差异是 2.0（每个分量符号相反），远大于 1e-6。

解法：计算 `err(+1.0)`（直接比较）和 `err(-1.0)`（翻转符号后比较），取最小值。如果两者中有一个在容差内，说明旋转相同。

---

## 设计要点

### 1. Fixture 嵌入而非外部文件

`include_str!("fixtures/fk_alpha.json")` 在编译时嵌入 JSON。好处：
- `cargo test` 自包含，不需要 fixture 文件在文件系统中
- CI 中测试不会因为路径问题失败
- fixture 版本与代码版本绑定（改代码时 fixture 不会意外不匹配）

### 2. 64 个随机配置而非边界值

64 个随机关节配置覆盖了工作空间的充分采样。不是只测零位/伸直/弯曲等边界值——随机采样更能发现累积误差（每关节变换的小错误在链末端放大）。

### 3. 关节名和 site 名都通过测试验证

测试不仅验证数值，还验证：
- fixture 中的每个关节名在模型中都存在
- fixture 中的每个 site 名在模型中都存在
- 如果 MJCF 改了但 fixture 没重新生成，测试会因为名称不匹配而失败（响亮失败）

### 4. 重新生成流程

当 `robot_walk.xml` 改变时：
1. 修改 XML（机械修订）
2. 运行 `scripts/gen_fixtures.py`（通过 `uv`）生成新的 fixture JSON
3. 运行 `cargo test`——如果运动学实现正确，测试通过
4. 如果测试失败，说明自实现 FK 与新 MJCF 不一致，需要修正

这个流程确保：**机械修订 → 更新 XML → 重新生成 fixture → 验证 FK**，一步都不能少。

---

## 与其他文件的关系

- **`kinematics::Model`**：被测对象，提供 `alpha()`、`num_joints()`、`joint_index()`、`site()`、`site_pose()`
- **`fixtures/fk_alpha.json`**：fixture 数据文件（编译时嵌入）
- **`scripts/gen_fixtures.py`**：fixture 生成脚本（在 `microduck_kinematics_rs` 仓库中，通过 `uv` 运行）
- **`robot_walk.xml`**：MJCF 模型文件，测试和 MuJoCo 共用同一个
- **`perf_probe.rs`**：同目录的性能探针测试文件

---

## 关键踩坑点总结

1. **四元数符号歧义**：`q` 和 `-q` 是同一个旋转，直接逐分量比较会失败。必须计算两种符号的误差取最小值。这是四元数比较中最常见的陷阱。

2. **容差是浮点误差级别而非工程级别**：1e-6 不是"足够好"的工程容差，而是验证"数学完全一致"。任何运动学树解析错误、关节轴方向错误、零位偏移错误都会导致远大于 1e-6 的偏差。

3. **fixture 必须与 MJCF 同步**：机械修订后只改 XML 不重新生成 fixture，测试会因为名称不匹配而响亮失败——这是特性而非 bug，它强制执行"改模型就重新验证"。

4. **位置比较用无穷范数**：取三个分量误差的最大值，而非均方根。因为最坏分量必须在容差内——一个分量错了其他两个对了仍然是错误。

5. **关节名和 site 名是测试的一部分**：按名查找并 panic on missing，意味着 fixture 和模型之间的任何不一致都会被立即发现，而非静默忽略。
#（注：内容由AI生成）
