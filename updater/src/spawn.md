# spawn.rs 文件解析

## 文件位置

`d:\microduck\updater\src\spawn.rs`

## 问题

内核拒绝 exec 任何进程已打开写的文件（按 inode 跟踪，`ETXTBSY`）。updater 提取发布（写 `hooks/preinstall`）后片刻执行它，同时还在 spawn 其他子进程（另一钩子、systemctl、健康探针命令）：fork 时子进程继承整个 fd 表的副本，仅在自身 execve 时丢 `O_CLOEXEC` 描述符，所以几微秒内某无关子进程持有要运行文件的写句柄，exec 失败"Text file busy"。

未处理→失败的钩子→失败的更新→回滚本来正常的发布——罕见、不可复现、报告为"RunningPreHook 失败"无可操作信息。

## `retrying_busy(command)`

重试 `ETXTBSY`（10 次，每次 10 ms，共 ~100 ms）。重试是唯一解：冒犯的描述符属于**不同**进程，关闭/同步自己的无用，rename 保持同 inode。

**本 crate 所有 spawn 都经过此处**，包括从不写的程序（如 systemctl）——因为测试套件写 stub systemctl 并从并行线程执行，同样的竞态曾让只改文档的 PR 红 CI。源 grep 测试强制此约束。

## 测试

`every_spawn_in_the_crate_goes_through_the_retry` — 源码 grep 验证除 spawn.rs 外无 `.output()`/`.spawn()`/`.status().await` 直接调用。

## 关键摘要

spawn.rs 用 `retrying_busy` 统一所有子进程 spawn，重试 `ETXTBSY`（fork 继承 fd 导致 exec 时文件仍被写打开）：10 次×10ms；源 grep 测试强制所有 spawn 走此路径。
