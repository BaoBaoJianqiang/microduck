# power.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/power.rs`
- **角色**：实现 `system.reboot` 的重启动作。核心立场是**经由 systemd（logind）重启，而不是绕过它直接调 `reboot(2)`**。

## 2. 为什么不能直接 `reboot(2)`（模块文档）

- `reboot(2)` 只是一个系统调用，但在此场景是错的：它直接断电、不停服务，于是 `robotd` 永远不会释放舵机扭矩，更新日志也来不及刷盘。
- 「一台自己摔倒式重启的机器人」不是 `system.reboot` 可接受的实现。
- logind 的 `Reboot` 受 polkit 门控，而**这块板上没有 polkit**，无会话的非 root 调用者会被直接拒绝——这就是 `configd` 以 root 运行的全部理由。这是一个狭窄的、被沙箱约束的 root（见 `systemd/configd.service`）；另一条路是装一个 JS 策略引擎只为授权一个调用。

## 3. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `REBOOT_DELAY` | `Duration::from_secs(3)` | **应答与真正重启之间的 3 秒间隔**。它是承重设计而非礼貌：若在调用内部直接重启，连接会在响应发出前被断开，手机等客户端只能把断管理解成成功。3 秒够响应经 BLE 以每通知 20 字节分片送出，又短到没人会怀疑是否生效 |

## 4. 函数逐项解析

### `pub fn schedule()`

「先应答，稍后重启」。

- `tokio::spawn` 一个后台任务：先 `sleep(REBOOT_DELAY)`，再以 `warn` 级日志记录「应 IPC 请求正在重启」，然后调用 `reboot()`。
- 若重启失败：记录 `error`「reboot failed; the robot is still running」。失败后没有别的办法——重启失败会让机器人继续运行，这是安全方向，但理由必须进入日志，否则这次调用看起来像被悄悄忽略。
- `main.rs` 在 `SystemReboot` 分支中先调用 `power::schedule()`，再立即回复 `RebootResult { in_seconds: REBOOT_DELAY.as_secs() }`。

### `async fn reboot() -> Result<(), String>`（两个 `cfg` 版本）

- **Linux 版**（`#[cfg(target_os = "linux"))]`）：
  1. 连接系统总线 `zbus::Connection::system()`。
  2. 向 `org.freedesktop.login1` 的 `/org/freedesktop/login1` 路径、`org.freedesktop.login1.Manager` 接口调用 `Reboot`，参数为 `false`——含义是「非交互」，logind 不会试图询问任何人。
  3. 总线错误被格式化为 `logind refused: {e}` 返回。
- **非 Linux 版**（`#[cfg(not(target_os = "linux"))]`）：直接返回 `Err("not rebooting: this is not the robot")`。注释：因为测试调了 `system.reboot` 就把开发者笔记本重启，会是学习 `cfg` 的一种难忘方式。

## 5. 要点小结

- 优雅重启走 logind D-Bus `Manager.Reboot(false)`，保证服务有序停止、舵机卸力、日志落盘。
- 先回复、延迟 3 秒再重启，让 BLE 客户端能确切收到成功应答。
- 重启失败只记日志不做别的：机器人继续运行是安全方向。
- 非 Linux 下永远返回错误，保护开发机；这也是 `configd` 需要 root 的唯一直接原因。
