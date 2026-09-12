# updater_gate.rs 文件解析

## 文件位置

`d:\microduck\robotd\tests\updater_gate.rs`

## 核心设计决策

### 1. 针对**真实 `robotd` 进程**的健康门测试

`updater` 的 `apply.rs` 用进程内 `FakeRobot` 测试引擎决策，但 `FakeRobot` 永不序列化，无法抓住线协议分歧（重命名字段、socket 模式错误、接受连接却不回答）。本文件 spawn 真实二进制。

### 2. 放在 `robotd/tests/` 而非 `updater/tests/`

只有在本包内 cargo 才定义 `CARGO_BIN_EXE_robotd` 并保证二进制先重建。在 `updater/tests/` 里路径需手猜，且 `cargo test --test <name>` 不重建兄弟二进制——会静默运行陈旧 robotd，曾产生过一次自信的错误结果。

### 3. `on_apply = restart` 故意不在范围内

容器内无 systemd，stub 它等于测试 stub。这部分落到硬件上测（roadmap M4）。

## 关键结构

### `Robotd` 子进程管理

- `spawn(socket, extra_args)`：以 `--fake --no-policy` 启动真实 robotd，等待 socket 应答。
- `wait_until_answering`：等 socket 回答任何东西（健康或不健康）。
- `wait_until_healthy`：等控制循环真正开始（socket 比第一 tick 先起来，故有一段正确报告「未完成一周期」的窗口）。
- `health()`：原始 JSON 取 `HealthResult`（含 battery，而 `SocketRobotClient::health()` 只取三态裁决）。
- `Drop`：kill 子进程——失败的测试 unwind 也会清理，避免残留 daemon 占 socket。

### `Fixture`

临时目录 + 签名发布器 + 配置模板。`engine()` 用真实 `SocketRobotClient`，门超时 3s（真机 30s）。

## 测试

### 线协议层

- **`real_robotd_answers_every_method_the_engine_calls`**：运行中的 robotd 必须 `Healthy`、`safe_to_restart==Yes`、`model_api==Some(1)`、无远程会话。这是防止重命名字段导致永远 `Unreachable` 而回退所有更新的唯一防线。
- **`real_robotd_reports_a_mapped_battery`**：电池从 `RobotIo::bus_voltage`→EMA→原子→volts→percent→JSON 全链路验证。FakeIo 报 7.4V→50%。单元测试直接写原子，若采样器根本没被调用也会全过——这正是本测试要抓的。
- **`unhealthy_robotd_reports_a_reason_not_unreachable`**：`--unhealthy` 必须是 `Unhealthy(reason)` 而非 `Unreachable`——回退理由必须准确。
- **`absent_robotd_is_unreachable_and_does_not_hang`**：缺 socket 立即失败，不等超时。

### M1 完成测试

- **`apply_gates_against_a_healthy_robotd_and_commits`**：健康 robotd 下 apply 提交，live_version 变 1.0.0。
- **`unhealthy_robotd_triggers_automatic_rollback`**：先 1.0.0 成功，再用 `--unhealthy` 模拟坏版本 1.1.0，apply 应 `RolledBack` 且 content（不仅是 symlink）回到 1.0.0。

### 版本上报

- **`real_robotd_reports_its_own_version_over_hello`**：`hello` 须报告自身 crate 版本（`robotctl version` 读的），版本不能只存在日志里。

## 关键摘要

`updater_gate.rs` 是唯一跨真实 socket 验证 robotd↔updater 线契约的测试套件。它 spawn 真实 robotd（`--fake --no-policy`），覆盖：所有引擎调用方法可达、电池全链路映射、`--unhealthy` 准确到达、健康门提交与自动回滚（content 层面）、版本通过 `hello` 上报。放在 `robotd/tests/` 是为了 `CARGO_BIN_EXE_robotd` 的重建保证。
