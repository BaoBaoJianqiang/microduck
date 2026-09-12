# Cargo.toml 文件解析

## 文件位置

`d:\microduck\duckctl\Cargo.toml`

## 核心设计决策

`duckctl` 是**客户端侧**工具，运行在离机器人很远的地方。随机器人发布的工具是 `robotctl`，它在板上讲 Unix socket；此工具从外部到达机器人，因此携带蓝牙栈、扫描器以及守护进程绝不能有的所有笔记本侧关注点。

### 1. 从 `btd` 的 example 独立成 crate

它曾是 `btd` 的一个 example。example 的依赖是 dev-dependency，所以 `btleplug` 永远无法进入发布产物——这是一种真实保证，靠文件所在目录的副作用获得。独立成 crate 后保留该保证并直接声明：**机器人上没有任何东西依赖 `duckctl`**，因此此处的一切都不在发布版中。

顺便解决了两件尴尬事：
- 安装命令从 `cargo install --path btd --example duck-btctl`（`dev-push.sh` 还带了回退）变为 `cargo install --path duckctl`
- 名字不再含 `bt`，因为机器人即将拥有第二条传输通道（WebRTC）

### 2. 被排除在 workspace 的 `default-members` 之外

这是让它不进板载构建的关键：`cargo board --bins` 为 aarch64 构建每个默认成员，而此工具是给开发者坐着的那台机器用的。`--workspace` 仍会 lint 和测试它。

## 依赖分析

| 依赖 | 版本/特性 | 理由 |
|---|---|---|
| `btd` | `path = "../btd"` | 机器人线协议的自有半部分，复用而非重实现。`framing` 尤其重要：它是机器人分块所用模块的*客户端*侧，不对称会表现为此工具不工作——使其成为对协议的测试而非可自说自话的第二实现 |
| `duck-ipc-proto` | `path = "../duck-ipc-proto"` | `API_VERSION` 及与之比较的 `semver` 重导出。版本握手是此客户端与每个守护进程之间的契约，从定义它的 crate 读取而非在此处维护一个可能漂移的数字 |
| `btleplug` | `"0.11"` | 而非 `bluer`，因为运行在开发者机器上：macOS CoreBluetooth、Linux BlueZ、Windows WinRT。`bluer` 会使客户端仅限 Linux，违背"一个工具跨平台"的初衷 |
| `clap` | workspace, `derive` | CLI 解析 |
| `futures` | `"0.3"` | 流处理 |
| `webbrowser` | `"1.2.4"` | `open` 命令启动浏览器，跨平台（macOS `open`、Linux `xdg-open` 及五个回退、Windows `cmd /c start`），用 crate 而非 `target_os` match |
| `serde_json` | workspace | JSON-RPC 序列化 |
| `tokio` | workspace, `rt-multi-thread`/`macros`/`time` | 异步运行时 |

## Example 声明

```toml
[[example]]
name = "advwatch"
```

`advwatch` 测量广告实际到达频率。放在 `duckctl` 而非 `btd` 的理由与客户端相同：它是扫描器，扫描器运行在笔记本上。

## 关键摘要

`duckctl` 的 Cargo.toml 核心是"客户端侧隔离"：通过独立 crate + 排除 `default-members`，确保 `btleplug` 等笔记本侧依赖绝不进入机器人发布版。依赖选择均围绕跨平台（开发者机器可能是 macOS/Linux/Windows）与复用机器人自有协议（`btd::framing`、`duck-ipc-proto`）展开。
