# Cargo.toml 文件解析（xtask）

## 文件位置

`d:\microduck\xtask\Cargo.toml`

## 定位

`xtask` — 发布方构建工具：打包、签名、提升（promote）机器人发布版。**刻意做成独立 crate 而非 updater 的子命令**：它永远不发货到机器人。

## 关键设计：签名能力隔离

- xtask 链接完整 `minisign` crate（可签名）
- `updaterd` 只链接 `minisign-verify`（只能验签）
- **daemon 无权签名任何东西**——这是信任边界

## 为什么用 Rust 而非 shell 脚本

复用和 updater 测试完全相同的 `minisign`/`tar`/`zstd`/`sha2` crate。shell 版本会依赖单独安装的二进制，其行为可能与机器人验签时所用的不同——而那是最不允许差异存在的地方。

## 依赖

- `clap`（derive + env）— CLI
- `minisign 0.9` — 签名
- `semver` — 版本匹配
- `serde`/`serde_json` — manifest
- `sha2` — sha256
- `tar` + `zstd` — artifact
- `toml` — 读 Cargo.toml 元数据
- dev：`tempfile`（仅 artifact 集成测试用，打包到临时目录）

## 三个命令

```text
cargo xtask package --version 1.2.3 --channel daemon --bin-dir <dir> --out dist/
cargo xtask sign    --dir dist/ --key secret.key
cargo xtask promote --version 1.2.3 --staging-tag ... --stable-tag ... --repo ORG/REPO --out dist/
```

## promote 的核心含义

§16.3 的 `staging → stable`：发出一份 stable manifest，指向**已在 staging 验证过的同一 artifact 字节**（同一 sha256），而非重新构建。提升即重新签名，发出的可证明就是测试过的。

stable manifest 指向 **stable release** 上的 artifact（不再回指 staging）——曾因删除 staging release 导致三个 stable release 指向空。sha256 在机器人安装前验证，副本分歧不可能静默安装。
