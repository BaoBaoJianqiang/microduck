# source/hf_hub.rs 文件解析

## 文件位置

`d:\microduck\updater\src\source\hf_hub.rs`

## 定位

Hugging Face Hub 源——模型通道。文件解析于 `https://huggingface.co/{repo}/resolve/{revision}/{file}`。

## HF 不签名

HF 不为我们签名任何东西，所以我们在每个工件旁发布自己的 minisign 签名并验证它。

## 与 GitHub Releases 的两点不同

1. **无 "releases" 概念，仅有 git revisions。** `revision` 通常是移动分支，所以 "latest" 意为"该分支当前指向"——manifest 自身的 `version` 字段才是实际安装了什么的权威，绝非分支名。
2. **精确版本按惯例映射到 tag**（`v1.2.3`），所以精确获取是同一路径不同 revision。

## 关键摘要

hf_hub.rs 是模型通道源：HF 不签名故自签 minisign；无 releases 概念，revision 通常是移动分支，manifest.version 才是权威；精确版本按 tag 惯例映射。
