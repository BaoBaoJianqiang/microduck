# Cargo.toml 文件解析

## 文件位置

`d:\microduck\duck-detect\Cargo.toml`

## [package] 段

| 字段 | 值 |
|---|---|
| `name` | `duck-detect` |
| `version` / `edition` / `rust-version` / `license` | `workspace = true`（从工作区继承） |

## 依赖分析

| 依赖 | 版本/特性 | 说明 |
|---|---|---|
| `anyhow` | "1" | 错误处理 |
| `libloading` | "0.9" | **dlopen 而非链接**：NPU runtime 是厂商 blob，不在笔记本、不在 Debian、不需要用于构建——交叉编译守护进程不应要求它。与 `ort` 对 ONNX Runtime 的决定相同 |
| `tracing` | workspace | 日志 |
| `clap` | workspace + `derive` | 命令行解析（`duck-bench`） |
| `tracing-subscriber` | workspace + `env-filter` | 日志订阅器 |
| `image` | "0.25" + `jpeg`（默认特性关） | 仅 JPEG 解码，`duck-bench` 用 |
| `ort` | `=2.0.0-rc.11` + `load-dynamic` | CPU 回退。`load-dynamic` 与 dlopen NPU runtime 同理由：库属于板上而非构建——`setup-board.sh` 已为 `robotd` 策略放置它，能走的板也能看 |

## [[bin]] 段

- `name = "duck-bench"`
- `path = "src/bin/duck-bench.rs"`

## 关键摘要

`duck-detect` 的依赖围绕一个核心原则：**运行时库（NPU 的 `librknnrt.so` 和 ONNX Runtime）都通过 dlopen 加载，不参与链接**，使交叉编译不依赖目标板专有库。`ort` 固定 `2.0.0-rc.11` 与 `duck-control` 一致（同一 ABI）。`image` 仅启用 JPEG 特性减小体积。`clap` 供 `duck-bench` 基准工具使用。
