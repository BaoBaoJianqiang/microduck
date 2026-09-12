# Cargo.toml 文件解析

## 文件位置

`d:\microduck\kinematics\Cargo.toml`

## 核心设计决策

鸭子的正运动学，由 MJCF 自身驱动。

运动学树从策略训练所用的同一 `robot_walk.xml` 解析出来，因此机械修订意味着更新一个 XML 文件，而非手工把关节偏移转录进 Rust。吸收自 `microduck_kinematics_rs` 卫星仓库，为控制循环热路径重做：关节是切片中的索引而非 `HashMap` 中的名字，site 查询走自己的预编译链而非重算树中每个 body。由 fixture 验证对抗 MuJoCo 的 `mj_kinematics`。

## 依赖分析

| 依赖 | 类型 | 版本 | 理由 |
|---|---|---|---|
| `roxmltree` | 普通 | `"0.21"` | 解析 MJCF XML。只读取树，不修改，因此选只读解析器 |
| `thiserror` | 普通 | workspace | `ParseError` 派生 |
| `serde` | dev | workspace | fixture JSON 反序列化 |
| `serde_json` | dev | workspace | 同上 |

## 关键摘要

`kinematics` 的依赖极简：仅 `roxmltree`（解析 MJCF）与 `thiserror`（错误类型），dev 依赖 `serde`/`serde_json` 用于加载 MuJoCo fixture。无 nalgebra——刚体代数手写并由 fixture 验证，避免增加编译时间。
