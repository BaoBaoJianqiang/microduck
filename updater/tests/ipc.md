# tests/ipc.rs 文件解析

## 文件位置

`d:\microduck\updater\tests\ipc.rs`

## 定位

IPC 测试：真实 `Server` 在真实 unix socket 上，由手写 JSON-RPC 客户端驱动。故意不用 robotctl——测的是**协议**，走 CLI 会混淆线行为与参数解析/输出格式化。

## 被测属性

来自 architecture.md §1.1 与 updater-design.md §7：
- socket 组限制
- 更新运行时 `status` 仍可应答
- 客户端中途断开不取消更新
- 错误码经往返存活，客户端可分支

## 夹具 `FakeRobot`

健康状态共享可变，测试可在更新间改变——真实坏发布形状（旧版本好，新版本病）。

## 关键摘要

ipc.rs 用真实 socket + 手写 JSON-RPC 客户端测试协议：验证 status 在更新期间仍可应答、客户端断开不取消更新、错误码往返存活、组访问控制；用可变健康状态模拟真实坏发布场景。
