# lib.rs 文件解析

## 文件位置

`d:\microduck\updater\src\lib.rs`

## 核心定位

`updater` 是配置驱动的更新引擎库，**机器人无关**——所有机器人特定逻辑都在 `config::Config` 中。适配新机器人应只需新配置文件、新签名密钥、可能的新健康探针，而非 fork 本 crate。

## 模块结构

```
config  engine  faults  fsutil  hooks  ipc  journal  manifest
orphan  preflight  reconcile  robot  source  spawn  store  transcript  verify
pub use duck_ipc_proto as proto;   // IPC 契约重导出
```

## 明确的非目标

- **不提前抽象运行时。** 仅在真正等待处用 async（IPC 服务、超时钩子/健康探针、取消更新）；CPU 密集与快速文件系统工作（store/verify/journal）保持同步，通过 `spawn_blocking` 调用。
- **不提前拆 crate。** 仅当第二个服务需要线类型时才把 proto 移出到 `duck-ipc-proto`；`robotd`/`robotctl` 依赖 proto 而非本 crate，使恢复路径不继承 http/tar/zstd/crypto 依赖树。
- **不做 OS/内核更新。** 仅应用层。
- **无硬件能力矩阵。** v1 目标单一硬件配置。

## `Error` 枚举

每个变体映射到一个不同的 JSON-RPC 错误码，因为"更新失败"单独对客户端与支持毫无用处。关键变体：

| 变体 | 含义 |
|---|---|
| `UnknownComponent` | 配置中无此组件 |
| `NotInstalled` | 组件已配置但该版本不在磁盘上 |
| `WouldDowngrade` | 拒绝降级（回滚攻击防护，仅 `Latest`） |
| `StagingBehind` | staging 通道无更新版本（与 WouldDowngrade 不同——源没问题，通道落后） |
| `WouldOrphanUnit` | 候选不包含已安装 unit 的二进制 |
| `Busy` | 另一更新进行中（脚本可重试） |
| `ReleaseNotReady` | 发布存在但尚无资产（构建未完成上传） |
| `Verification` | 签名/哈希不匹配，永不自动重试 |
| `SelfTest` | 新 updaterd 自检失败（区别于 Health） |
| `RollbackFailed` | 更新失败且回滚失败——最严重结果，单独暴露 |
| `ArchiveTooLarge` | 工件超限（区别于 Verification，非篡改） |
| `NoSuchRun` | 无此 transcript，携带可用列表 |

`code()` 方法将变体映射到 `proto::code` 常量；部分变体共享代码（如 `StagingBehind` 与 `WouldDowngrade` 共享 `WOULD_DOWNGRADE`），因为对客户端是同一答案，区分在消息中。

## 测试

- `busy_has_its_own_code` — Busy 不得像内部错误（脚本依赖它重试）。
- `verification_failure_is_distinct_from_network` — 篡改不得被当作网络故障自动重试。

## 关键摘要

lib.rs 定义引擎的错误类型体系：每个失败变体对应明确的客户端可操作错误码，区分降级攻击、签名失败、回滚失败等关键场景；非目标声明确立了"仅在等待处异步、不提前拆分"的设计原则。
