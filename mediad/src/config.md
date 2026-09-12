# config.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\config.rs`

## 核心设计决策

本守护进程"流化什么、寻找什么"，全部来自 `robotd` 已经在读取的配置文件 `/etc/robot/robotd.toml` 中的 `[media]` 与 `[detect]` 段。schema、默认值与校验都属于 `robotd_params`，目的是：`robotctl configure` 通过同一 crate 写，`mediad` 通过同一 crate 读，编辑器不可能提供本守护进程不理解的值。

之所以单独成模块而不是放在 `main` 里写四行：`main` 是 Linux-only，任何住在里面的东西在编写机器上都不会被编译——更不会被测试。

## 函数

### `default_path() -> PathBuf`
返回 `robotd_params::DEFAULT_PATH`，即 `/etc/robot/robotd.toml`。

### `load(path: &Path, explicit: bool) -> Params`
读取文件，失败则回退到内置默认值。关键设计点：

- **本守护进程读不了文件不是没有视频的理由**。`robotd` 在损坏参数文件上拒绝启动（那是大信号，且它是被该文件配置的控制循环所在的守护进程）；处于那种状态的机器人已经宕机，而它的摄像头正是有人查看它的方式。所以这里只 `warn`、点名文件，然后流化默认值，而不是加入停机。
- 默认路径下**文件缺失连警告都不算**：未配板的板子没有文件，以 720p30 流化摄像头即可。
- 命令行显式指定的路径必须存在——这是 `Params::load` 自身的规则，也是 `explicit` 被透传的原因。

## 单元测试

| 测试 | 意图 |
|---|---|
| `a_quality_in_the_file_is_what_gets_streamed` | 设置 `quality = "360p30"` 后实际以 640×360 @30 流化，其余键保持默认 |
| `a_missing_file_streams_the_defaults` | 默认路径无文件时流化默认值且不报错 |
| `a_broken_file_still_streams` | 损坏的 TOML（`[media` 未闭合）仍流化默认值——印证 doc 注释的主张 |
| `a_key_from_another_build_does_not_cost_the_ones_this_build_has` | 其他构建的未知键（如 `chroma_subsampling = "4:4:4"`）被逐键忽略，本构建认识的键仍生效 |

## 关键摘要

- 与 `robotd` 共享同一参数文件与同一解析 crate，杜绝编辑器/读取器漂移。
- 读失败 → 回退默认值而非拒绝启动，因为摄像头是观察宕机机器人的手段。
- 未知键逐键忽略，不是整段丢弃。
