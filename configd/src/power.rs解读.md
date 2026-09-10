# `power.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 60 行 |
| 角色 | 重启——通过 systemd/logind 而非直接 `reboot(2)` |
| 平台 | Linux 运行；非 Linux 拒绝（防误重启开发机） |

## 二、核心设计：为什么不用 `reboot(2)`

### `reboot(2)` 的问题

- `reboot(2)` 是一个系统调用，直接切断电源而**不停止服务**。
- 后果：
  - `robotd` 永远不会释放舵机扭矩——机器人会摔倒。
  - 更新日志永远不会刷新——可能丢失更新状态。
  - 其他守护进程没有机会优雅退出。
- "摔倒着重启的机器人"不是 `system.reboot` 的可接受实现。

### logind 的 `Reboot`

- 通过 D-Bus 调用 `org.freedesktop.login1.Manager.Reboot(false)`。
- logind 会：
  1. 通知所有服务优雅退出。
  2. 卸载文件系统。
  3. 然后才重启。
- `false` 参数表示"非交互式"，logind 不会尝试询问任何人。

### 为什么 configd 以 root 运行

- logind 的 `Reboot` 是 polkit 门控的。
- **此板上没有 polkit**——因此无会话的非 root 调用者直接被拒绝。
- 这就是 `configd` 以 root 运行的全部原因。
- 它是一个窄的、沙箱化的 root（见 `systemd/configd.service`），替代方案是安装一个 JS 策略引擎来授权单个调用——不值得。

## 三、常量

### `REBOOT_DELAY = 3 秒`

- 回答请求和实际重启之间的间隔。

#### 承重设计而非礼貌

- 如果守护进程在调用内部重启，会在响应之前断开连接。
- 每个客户端（尤其是手机）将不得不把"管道破裂"当作成功处理。
- 3 秒足够：
  - 响应通过 BLE 以每次通知 20 字节分块发出。
  - 短到没有人会怀疑是否成功。

## 四、`schedule()` 函数

```rust
pub fn schedule() {
    tokio::spawn(async {
        tokio::time::sleep(REBOOT_DELAY).await;
        tracing::warn!("rebooting now, as requested over IPC");
        if let Err(e) = reboot().await {
            tracing::error!(error = %e, "reboot failed; the robot is still running");
        }
    });
}
```

### 设计要点

- **先回答，后重启**：调用者立即得到 `Ok`，3 秒后才实际重启。
- **spawn 独立任务**：不阻塞调用者的连接。
- **失败处理**：
  - 重启失败意味着机器人仍在运行——这是安全方向。
  - 但原因必须到达日志，否则这看起来像被静默忽略的调用。
  - 记录 `error` 级别日志。

## 五、平台特定实现

### Linux：`reboot()`

```rust
async fn reboot() -> Result<(), String> {
    let bus = zbus::Connection::system().await?;
    bus.call_method(
        Some("org.freedesktop.login1"),
        "/org/freedesktop/login1",
        Some("org.freedesktop.login1.Manager"),
        "Reboot",
        &(false),
    ).await?;
    Ok(())
}
```

- 通过 `zbus` 连接系统 D-Bus。
- 调用 logind 的 `Reboot` 方法，参数 `false`（非交互式）。
- 错误格式化为 `"logind refused: {e}"`。

### 非 Linux：`reboot()`

```rust
async fn reboot() -> Result<(), String> {
    Err("not rebooting: this is not the robot".to_owned())
}
```

- **故意拒绝**：因为测试调用 `system.reboot` 而重启开发者的笔记本会是"学习 cfg 的难忘方式"。
- 返回错误而非静默忽略。

## 六、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `main.rs` | `system.reboot` 调用 `power::schedule()` |
| `configd.service` | 以 root 运行（沙箱化），才能调用 logind |
| `btd` | 通过 `system.reboot` 转发手机应用的重启请求 |
| `robotd` | 重启前优雅退出，释放舵机扭矩 |
| `updaterd` | 重启前刷新更新日志 |
| `logind` | 实际执行重启的系统服务 |

## 七、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **优雅重启** | 通过 logind 而非 `reboot(2)`，确保服务有机会退出 |
| **先回答后执行** | 3 秒延迟，确保客户端收到响应 |
| **最小 root** | configd 以沙箱化 root 运行，仅用于调用 logind |
| **跨平台安全** | 非 Linux 拒绝重启，防止误重启开发机 |
| **失败可观测** | 重启失败记录 error 日志，不静默忽略 |
#（注：内容由AI生成）
