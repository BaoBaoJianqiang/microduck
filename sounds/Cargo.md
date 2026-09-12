# Cargo.toml 文件解析

## 文件位置

`d:\microduck\sounds\Cargo.toml`

## 包信息

- `name = "sounds"`
- `version/workspace` —— 继承工作区版本
- `edition.workspace` —— 继承工作区 edition
- `rust-version.workspace` —— 继承工作区 rust 版本
- `license.workspace` —— 继承工作区许可证
- `description` —— 机器人声音：可种子化的鸭子叫声合成器

## 依赖分析

- `anyhow = "1"` —— 错误处理。
- `clap = { workspace = true, features = ["derive"] }` —— CLI 参数解析。
- `hound.workspace = true` —— WAV 文件读写（与 `duck-control`/`pet-detect` 等共享版本）。
- `sha2 = "0.11"` —— SHA-256，用于硬件种子派生（`seed_from_id`）。

## 设计要点

整个 crate 无 C 依赖、无运行时 .so、无外部音频库。声音合成是纯 Rust 计算：
- RNG（xoshiro256++ + CRC-32）手写
- DSP 原语（包络、振荡器、滤波器）手写
- MIDI 解析手写
- 文本乐谱解析手写
- WAV 输出用 `hound`（纯 Rust）

唯一外部算法依赖是 `sha2`（硬件种子派生与 Python 安装程序兼容）。这与 `duck-detect`/`pet-detect` 通过 dlopen 加载 ONNX Runtime 不同——sounds 完全自包含，因为声音合成不需要第三方推理运行时。

## 关键摘要

Cargo.toml 体现了 sounds crate 的自包含设计：4 个依赖（anyhow、clap、hound、sha2），全部纯 Rust，无 C 工具链、无运行时 .so、无音频库。声音合成的所有 DSP、RNG、MIDI/文本解析都在 crate 内手写实现。
