# `Cargo.toml` 解读 — duck-detect

> 文件路径：`duck-detect/Cargo.toml`
> 角色：在 NPU/CPU 上检测其他 Microduck 机器人的 YOLO 目标检测器 crate

---

## 一、[package] 元数据

```toml
[package]
name = "duck-detect"
version.workspace = true
edition.workspace = true
rust-version.workspace = true
license.workspace = true
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `duck-detect` | crate 名，目标检测器 |
| `version` | workspace | 从工作区根继承 |
| `edition` | workspace | 从工作区根继承 |
| `rust-version` | workspace | 从工作区根继承 |
| `license` | workspace | 从工作区根继承 |

所有可变元数据从工作区根继承，确保整个仓库版本一致。

---

## 二、[dependencies] 依赖详解

### 2.1 `anyhow = "1"`

通用错误处理库。这个 crate 的错误路径（模型加载失败、推理失败、运行时缺失）都需要可组合的错误类型，`anyhow::Result` 是标准选择。

### 2.2 `libloading = "0.9"` — NPU 运行时 dlopen

```toml
# `dlopen` rather than link: the NPU runtime is a vendor blob that is not on a laptop, not in any
# Debian suite and not needed to *build* — cross-compiling a daemon must not require it. Same
# decision, and the same crate, as ort makes for ONNX Runtime.
libloading = "0.9"
```

**这是这个 crate 最关键的依赖决策。**

- **dlopen 而非链接**：`librknnrt.so` 是 Rockchip 的厂商 blob，不在笔记本上，不在任何 Debian 套件中，也不需要用来*构建*。一个链接它的守护进程根本无法在 CI 中交叉编译。
- **与 `ort` 对 ONNX Runtime 的决策相同**：`ort` 也用 dlopen 加载 `libonnxruntime.so`，同样的理由，同样的 crate（`libloading`）。
- **代价是 `rknn.rs` 中的手动 FFI**：七个函数指针类型、五个 `#[repr(C)]` ABI 结构体、手动符号查找。好处是 `cargo board --bins` 在没有任何 Rockchip 东西的机器上继续工作。

### 2.3 `tracing.workspace = true`

结构化日志。从工作区继承版本。NPU 初始化失败、推理超时等关键事件需要日志记录。

### 2.4 `clap = { workspace = true, features = ["derive"] }`

CLI 参数解析。`duck-bench` 二进制（见下方 `[[bin]]`）需要命令行参数（模型路径、帧数、后端选择等）。`derive` 特性允许用 `#[derive(Parser)]` 宏定义参数结构体。

### 2.5 `tracing-subscriber = { workspace = true, features = ["env-filter"] }`

日志订阅器。`env-filter` 特性允许通过 `RUST_LOG` 环境变量控制日志级别（如 `RUST_LOG=duck_detect=debug`）。

### 2.6 `image = { version = "0.25", default-features = false, features = ["jpeg"] }`

图像处理库，**仅启用 JPEG 解码**。

- **`default-features = false`**：禁用默认特性（包含 PNG、GIF、BMP、TIFF、WebP 等大量格式），大幅减少编译时间和二进制体积。
- **只需要 JPEG**：`duck-bench` 从 JPEG 文件加载测试图片进行基准测试。检测器的推理路径直接操作原始 RGB 字节（`letterbox_rgb` / `letterbox_from_uyvy`），不需要 `image` crate。
- **0.25 版本**：当前最新的稳定 0.x 系列。

### 2.7 `ort = { version = "=2.0.0-rc.11", default-features = false, features = ["load-dynamic"] }` — CPU 回退后端

```toml
# The CPU fallback. `load-dynamic` for the same reason `duck-detect` dlopens the NPU runtime: the
# library belongs to the board, not to the build — and `setup-board.sh` already puts it there for
# robotd's policies, so a board that can walk can also see.
ort = { version = "=2.0.0-rc.11", default-features = false, features = ["load-dynamic"] }
```

**ONNX Runtime Rust 绑定，用于 NPU 被关闭时的 CPU 回退。**

#### 版本钉死：`=2.0.0-rc.11`

- **精确版本钉死**（`=` 前缀），不使用 `^` 语义化版本范围。
- 这是一个 release candidate（rc）版本，API 可能在 rc 之间变化，钉死版本确保构建可复现。
- 2.0.0 是 `ort` 的大版本重写，API 与 1.x 不兼容。

#### `default-features = false`

禁用默认特性。`ort` 的默认特性可能包含静态链接 ONNX Runtime、下载预编译库等，这些在交叉构建场景下是有害的。

#### `features = ["load-dynamic"]` — 动态加载

- **`load-dynamic`**：通过 `dlopen` 动态加载 `libonnxruntime.so`，而非链接。
- **与 NPU 运行时相同的理由**：库属于板子，不属于构建机器。交叉编译守护进程不需要目标架构的预编译 ONNX Runtime 库。
- **`setup-board.sh` 已经为 robotd 的策略安装了它**，所以一个能走路的板子也能"看见"（运行检测器）——ONNX Runtime 已经在那里了。

#### 为什么需要 CPU 回退

RK3566 有 NPU，厂商内核有驱动——但 Armbian 把 `npu@fde40000` 标记为 `disabled`。启用它是一个 overlay 和一次重启，这是关于某人的机器人的决定。所以检测器在四个 A55 核上运行（ONNX Runtime），直到 NPU 启用，然后通过改一个配置值移到 NPU。

---

## 三、[[bin]] 二进制

```toml
[[bin]]
name = "duck-bench"
path = "src/bin/duck-bench.rs"
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `name` | `duck-bench` | 二进制名 |
| `path` | `src/bin/duck-bench.rs` | 源文件路径 |

**`duck-bench`**：在真实板子上测量检测器性能的基准测试工具。测量 letterbox、推理（NPU/CPU）、decode 各阶段的耗时，帮助定位性能瓶颈和验证优化效果。

需要 `clap`（CLI 参数）、`image`（加载 JPEG 测试图）、`tracing-subscriber`（日志）。

---

## 四、依赖关系图

```
duck-detect
├── anyhow 1              — 错误处理
├── libloading 0.9        — dlopen librknnrt.so（NPU 后端）
├── tracing (workspace)   — 结构化日志
├── clap (workspace)      — CLI 解析（duck-bench）
├── tracing-subscriber    — 日志订阅器（duck-bench）
├── image 0.25 (jpeg only)— JPEG 解码（duck-bench 测试图）
└── ort =2.0.0-rc.11      — ONNX Runtime（CPU 回退，load-dynamic）
    └── load-dynamic      — dlopen libonnxruntime.so
```

---

## 五、关键设计决策总结

### 5.1 两个运行时都用 dlopen，不链接

| 运行时 | 加载方式 | 库 | 理由 |
|--------|----------|-----|------|
| NPU (Rockchip) | `libloading` dlopen | `librknnrt.so` | 厂商 blob，不在 Debian/笔记本上，链接破坏交叉编译 |
| CPU (ONNX Runtime) | `ort` load-dynamic | `libonnxruntime.so` | 库属于板子，setup-board.sh 已安装，交叉构建不需要目标预编译库 |

**一致性**：两个后端用相同的策略（dlopen），相同的底层 crate（`libloading`，`ort` 内部也用它）。这使得构建环境完全不需要目标架构的原生库——笔记本上能编译、能测试（没有运行时推理会失败，但编译通过）。

### 5.2 最小依赖原则

- `image` 只启用 JPEG（禁用 PNG/GIF/BMP/TIFF/WebP 等）
- `ort` 禁用默认特性，只启用 `load-dynamic`
- 没有 `tokio`（检测器是同步的，在自己的线程上运行，由 `mediad` 调度）
- 没有 `serde`/`serde_json`（检测器不需要 JSON，输入是原始字节，输出是 `Vec<Detection>`）

### 5.3 版本钉死

- `ort = "=2.0.0-rc.11"`：rc 版本 API 不稳定，精确钉死确保可复现构建
- 其他依赖用语义化版本范围（`anyhow = "1"`、`libloading = "0.9"`、`image = "0.25"`）

---

## 六、与其他 crate 的关系

- **`duck-control`**：机器人控制核心，也使用 `ort`（ONNX Runtime 策略推理），同样的 `load-dynamic` 决策
- **`robotd`**：使用 ONNX Runtime 进行策略推理，`setup-board.sh` 为它安装运行时——`duck-detect` 复用同一个运行时
- **`mediad`**：在自己的线程上运行检测器（`Model: Send` 但不 `Sync`），提供摄像头帧
- **`btd`**：BLE 守护进程，不直接依赖，但 `duck-bench` 的结果可能通过 BLE 报告

---

## 七、构建与交叉编译

### 笔记本上构建

```bash
cargo build -p duck-detect
```

- 不需要 `librknnrt.so`（dlopen）
- 不需要 `libonnxruntime.so`（load-dynamic）
- 编译通过，推理在运行时找不到库时会失败（明确的错误消息）

### 交叉编译到 aarch64

```bash
cargo board --bins  # 项目自定义的交叉构建命令
```

- 同样不需要目标架构的原生库
- `libloading` 和 `ort load-dynamic` 都不引入链接时依赖
- 这是 dlopen 决策的核心收益

### 运行时依赖

板子上需要：
- `librknnrt.so`（NPU 后端，由 `scripts/setup-npu.sh` 安装）
- `libonnxruntime.so`（CPU 回退，由 `setup-board.sh` 为 robotd 安装）

两个都不在构建时需要，都在运行时通过 dlopen 加载。
#（注：内容由AI生成）
