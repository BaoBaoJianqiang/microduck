# source/github.rs 文件解析

## 文件位置

`d:\microduck\updater\src\source\github.rs`

## 定位

GitHub Releases 源——daemon 通道。

## "latest" 解析

列出 releases，取匹配 `tag_prefix` 的标签中**最高 semver**，而非用 `/releases/latest`：
- 该端点是 repo 级的，第二个通道共享 repo 就坏
- 它回答"最近发布"而非"最高版本"——给旧线打 patch 时两者不同

## 三前缀

- `tag_prefix`（稳定，如 `daemon-v`）
- `ref_tag_prefix`（分支构建，如 `daemon-dev-`，可移动）
- `staging_tag_prefix`（候选，如 `daemon-staging-v`）

分隔以防混淆：dev 标签可移动而发布标签不可变，`latest` 绝不能考虑 dev 标签。

## 不信任

标签、发布名、资产 URL 都可能被攻击者或失误影响；唯一使工件可接受的是调用方事后检查的 minisign 签名。

## 资产下载

用 API endpoint（`/repos/{repo}/releases/assets/{id}`）+ `Accept: application/octet-stream`，而非 `browser_download_url`（私有仓库 404）。

## 关键摘要

github.rs 是 daemon 通道源：按 semver 最高版本解析 latest（非 GitHub latest 端点）；三前缀分隔稳定/分支/候选流；资产通过 API endpoint 下载支持私有仓库；所有元数据不信任，签名是唯一真相。
