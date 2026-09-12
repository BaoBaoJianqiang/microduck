# source/mod.rs 文件解析

## 文件位置

`d:\microduck\updater\src\source\mod.rs`

## 定位

工件来源。一个 trait，三个后端：GitHub Releases（daemon）、HF Hub（模型）、本地目录。本地后端不是玩具——是引擎在无网络下用真实代码路径测试的方式。

## `Source` trait

| 方法 | 说明 |
|---|---|
| `latest_manifest()` | 取当前最新 manifest（`SignedBytes<Manifest>`） |
| `manifest_for(version)` | 精确版本（`--version X`） |
| `manifest_at_ref(git_ref)` | 命名 ref（`--ref my-branch`，默认不支持） |
| `staging_manifest()` | 最新候选（`--staging`，默认不支持） |
| `staging_manifest_for(version)` | 命名候选 |
| `fetch_artifact(manifest, dest_dir, progress)` | 下载工件+签名到 dest_dir，流式到磁盘 |

`manifest_at_ref`/`staging_manifest` 默认返回 `Incompatible`——不是每个源都有 ref/候选概念。

## `SignedBytes<T>`

```rust
pub struct SignedBytes<T> {
    pub bytes: Vec<u8>,       // 原始接收字节
    pub signature: Vec<u8>,   // 分离签名
    pub parsed: T,            // 解析结果
}
```

保留原始字节使签名验证在**精确接收的字节**上运行，而非重序列化。

## `from_config(config)`

按 `SourceConfig` 构造对应后端。

## 关键摘要

source/mod.rs 定义工件来源 trait 与三个后端：GitHub Releases（按 semver 最高版本非 GitHub latest）、HF Hub（模型，自签 minisign）、LocalDir（测试/旁加载）；`SignedBytes` 保留原始字节供签名验证；ref/staging 方法默认不支持。
