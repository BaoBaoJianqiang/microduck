# registry.rs 文件解析

**文件位置**：`d:\microduck\robotd-params\src\registry.rs`

## 核心设计决策

`robotd.toml` 中**每个 key 的机器可读索引**——人机交互式编辑器（`robotctl configure`）所需的一切。`Params` 及其 `Default` impl 是*值*的真相，但它们在运行时无法承载编辑器需要的部分：一行描述、key 取值类型、枚举的可选名称。这就是本表。

**防漂移是测试而非希望**：`tests::the_registry_covers_every_key_exactly` 序列化 `Params::default()` 并遍历 key 树——每个叶子必须出现在此，且除此之外不得有其他。向 `Params` 加一个 section，构建仍绿但测试会点名 registry 缺失的 key，编辑器永远不可能静默不知道某个设置。

## 类型

### `Kind`（key 取值种类）
- `Bool`：开关。
- `TriBool`：开/关/未设（`Option<bool>`，例如 pet 检测未设时按模式解析）。
- `Integer`：整数。
- `Float`：浮点。
- `OptionalFloat`：浮点，未设意为"按模式/测量/默认解析"。
- `OptionalInteger`：整数，未设意为"跟随其他项"（如 `media.bitrate` 跟随 quality）。
- `Choice(&[&str])`：固定名称集合之一。
- `Text`：自由文本（ALSA 设备、socket 路径等）。
- `OptionalPath`：文件路径，未设意为发布自带副本，`"none"` 字面量禁用。
- `IntegerList`：整数列表，以逗号分隔文本编辑。

### `Entry`（一个 key）
- `key`：`section.key`，与 serde 拼写完全一致。
- `kind`：`Kind`。
- `doc`：一行描述，给编辑器页脚。
- `feature`：*功能开关*——少数人会打开编辑器去翻转的 key，区别于需读完整文档才能动的调优项。编辑器把这些列在最前。

### `REGISTRY` 常量
所有 key，按 section 分组，section 顺序与发布文件一致。约 80 个条目，按 section：bus、control、update_gate、policy、safety、detect、chorale、theremin、audio、media。

标为 `feature` 的 13 个开关：
`policy.enabled`、`policy.mode`、`policy.voltage_adapt`、`safety.battery_empty_shutdown`、`safety.limp_fall`、`detect.enabled`、`chorale.accept`、`theremin.enabled`、`audio.enabled`、`audio.greet`、`audio.pet_detect`、`media.camera`、`media.quality`。

## 函数

- `entry(key, kind, doc)`：构造非 feature 条目。
- `feature(key, kind, doc)`：构造 feature 条目。
- `entry_for(key)`：查 key 的 registry 条目。
- `has_section(section)`：本构建是否有该 section。与 `entry_for` 分开，因为未知 section（本构建无此功能）与已知 section 内未知 key（多半是 typo）应得到不同报告。

## 单元测试

### `the_registry_covers_every_key_exactly`
防漂移核心测试，双向：
1. **覆盖完整性**：从顶层 serde 拒绝消息提取所有 section 名，对每个 section 用 `fields_of(section)` 提取其所有字段（含 `Option`）。技巧：用 `__no_such_key__` 探针触发 `deny_unknown_fields`，从错误消息里读出 serde 本要打印的字段列表——序列化 `Params::default()` 会省略所有 `None` 的 `Option`，无法覆盖可选字段。
2. **每个 registry key 必须真实**：对每个 entry 按 kind 生成探针 TOML，必须能解析进 `Params`。这抓住被重命名或删除的 key 的残留条目。
3. **无重复**：`entry_for` 首次匹配会隐藏重复，故显式去重断言。

### `choices_match_the_types`
每个 `Choice` 列出的值必须被类型接受，且类型不接受列表之外的值——编辑器循环选项时永远写不出无效文件。

### `the_feature_switches_are_the_expected_set`
钉死 feature 开关集合——typo 会静默把开关降级为调优项。

## 关键摘要

本模块是编辑器的 schema。通过"用 serde 自己的错误消息提取字段列表"这一技巧，完整性测试能覆盖 `Option` 字段（普通序列化会漏掉），从而真正保证 registry 与 `Params` 双向一致。一行文档刻意简短，完整推理在 `Params` 的 doc 注释与发布的 `deploy/robotd.toml` 里。
