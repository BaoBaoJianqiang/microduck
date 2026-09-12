# 解析：`Cargo.toml`

## 这是什么

Rust workspace 的**根清单文件**（84 行，几乎一半是有内容的注释）。它不编译任何代码，而是定义一个 workspace：成员 crate 列表、成员共享的包元数据、集中声明的依赖版本，以及三个自定义的 `[workspace.metadata]` 版本钉。

文件头注释交代了组织原则：**每个服务或工具一个 crate**，与 [`docs/design/architecture.md`](docs/design/architecture.md) §1 的服务划分一一对应——`robotd`、`btd`、`mediad` 等都是平级兄弟。

## 逐节解析

### `[workspace]`（第 5-7 行）

| 配置 | 值 | 说明 |
|---|---|---|
| `resolver` | `"3"` | 第 3 版依赖解析器（edition 2024 的默认风格） |
| `members` | 19 个 crate | `btd`、`configd`、`duck-control`、`duck-detect`、`duck-ipc-proto`、`duckctl`、`kinematics`、`mediad`、`odometry`、`padd`、`pet-detect`、`sounds`、`tof`、`updater`、`robotctl`、`robotd`、`robotd-params`、`test-support`、`xtask` |

### `default-members`（第 9-21 行）—— 注释最长的节

**除 `duckctl` 之外的全部成员，"而这个例外正是这个键存在的全部理由"**。注释给出完整的因果链：

- `cargo board --bins`（在 [`scripts/dev-push.sh`](robot/dev-push.md) 与 release workflow 中使用）为 aarch64 构建每一个默认成员——这对机器人的守护进程是对的，对开发者在自己电脑上运行的客户端是错的：它会在**发布路径上**、为一台永远不该见到蓝牙的板子交叉编译一个蓝牙栈，纯属浪费。
- 替代方案是在每个 `--bins` 调用点显式写出二进制名——那样的调用点有两个，必须靠人工保持同步，"这正是本仓库反复写下的那种失败"。
- 一处列表，新加的守护进程自动被两条路径同时拾取，无需任何人记住什么。
- `--workspace` 不受影响，所以 CI 仍然照旧 lint 与测试 `duckctl`。

### `[workspace.metadata.onnxruntime]`（第 23-32 行）

```toml
floor  = "1.23"
target = "1.28.0"
```

`robotd` **dlopen**（运行时动态加载而非链接）的 ONNX Runtime。单一事实来源原则：`xtask package` 把这两个值烘进发布版的 preinstall hook，并且有一个测试断言 `scripts/setup-board.sh` 与它们一致——"两份拷贝的漂移，正是一块板子最后拿着 1.20.1、而 `ort` 在 1.23 以下 panic 的原因"。

- `floor` 是 `ort` 能接受的最低版本——随 `ort` 依赖同步上调；
- `target` 是板子低于 floor 时安装到的版本。

### `[workspace.metadata.rknpu]`（第 34-42 行)

```toml
runtime = "v2.3.2"
```

`duck-detect` dlopen 的 Rockchip NPU 运行时。**与 ONNX Runtime 钉同一种形态、为同一个原因**：`scripts/setup-npu.sh` 被 curl 独立拉取、读不到本文件，所以它带一个字面量，测试断言两者一致。版本是 Rockchip `rknn-toolkit2` 仓库里的 tag（`.so` 所在地），且必须**不低于**转换器产出的模型——运行时旧于模型会在 `rknn_init` 时报错。

### `[workspace.metadata.gst-plugins]`（第 44-54 行）

```toml
repo    = "pollen-robotics/microduck-gst-plugins"
version = "v3"
```

`mediad` 需要的**预编译 GStreamer 插件**，在 CI 里从钉住的 upstream 源构建：`mpph264enc`（经 Rockchip MPP 的硬件 H.264 编码）与 `webrtcsink`/`webrtcsrc`——**任何一个 Debian suite 里都不存在**。同样的单一事实来源理由：`scripts/setup-gstreamer.sh` 被 curl 独立拉取，带字面量，测试断言一致——"一个版本的两份拷贝漂移开，就是一块板子拿到其 `ort` 会 panic 的 ONNX Runtime 的方式"。

### `[workspace.package]`（第 56-64 行）

| 配置 | 值 | 说明 |
|---|---|---|
| `version` | `0.10.0` | 全 workspace 共享版本号（Cargo.lock 中 19 个成员全部是 0.10.0，可互相印证） |
| `edition` | `2024` | Rust 2024 版 |
| `rust-version` | `1.89` | 注释解释：为了 `std::fs::File::try_lock`——single-flight 更新锁（`updater/src/journal.rs`）不引依赖的实现就靠它；edition 2024 本身要求 1.85，1.89 两者都覆盖。显式声明后，旧工具链的队友得到的是"requires rustc 1.89"而不是一个"方法不存在"的费解报错 |
| `license` | `Apache-2.0` | 许可证 |

### `[workspace.dependencies]`（第 66-77 行）

集中声明成员共享的依赖与版本，成员以 `workspace = true` 引用：`serde`（derive 特性）、`serde_json`、`semver`（serde 特性）、`thiserror 2`、`tokio 1`、`clap 4`（derive）、`hound 3.5`（wav 音频）、`tracing 0.1`、`tracing-subscriber 0.3`、`tokio-util 0.7`、`async-trait 0.1`。

### 刻意缺席的 `[profile.release]`（第 79-83 行）

注释解释为什么不自定义 release profile：二进制体积不值得优化——模型产物远大于几 MB 的二进制，调优成本高于收益。其中最要紧的一条：**`strip = true` 会剥掉符号表，把 journald 里的 panic 回溯变成裸地址**——而这恰是诊断一台你无法接调试器的机器人时最需要的东西。`lto`/`codegen-units` 实测只是拉长构建。

## 在整个项目里的位置

三个 `[workspace.metadata]` 钉 + 一个测试断言，构成"清单 ↔ 独立脚本"的版本同步机制；`default-members` 划清"机器人上的守护进程"与"开发者本机的客户端"的构建边界；`rust-version`/`edition` 保证工具链兼容性报错可读。
