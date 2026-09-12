# ipc.rs 文件解析

## 文件位置

`d:\microduck\updater\src\ipc.rs`

## 定位

unix socket 上的 JSON-RPC 2.0 服务器，供 `robotctl` 与 `btd` 使用。NDJSON 分帧。

## 架构要求

- 服务从不依赖 robotd 存活
- 慢/消失的客户端不得延迟进行中的更新
- 所有客户端断开后更新仍完成（机器人拉取，BLE 中途断开是正常）

结构：每连接一个 task，`Engine` 在 Mutex 后。长操作持锁；只读请求用 `try_lock` 回退到缓存快照——使 `status`/`subscribe` 在更新期间仍可应答。

## 访问控制

**socket 文件模式即访问控制。** socket 创建为 `0o660` 组所有，每个变更请求记录调用者 uid/pid（`SO_PEERCRED`）。

`PeerPolicy` 两层：
1. socket 组（0660）— 谁可**对话**
2. allow_uids/allow_gids — 谁可**变更**（updaterd 自身 uid 总允许）

只读请求（status/log/listInstalled/check/subscribe）不受第二层门控——支持需能查看机器人而不需被授权变更它。

## 关键常量

- `PROGRESS_BUFFER = 256` — 广播缓冲，慢订阅者落后则丢弃（进度是建议性的）
- `SOCKET_MODE = 0o660`
- `MAX_LINE = 1MB`
- `INITIAL_CHECK_DELAY = 60s` — 首次调度检查延迟（开机网络常未就绪，避惊群）

## `Server` 方法

- `with_policy(engine, allow_uids, allow_gids)` — 构造
- `serve(socket)` — 绑定、设权限、accept 循环
- `spawn_periodic_checks(interval, auto_apply)` — 周期性检查调度器（使 min_supported 生效）

## 关键摘要

ipc.rs 是 updaterd 的 JSON-RPC 服务器：Engine 后 Mutex + try_lock 使只读在更新期间仍可应答；两层访问控制（socket 组=对话权，allow_uids/gids=变更权）；进度广播慢消费者丢帧不反压；周期性检查调度器使无人值守强制升级生效。
