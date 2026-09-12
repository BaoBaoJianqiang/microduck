# tests/apply.rs 文件解析

## 文件位置

`d:\microduck\updater\tests\apply.rs`

## 定位

更新状态机的端到端测试，基于 `LocalDir` 源。Tier-1 机制测试：驱动**真实引擎代码路径**无网络无机器人，不能漂移。

回滚是最可能被悄悄破坏的（仅在出错时运行），所以大部分测试故意破坏东西。

## 关键设计

- `FakeRobot` — 健康状态可变的测试替身（真实坏发布场景：旧版本健康，新版本病）
- `AbsentRobot` — robotd 现在不在，但好发布链接后会起来
- 通过 `Faults` 注入故障验证回滚路径

## 覆盖场景

- 正常 apply → Applied
- 健康失败 → RolledBack
- 工件篡改 → 验证失败不安装
- post 钩子失败 → 回滚
- abort_after_swap → kill -9 后恢复一致
- 降级防护（WouldDowngrade）
- staging 落后（StagingBehind）
- orphan unit 拒绝
- pin 版本
- select/rollback/reset-to-golden
- known_bad 版本不被无人值守重试

## 关键摘要

apply.rs 是更新引擎的 Tier-1 机制测试套件：用 LocalDir 源+FakeRobot+Faults 驱动真实代码路径，覆盖正常 apply、各类失败回滚、降级防护、orphan 拒绝、pin/select/rollback 等全状态机路径。
