# Cargo.toml 文件解析

## 文件位置

`d:\microduck\robotctl\Cargo.toml`

## 定位

`robotctl` 是机器人的本地 CLI。独立 crate 而非 `updater` 的 binary——角色更广（将 front `robotd` 等），但目前只实现 `update` 命名空间。

## 依赖策略

**核心规则**：仅依赖 `duck-ipc-proto`，不依赖 `updater`。提取协议的目的：恢复路径上的工具不应链接更新引擎的 http/tar/zstd/crypto 树，且结构上无法触及引擎内部而只能走 socket。

## 依赖列表

| 依赖 | 用途 | 恢复路径考量 |
|---|---|---|
| `duck-ipc-proto` | IPC 契约 | 纯协议，零重型依赖 |
| `clap` + `clap_complete` | CLI 解析+补全 | 纯 codegen，无运行时 |
| `semver` | 版本比较 | |
| `serde`/`serde_json` | 序列化 | |
| `libc` | `SIGPIPE`/`SIGINT` | |
| `kinematics` | ToF 重投影几何 | **故意不用 `tof` crate**（其 vendor 的 C 驱动会拖跨 C 工具链进每次 robotctl 构建） |
| `robotd-params` | `configure` 的 schema | 与 `robotd` 解析同一文件，纯 serde+toml |
| `toml` | 配置解析 | |
| `toml_edit` | 无损编辑 robotd.toml | 纯 `toml` 会破坏注释 |
| `ratatui` | `monitor`/`configure` 终端 UI | 无 http/crypto/async 运行时 |

## [dev-dependencies]

仅 `tempfile`（configure 测试写临时文件）。

## 关键摘要

Cargo.toml 驱动 robotctl：故意仅依赖 proto 而非 updater（恢复路径不链接引擎重型依赖树）；用 kinematics 几何而非 tof crate（避免 C 工具链）；toml_edit 保留注释；ratatui 纯终端无 async；clap_complete 从同一 Cli 树生成补全防漂移。
