# Cargo.toml 文件解析

## 文件位置

`d:\microduck\pet-detect\Cargo.toml`

## 包声明

- `name = "pet-detect"`
- `version`/`edition`/`rust-version`/`license` 全部继承 workspace

## 目标

- `[lib] name = "pet_detect"`
- `[[bin]] pet-detect` → `src/bin/detect.rs`
- `[[bin]] pet-features` → `src/bin/features.rs`

## 依赖

| 依赖 | 版本 | 用途 |
|---|---|---|
| `anyhow` | 1 | 错误处理 |
| `clap` | workspace, `derive` | CLI 参数解析 |
| `hound` | workspace | WAV 读取 |
| `rustfft` | 6.2 | FFT（mel 频谱） |
| `tracing` | workspace | 日志 |
| `ort` | `=2.0.0-rc.11`, `default-features=false`, `features=["load-dynamic"]` | ONNX Runtime |

## 关键依赖理由

### `ort = "=2.0.0-rc.11"`

与 `duck-control` 固定同一版本——板上只有一个 ONNX Runtime，两个 crate 共享它。`load-dynamic` 在首次使用时 dlopen libonnxruntime 而非链接，使 aarch64 交叉编译无需目标架构的 ONNX Runtime。

### `rustfft = "6.2"`

纯 Rust FFT，无 C 依赖，用于 mel 频谱计算。

### `hound`

WAV 解码，`pet-features` 用它加载训练数据。

## 关键摘要

`pet-detect` 的依赖极简：`rustfft`（纯 Rust FFT）+ `hound`（WAV）+ `ort`（ONNX Runtime，dlopen 不链接，与 `duck-control` 同版本）+ CLI/错误/日志基础库。运行时 .so 通过 `load-dynamic` 动态加载，不进交叉编译链路。
