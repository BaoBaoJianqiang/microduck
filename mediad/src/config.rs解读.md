# `config.rs` 解读

## 概述

`config.rs` 是 `mediad`（Microduck 机器人的媒体守护进程）中负责**读取并解析媒体配置**的薄封装模块，共 101 行。它本身不定义任何配置 schema，而是把 `/etc/robot/robotd.toml` 中 `[media]`（摄像头还是测试图样、分辨率、帧率、码率）与 `[detect]`（目标检测参数）两个段落的读取工作整体委托给共享 crate `robotd_params`。

这个文件在 crate 中的角色可以用三句话概括：

1. **它是配置的入口，但不是配置的定义者**：schema、默认值、校验规则全部由 `robotd_params::Params` 提供。这样做的关键理由写在文件头注释里——`robotctl configure` 这个编辑器写回的正是同一个 crate 解析的文件，因此编辑器不可能提供一个 `mediad` 读不懂的值（"the editor cannot offer a value this daemon would not understand"）。
2. **它选择"容忍"而不是"崩溃"**：`robotd` 本体在参数文件损坏时拒绝启动（那是响亮的告警信号），但 `mediad` 不会跟着一起挂——机器人已经在宕机状态，摄像头正是人去查看它的窗口。所以 `mediad` 在文件不可读时**打一条 warn 日志并回退到内置默认值继续推流**。
3. **它单独成模块，是为了能在开发机上编译和测试**：文件头明确说明 `main.rs` 是 Linux-only 的，任何放在 `main` 里的代码在写代码的笔记本上根本不会被编译、更不会被跑测试。把配置逻辑抽成独立模块，是为了让它在任何平台都能跑单测。

## 关键结构体与函数

### `pub fn default_path() -> PathBuf`

```rust
/// The file, when `--config` said nothing.
/// —— 当命令行没有传 `--config` 时使用的默认路径。
pub fn default_path() -> PathBuf {
    PathBuf::from(robotd_params::DEFAULT_PATH)
}
```

用途：返回默认配置文件路径，直接转发 `robotd_params::DEFAULT_PATH`。本身没有逻辑存在，唯一价值是把"默认路径在哪"这个事实收敛到本模块一个符号上，`main.rs` 不需要知道 `robotd_params` 的存在。

### `pub fn load(path: &Path, explicit: bool) -> Params`

这是整个文件的核心函数，也是设计意图最密集的地方：

```rust
/// Read the file, or fall back to the built-in defaults.
/// —— 读取文件，失败则回退到内置默认值。
///
/// **A file this daemon cannot read is not a reason to have no video.**
/// —— 一个本守护进程读不动的文件，绝不是"没有视频"的理由。
/// `robotd` refuses to start on a broken params file — that is the loud signal, and it is the
/// daemon whose control loop the file configures. A robot in that state is already down, and its
/// camera is how somebody looks at it. So this warns, names the file, and streams the defaults
/// rather than joining the outage.
/// —— `robotd` 在参数文件损坏时拒绝启动——那才是响亮的信号，而且那个文件配置的正是它自己的控制环。
/// 处于那种状态的机器人本来就已经宕机了，它的摄像头是人去查看它的窗口。所以这里只打一条警告、点名是哪个文件，
/// 然后用默认参数推流，而不是跟着一起宕机。
///
/// A *missing* file is not even a warning at the default path: an unprovisioned board has none and
/// streams its camera at 720p30 like every other.
/// —— 在默认路径下，文件**不存在**甚至连警告都不会有：一块未初始化的板子本来就没有这个文件，
/// 它和其他板子一样以 720p30 推摄像头。
/// A path named on the command line must exist, which is `Params::load`'s own rule and the reason
/// `explicit` is passed through.
/// —— 命令行显式指定的路径必须存在，这是 `Params::load` 自己的规则，也是 `explicit` 参数被透传下去的原因。
pub fn load(path: &Path, explicit: bool) -> Params {
    match Params::load(path, explicit) {
        Ok(params) => params,
        Err(e) => {
            tracing::warn!(
                error = %e,
                path = %path.display(),
                "unusable params file; streaming the built-in defaults"
                // —— 参数文件不可用；使用内置默认参数推流
            );
            Params::default()
        }
    }
}
```

设计要点与踩坑点：

- **失败语义的分级**：默认路径下文件缺失 = 完全静默（未初始化板子的正常状态）；显式路径缺失或文件损坏 = warn + 默认值。这两级区分很重要，避免在正常的首次开机场景刷假告警。
- **`explicit: bool` 的语义**：这个参数不是 `mediad` 自己的判断，而是透传给 `robotd_params::Params::load` 的。注释解释得很清楚——"命令行显式指定的路径必须存在"这条规则是底层库的，`mediad` 只是不替它做决定。调用方（`main.rs`）负责把"用户在 `--config` 里写了路径"这件事翻译成 `true`。
- **为什么不用 `anyhow::Result` 向上传播**：因为这里的失败是**预期内的降级路径**，不是"系统出问题了"。返回 `Params::default()` 让下游无差别地继续走正常推流路径，避免在 pipeline 装配处再写一层"如果配置加载失败就用默认值"的分支。
- **日志字段设计**：warn 里同时带 `error = %e` 和 `path = %path.display()`，运维一眼能看出是哪个文件、为什么挂——这与 `robotd` 直接拒启动的"响亮信号"形成互补：`robotd` 负责"必须被看到"，`mediad` 负责"出事了也能看到机器人"。

## 重要常量

本文件自身没有定义常量。所有常量都在 `robotd_params` crate 中：

- **`robotd_params::DEFAULT_PATH`**：默认配置文件路径（即 `/etc/robot/robotd.toml`）。`default_path()` 直接转发它。
- **`MediaParams::default().quality`**：默认画质，注释与测试中反复出现的"720p30"就是它（测试 `a_missing_file_streams_the_defaults` 断言 `media.quality == MediaParams::default().quality`）。
- **`media.camera`**：默认 `true`——未配置的板子默认推真实摄像头而不是测试图样。

## 测试要点

`#[cfg(test)] mod tests` 共 4 个测试，全部围绕"降级行为"和"未知键容忍"这两个核心承诺：

1. **`a_quality_in_the_file_is_what_gets_streamed`** —— 文件里写了 `[media] quality = "360p30"`，断言解析出 `(640,360)`、30 fps、码率走 `quality.default_bitrate()`，并且未被触碰的 `camera` 键保持默认 `true`。验证"一个键被设置不会影响其他键"。
2. **`a_missing_file_streams_the_defaults`** —— 指向一个不存在的文件，且 `explicit=false`，断言拿到默认 quality 和 `camera=true`。对应"未初始化板子静默推流"的承诺。
3. **`a_broken_file_still_streams`** —— 写入一个语法损坏的 TOML（`[media\nquality = ` 这种半截），断言 `load` 仍然返回默认 `Params`。这是把文件头 doc comment 里的核心主张**钉死**成测试："`robotd` 拒绝启动的那个坏文件，仍然给机器人留下一个可供查看的摄像头"。
4. **`a_key_from_another_build_does_not_cost_the_ones_this_build_has`** —— 文件里写了当前 build 不认识的键 `chroma_subsampling = "4:4:4"`，断言本 build 认识的 `quality = "720p15"` 仍然生效。注释明确提到这复刻了"曾经阻止一个分支上的 `[chorale]` 段落把机器人压垮"的教训——未知键必须**逐键忽略**，不能整段丢弃。

测试共同使用 `tempfile::tempdir()` 在临时目录里写 TOML，不依赖真实 `/etc/robot/robotd.toml`。

## 与其他模块的关系

- **`robotd_params`（外部 crate）**：真正的 schema 拥有者。`Params`、`MediaParams`、`DEFAULT_PATH` 全部来自它。`mediad` 与 `robotd` 共享这个 crate，保证"编辑器能写的，守护进程一定能读"。
- **`main.rs`**：调用 `load()`，决定 `explicit` 取 `true` 还是 `false`（取决于用户是否传了 `--config`）。`main` 是 Linux-only，所以本模块独立存在的意义就是绕开它。
- **`pipeline`（Linux-only）**：拿到 `Params` 后，由 pipeline 把 `[media]` 翻译成 GStreamer 管线（分辨率/帧率/码率），把 `[detect]` 喂给检测模块。
- **`web` / `producer`**：不直接依赖本模块。`web` 用自己的端口参数；`producer` 从 `configd` 取名字，不读 TOML。
- **`detect`**：本模块负责把 `[detect]` 段也解析进 `Params`，`detect` 模块再取用。注释说"`[detect]` 归这个守护进程管，因为帧是从本守护进程的 tee 上分叉出来的"。

## 设计哲学小结

这个文件虽小，却浓缩了整个 mediad 的可靠性取向：**机器人已经出问题的时候，媒体通道不能再成为第二个问题**。配置 schema 让渡给共享 crate（保证跨工具一致）、解析失败降级到默认值（保证可观测性不丢）、未知键逐键容忍（保证跨版本升级不坏）、独立成模块（保证在开发机上可测）——每一条都是为了让"看一眼机器人"这件事尽可能少地依赖配置文件本身是健康的。
#（注：内容由AI生成）
