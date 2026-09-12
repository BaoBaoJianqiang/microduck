# Cargo.toml 文件解析（test-support）

## 文件位置

`d:\microduck\test-support\Cargo.toml`

## 定位

`test-support` — 供测试用的已签名 release 夹具（fixtures）。**永不发货**。

## 设计原则

刻意依赖和引擎验签时**相同**的 crate（`minisign`、`tar`、`zstd`、`sha2`），使夹具不可能产出真实代码会因"夹具自创的理由"而拒绝的 artifact。

## 依赖

- `minisign 0.9.1` — 签名夹具
- `semver` — 版本
- `serde_json` — manifest
- `sha2 0.11.0` — sha256
- `tar 0.4.46` — artifact
- `zstd 0.13` — 压缩
- `tempfile 3.27.0` — 临时目录
- dev：`clap`（仅 `fake-release` 示例 CLI 用）

## 背景

四个测试文件各自复制了一套：生成密钥对、构建 `.tar.zst`、签名、写 manifest、再签名。改 manifest 格式要改四处，且已在细节上漂移（如 tar mtime 是否固定）。
