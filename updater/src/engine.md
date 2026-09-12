# engine.rs 文件解析

## 文件位置

`d:\microduck\updater\src\engine.rs`

## 状态机

```text
preflight → fetch manifest → verify sig → compatibility
  → download → verify hash → verify sig
  → extract → [pre hook] → ATOMIC SWAP → [post hook] → apply
  → HEALTH GATE → healthy ? commit+prune : ROLLBACK
```

三条规则塑造一切：
1. **交换后任何失败都回滚。** 钩子失败、健康失败、超时是同一结果——没有"大部分应用"。
2. **签名与哈希都通过前不提取到活路径。**
3. **启动计数器在交换前 armed**，使交换与健康检查之间的崩溃仍可恢复。

## 关键常量

| 常量 | 值 | 含义 |
|---|---|---|
| `MAX_BOOT_ATTEMPTS` | 2 | 待证明启动次数 |
| `ROBOT_QUERY_TIMEOUT` | 2s | robotd 查询超时 |
| `APPLY_ACTION_TIMEOUT` | 30s | systemctl 动作超时 |
| `HOOK_TIMEOUT` | 120s | post 钩子超时 |
| `PRE_INSTALL_HOOK_TIMEOUT` | `UPDATE_MAX_SILENCE_SECONDS` | pre 钩子超时（可装 ONNX/GStreamer，~100MB apt），长因交换前旧版本仍活 |
| `HEALTH_POLL_INTERVAL` | 500ms | 健康探针间隔 |
| `PROGRESS_MIN_GAP` | 250ms | 下载进度最小间隔（BLE 20 字节管道） |
| `SPACE_MULTIPLIER` | 3 | 空间余量倍数 |
| `LOG_CAPACITY` | 200 | 更新日志条目数 |
| `TRANSCRIPTS_KEPT` | 20 | transcript 保留数 |
| `SUPPORTED_SCHEMA_VERSION` | 1 | 引擎理解的最高 schema |

## `Engine` 结构体

持有 config、keys（Arc，可传给 spawn_blocking）、robot、journal、boot_counter、pins、faults、deferred_restarts、unit_dir。

## `Recorder`

统一进度与 transcript 记录——避免"phase 到达订阅者但不到磁盘"的分叉。`phase()` 同时发 ProgressTx 与 transcript；`note()` 仅 transcript；`progress_tx()` 给下载泵用百分比。

## 主流程 `apply()`

1. `UpdateLock::try_acquire`（文件锁，单飞）
2. `Recorder::begin`
3. `apply_inner`：
   - **preflight（无 manifest）**：时钟、机器人停止、无远程会话——在网络前（HTTPS 证书日期）
   - 获取 manifest（按 Target）+ 验证签名
   - 检查 channel、pin、是否已安装、降级防护（仅 Latest）、staging 落后、兼容性
   - **preflight（有 manifest）**：磁盘空间
   - **下载**到 staging，`DownloadProgress` 按 gap+去重节流
   - **验证**：sha256 + minisign 签名（spawn_blocking）
   - **提取**到 extract_dir（spawn_blocking，含路径穿越防护、大小/条目限制）
   - 写入 embedded manifest
   - **orphan 检查**：候选是否会让已安装 unit 无可执行文件
   - dry_run → 返回
   - **pre hook**（PRE_INSTALL_HOOK_TIMEOUT）
   - **arm boot counter** → `rename` 发布 release → `swap_to`（原子 symlink）
   - 故障 `abort_after_swap` 模拟 kill -9
   - **post_swap**：post hook → apply action → self-test updaterd → health gate
   - 健康通过 → `boot_counter.confirm` + prune；失败 → `rollback_to`
4. `record` 到 journal
5. 释放锁（在 spawn 前，避免 fork 复制锁描述符）
6. `schedule_restarts_if_needed`（延迟重启 updaterd/btd）
7. `rec.finish`

## 其他方法

- `check(component)` — 是否有更新可用
- `status()` — 各组件状态（phase 总 Idle，因引擎被更新持有时用 try_lock）
- `list_installed` / `log` / `show` / `known_bad`
- `recover_on_start()` — 启动恢复
- `reconcile_running_units()` — 检查延迟重启是否发生
- `rollback_to` / `select` / `reset_to_golden` / `pin`

## 关键摘要

engine.rs 是更新状态机核心：preflight→manifest→download→verify→extract→pre-hook→原子交换→post-hook→apply→health-gate→commit/rollback；boot counter 在交换前 armed 保证崩溃可恢复；锁在 spawn 前释放避免 fork 复制；下载进度按 gap 节流适配 BLE。
