# manifest.rs 文件解析

## 文件位置

`d:\microduck\updater\src\manifest.rs`

## 定位

签名 manifest 描述一个发布。无法追溯到已发货机器人的字段从首个发布即存在——从未读过 `min_supported` 的机器人无法被强制升级。

## `Manifest` 字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `channel` | String | 交叉检查配置，防 URL 误配把模型当 daemon 装 |
| `version` | semver::Version | |
| `url` | String | 工件位置（绝对或相对 manifest） |
| `sha256` | String | 小写 hex SHA-256 |
| `sig_url` | String | 分离 minisign 签名 |
| `size` | Option<u64> | 压缩大小，仅用于空间预检 |
| `min_hw_rev` | u32 | 最低硬件版本，高于机器人 hw_rev 则拒绝 |
| `schema_version` | u32 | 磁盘/配置 schema，**不是**兼容性门（交给 post 钩子迁移） |
| `min_supported` | Option<Version> | 最低版本地板，低于此必须升级 |
| `model_api` | Option<u32> | 模型通道：daemon 需实现的 model API |
| `source_revision` | Option<String> | 构建 git SHA，溯源 |
| `changelog` | Option<String> | 展示文本，不可信 |

## `Capabilities`

```rust
pub struct Capabilities {
    pub hw_rev: u32,
    pub model_api: Option<u32>,  // robotd 不可达时 None
    pub schema_version: u32,      // 仅钩子上下文/诊断，不比较
}
```

## `Manifest::compatibility(caps) -> Compatibility`

- `min_hw_rev > caps.hw_rev` → `Refused`
- `model_api` 要求：
  - `caps.model_api` 为 Some 且满足 → `Ok`
  - Some 但不足 → `Refused`（"先更新 daemon"）
  - None → `Unknown`（robotd 不可达）——**故意不 Refused**：daemon 通道需通过（修复死 robotd 的方式），模型通道应等待

## `is_mandatory_for(installed)`

运行版本低于 `min_supported` → true。无安装版本不视为强制（无坏版本可逃）。

## `Compatibility` 枚举

`Ok` / `Refused(String)` / `Unknown(String)`。`Unknown` 与 `Refused` 区分：daemon 通道应通过 Unknown，模型通道应等待。

## 测试

- `unreachable_robotd_is_unknown_not_refused` — 不可达 robotd 使模型兼容性 Unknown 而非 Refused。
- `daemon_manifest_unaffected_by_dead_robotd` — daemon manifest 无 model_api，死 robotd 不阻挡。
- `floor_makes_update_mandatory_below_it` / `nothing_installed_is_not_mandatory`

## 关键摘要

manifest.rs 定义发布描述与兼容性判定：`Compatibility` 三态（Ok/Refused/Unknown）区分"明确不兼容"与"无法确定"，使 daemon 恢复更新能通过而模型更新等待；`schema_version` 故意不门控（否则每个 schema 升级都无法送达）；`min_supported` 是强制升级地板。
