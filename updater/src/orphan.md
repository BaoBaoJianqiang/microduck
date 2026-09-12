# orphan.rs 文件解析

## 文件位置

`d:\microduck\updater\src\orphan.rs`

## 问题

`hooks/postinstall` 安装发布携带的 unit，回滚时**故意留下**它们（下次更新重装）。这对回滚成立，但对**降级到早于引入某守护进程的发布**不成立：unit 留下，`ExecStart` 指向旧发布不含的二进制，systemd 以 `203/EXEC` 失败，导致更新失败回滚。

## `would_orphan(unit_dir, current, candidate_root) -> Vec<Orphan>`

读取 `/etc/systemd/system/*.service`，过滤出 `Exec*=` 指向组件 `current` symlink 的 unit，检查候选发布是否包含该路径。不包含→`Orphan`。

- **读活目录**：只有活目录能看到孤儿（它比安装它的发布活得久）
- **过滤 Exec* 指向 current**：排除非我们的 unit（手装或其他包）
- **失败开放**：无法读或无法解析的 unit 不产生 finding（假阴性代价=203/EXEC 回滚，假阳性=拒绝更新——后者是更新系统绝不能做错的事）
- **不在恢复路径**（rollback/reset-to-golden/select）：那些是故意向后移动的恢复路径

## `execs_under(unit, current)`

解析 unit 文件中所有 `Exec*=` 指令（不仅 ExecStart），返回指向 current 的相对路径。处理 systemd 续行（`\`）与前缀字符（`@ - : + !`）。

## `refusal(version, orphans)`

拒绝消息携带**补救方法**：`systemctl disable --now <unit> && rm <unit_file>`。降级到早于引入某 daemon 的发布本就不该运行该 daemon，所以移除是使情况真实，而非覆盖检查。

## 关键摘要

orphan.rs 检测降级是否会留下无可执行文件的已安装 unit（203/EXEC 回滚）：读活 systemd 目录、过滤指向 current 的 Exec*、检查候选是否包含；失败开放（不拒绝无法解析的 unit）；不在恢复路径；拒绝消息带明确补救命令。
