# systemd-test.sh

## 文件位置

`d:\microduck\scripts\systemd-test.sh`

## 核心设计决策

针对真实 systemd 驱动一次真实更新，按需手动运行（非 CI）。

- **只有本脚本能回答的三个问题**：仓库其他测试都用 stub `systemctl` 测试引擎的决策（会重启哪些 unit、顺序、缺失时行为），但都看不到重启是否真的发生。三个机制在别处完全未测：
  1. `on_apply` 真的重启 release 附带的 unit；
  2. 延迟的 `systemd-run --on-active=5s` 瞬态 timer 真的触发并替换 `updaterd`（子进程做不到，因为会在被杀的 cgroup 里）；
  3. `hooks/postinstall` 真的安装、启用、启动板端从未有过的 unit。
- **只有 systemd 能产生的失败**：unit 安装干净但**无法启动**（`install-path-gap.md` 的 bug 1，板端症状是 `not healthy within 30s: unreachable`，既不点名 unit 也不点名命令）。
- **选 Docker 而非 systemd-nspawn**：`install-path-gap.md` 原计划用 nspawn 并在 CI 中拒绝特权容器（拒绝立场不变，本脚本非 CI）。Docker Desktop 能在笔记本上跑 systemd 为 pid 1 的特权 arm64 容器，nspawn 做不到，而这里重要的保真度（真实 unit、真实 cgroup、真实瞬态 timer）相同。
- **不进 CI**：需要 `--privileged` 和主机 cgroup 文件系统，代价大于任何检查的价值，且 `board` 任务已是最慢的。只在 `on_apply`、延迟重启或 hook 变化时运行。

## 常量/路径

| 名称 | 值 | 说明 |
|---|---|---|
| `WORK` | `target/systemd-test` | 工作目录 |
| `FIXTURE` | `$WORK/fixture` | 签名 release fixtures |
| `MOUNT` | `/fixture` | 容器内挂载点 |
| `IMAGE` | `duck-systemd-test` | 容器镜像名 |
| `NAME` | `duck-systemd-test` | 容器名 |
| `BIN` | `target/docker/release` | 容器内构建的二进制 |

## 辅助函数

- `pass`/`fail`：输出 `[ok]`/`[FAIL]`，fail 退出。
- `cleanup`：`docker rm -f`，trap EXIT/INT/TERM。
- `in_container CMD`：`docker exec NAME sh -c CMD`。
- `live()`：读 `current` 符号链接得出版本号。
- `main_pid UNIT`：`systemctl show -p MainPID`。
- `apply LOGFILE`：用 `current` 里的 robotctl 执行 `update apply daemon`。

## 执行流程

### 1. 构建二进制

在容器内（与运行环境相同 userland，复用 `dev-build.Dockerfile`）构建 updaterd 和 robotctl。arm64 主机上是原生构建。

### 2. mint fixtures

`test-support/examples/systemd-fixture` 生成：
`1.0.0 1.1.0 1.2.0 1.3.0:sysusers 1.4.0:missing-user 1.5.0:broken-unit`

初始 published 放 1.0.0。

### 3. 启动 systemd 容器

`docker run -d --privileged --cgroupns=host -v /sys/fs/cgroup`，入口 `/lib/systemd/systemd`。
轮询 `systemctl is-system-running` 直到 `running` 或 `degraded`。

### 4. 首次安装（bootstrap 路径）

`updaterd install --from published`（从树而非 release 运行，因板端裸机无安装）。
断言 postinstall 安装了板端从未见过的 `fake-robotd.service` 并启动，`updaterd` 运行，btd stand-in 发布 identity。

### 5. apply 1.1.0（通过运行中的 updaterd）

- 断言 `outcome: applied`，current=1.1.0。
- `on_apply` 重启了 release 附带的 daemon（fake-robotd PID 变化）。
- **未重启**只应由 timer 启动的 recovery-check（无 `[Install]`，由 timer 触发的 oneshot）。
- **延迟重启**：updaterd PID 在 ~5s 后变化（瞬态 timer 触发）。
- 重启后的 updaterd 运行 1.1.0（identity.json 指向 `releases/1.1.0/bin/updaterd`）。
- 启动协调（reconciliation）未重启任何东西。

### 6. injection：systemd-run 不可执行

在 `/usr/local/sbin/systemd-run` 放 `exit 1` 的 shadow（PATH 优先于真实 systemd-run）。
- apply 1.2.0 仍成功（延迟重启是 best-effort，update 已提交，不能因 timer 创建失败报失败）。
- journal 有 "until the next updaterd start notices"。
- btd 和 updaterd 留在 1.1.0（静默状态，本脚本存在的原因）。
- 手动重启 updaterd 后，其 reconciliation 重启陈旧 btd 到 1.2.0。
- 不重启自己（self-restart 会是死循环）。

### 7. injection：unit 带 release 自带的 User=

1.3.0 (sysusers) 携带 unit 及其所需账号。postinstall 安装 sysusers.d 并在 units 之前运行 `systemd-sysusers`。
- apply 成功，`duck-test` 用户存在，`needs-a-user` unit 运行。

### 8. injection：同名 unit 但无人创建用户

1.4.0 (missing-user) 的 unit 命名了不存在的用户。
- apply 失败（`enable --now` 在 hook 中只是警告，但 `on_apply` 的重启不是）。
- 失败原因点名 unit（`needs-a-user`）而非账号。
- current 回到 1.3.0（`rollback_to` 在重跑 apply action 前交换 current）。
- 报告为 "reverted" 而非 "rollback failed"，点名未恢复的 unit，不声称未观察的停机，提示 `robotctl health`。
- 其余 daemon 仍在运行（只是缺口而非停机）。

### 9. injection：unit 无法启动

1.5.0 (broken-unit) 携带安装后无法启动的 unit（bug 1 类）。
- apply 回滚（不报告成功）。
- current 回到 1.3.0（1.4.0 从未 live）。
- 原因点名 unit（"broken"）而非 "unreachable"。
- rollback 后 daemon 运行。

## 关键要点总结

1. **exit status 不是判定**：rollback 是成功调用报告不成功结果，robotctl 退出 0，读 status 会断言错误的契约。
2. **延迟重启是 best-effort**：因为不是最后手段（下次 updaterd 启动会 reconcile 陈旧 unit），所以不可调度时不报失败。
3. **accounts before units**：postinstall 先装 sysusers.d 再 `systemd-sysusers` 再 units，否则命名不存在 User= 的 unit 启动失败，表现为 daemon 损坏。
4. **重启而非启动的区分**：hook 中 `enable --now` 失败只是警告，`on_apply` 的重启不是，这是 `restart_one` 刻意的区分。
5. **不声称未观察的停机**：所有 daemon 有 `Restart=always`，拒绝重启的 unit 几秒后会回来，所以 outcome 报告拒绝并给出 `robotctl health`，不说"有东西宕了"。
