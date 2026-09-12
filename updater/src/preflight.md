# preflight.rs 文件解析

## 文件位置

`d:\microduck\updater\src\preflight.rs`

## 定位

下载或变更前检查的前置条件。每个失败都**无副作用**地干净中止。

## `Check` 枚举

| 检查 | 说明 |
|---|---|
| `Clock` | 时钟可信（无 RTC 板子启动时钟错，HTTPS 证书日期校验失败） |
| `RobotStopped` | 不在运动中 |
| `NoRemoteSession` | 无远程会话 |
| `DiskSpace` | 下载+提取+保留版本的空间 |
| `SideloadDir` | `--from` 目录本进程可读（`PrivateTmp=yes` 使 /var/tmp 是私有命名空间） |

## 两次运行

每次 apply 运行两次：
1. **无 manifest**：时钟、机器人停止、无会话——在任何网络访问前（manifest 获取是 HTTPS，未同步时钟会以不透明 TLS 错误失败）
2. **有 manifest**：磁盘空间检查（需求来自 manifest 的 size）

## `Clock` 细节

`CLOCK_FLOOR_UNIX = 2025-01-01`。无电池 RTC 板子在 epoch（或镜像构建日期）启动，这捕获"从未同步 NTP"的情况，无需与 timedatectl 对话。

## `Preflight::run()` — 不短路

故意不短路：一次告诉用户"时钟错 AND 磁盘满"比修一个重试一次好。

## 关键摘要

preflight.rs 在下载前检查前置条件：时钟（防 HTTPS 证书日期失败）、机器人停止、无远程会话、磁盘空间、sideload 目录可读性；运行两次（无 manifest 时网络前、有 manifest 时空间检查）；不短路以一次报告所有失败。
