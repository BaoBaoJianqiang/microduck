# Cargo.toml 文件解析

## 文件位置

`d:\microduck\robotd\Cargo.toml`

## 核心设计决策

### 1. 依赖结构

`robotd` 是聚合型 daemon，依赖所有功能 crate：

| 依赖 | 用途 |
|---|---|
| `duck-ipc-proto` | IPC 契约类型 |
| `duck-control` | 控制核心（model/bus/io/safety/policy/fall） |
| `robotd-params` | 参数 schema（单一真相源） |
| `kinematics` | 特雷门琴的手检测 |
| `odometry` | 接触式里程计 |
| `pet-detect` | 抚摸检测（麦克风） |
| `sounds` | 实时语音合成（特雷门琴/合唱） |
| `clap` | CLI |
| `tokio` | 多线程运行时（net/time/io-util/signal/sync） |
| `arc-swap` | 意图槽无锁交接 |

### 2. `arc-swap` 的用途

注释明确：意图交接用无锁单值槽，控制循环每 tick 一次原子 load，永不被 IPC 任务阻塞。

### 3. dev-dependencies 的特殊安排

| 依赖 | 理由 |
|---|---|
| `updater` | `tests/updater_gate.rs` 用真实更新引擎驱动本二进制。**单向**：updater 不依赖 robotd。 |
| `tokio`（test-util） | `start_paused` 用于启动重试测试（重试间隔 1s 真机合适，测试太慢）。仅 dev，不进发布二进制。 |
| `test-support` / `minisign` / `semver` / `sha2` / `tar` / `tempfile` / `zstd` | 测试 fixture（签名发布、tar 包、版本）。 |

`updater` 作为 dev-dependency 的注释特别说明：测试放本包而非 `updater/tests/`，因只有此处 cargo 定义 `CARGO_BIN_EXE_robotd` 并保证二进制重建。

## 关键摘要

`Cargo.toml` 反映 robotd 的聚合性质：依赖 duck-control 控制核心、kinematics/odometry/pet-detect/sounds 功能 crate。`arc-swap` 支撑无锁意图交接。dev-dependencies 含 `updater`（仅用于跨真实 socket 验证契约的测试），测试放本包是为了 `CARGO_BIN_EXE_robotd` 的重建保证。
