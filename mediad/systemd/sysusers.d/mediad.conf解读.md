# `mediad.conf` 解读

## 概述

systemd-sysusers 配置文件，创建 `mediad` 系统用户和组。

安装到 `/usr/lib/sysusers.d/mediad.conf`；`systemd-sysusers` 在启动时创建它们，或手动运行一次 `systemd-sysusers`。

---

## 核心行

```ini
u mediad - "Robot media and WebRTC gateway" - -
```

### 格式分解

`systemd-sysusers` 的行格式：`type name id gecos home shell`

| 字段 | 值 | 说明 |
|------|-----|------|
| `type` | `u` | 创建用户*和*组（同名） |
| `name` | `mediad` | 用户名和组名 |
| `id` | `-` | UID/GID 自动分配 |
| `gecos` | `"Robot media and WebRTC gateway"` | 描述（finger 信息） |
| `home` | `-` | 无 home 目录（`/nonexistent`） |
| `shell` | `-` | 无登录 shell（`/usr/sbin/nologin`） |

**为什么 `u` 而非 `g`**：`u` 创建用户和同名组。`g` 只创建组。mediad 既需要用户（`User=mediad` in service）又需要组（`Group=mediad`），所以用 `u`。

**为什么自动分配 UID/GID（`-`）**：不固定数字 ID，让 systemd-sysusers 自动选择。这避免了与其他发行版包冲突，也不需要手动管理 ID 分配。

**为什么无 home、无 shell**：这是系统服务用户，不是真人账户。不需要登录、不需要 home 目录。

---

## 为什么 mediad 必须作为自己的用户运行

> `mediad` runs as its own unprivileged user, and here that is not only hygiene: it is the case the whole permission story was worked out for. A `mediad` running as root would open `/dev/mpp_service` and `/dev/rga` regardless of their group, so every problem the udev rule in `scripts/setup-gstreamer.sh` exists to prevent would be invisible until something else ran as a user — which is exactly how those three failures were found in the first place.

**这不仅仅是卫生问题**：

- 以 root 运行的 mediad 会不管 `/dev/mpp_service` 和 `/dev/rga` 的组权限直接打开它们
- `scripts/setup-gstreamer.sh` 中的 udev 规则存在的全部理由就是预防权限问题
- 如果 mediad 以 root 运行，这些权限问题全部不可见
- 直到*别的东西*以普通用户运行时才暴露——而那正是那三个失败最初被发现的方式

**换句话说：以非 root 用户运行是权限验证机制。** 如果以 root 运行，所有权限配置都变成装饰，永远不会被测试。

---

## 用户获得的权限全部来自组成员

> What this user gets comes entirely from group membership, granted in mediad.service:

| 组 | 权限来源 | 访问什么 |
|----|----------|----------|
| `video` | 设备节点本身（root:video 模式 0660） | VPU（`/dev/mpp_service`）、2D 加速器（`/dev/rga`）、采集节点（`/dev/videoN`） |
| `robot` | unix socket（0660） | robotd、configd、updaterd 的 socket；padd 和 tofd 自己交给 `robot` 组的 tap |

**注意**：`render`（NPU）组不在这个文件中——它在 `mediad.service` 的 `SupplementaryGroups` 中。这个文件只创建用户，组成员在 service unit 中授予。

**为什么 `render` 不在 sysusers 中**：因为 `render` 不是这个项目创建的组——它是系统预定义的 DRM 渲染组。sysusers 文件只负责创建 `mediad` 用户，不管理系统预定义组。

---

## 与其他文件的关系

- **`mediad.service`**：使用 `User=mediad` 和 `Group=mediad`，并通过 `SupplementaryGroups=video render robot` 授予额外权限
- **`scripts/setup-gstreamer.sh`**：udev 规则设置 `/dev/mpp_service`、`/dev/rga`、`/dev/videoN` 的组和模式
- **`systemd-sysusers`**：在启动时处理此文件，创建用户和组
- **`/usr/lib/sysusers.d/`**：安装位置

---

## 关键设计要点

### 1. 非 root 运行是权限验证机制

以 root 运行会绕过所有权限检查，使得 udev 规则和组配置变成装饰。以非 root 运行确保每次启动都验证权限配置正确。

### 2. 最小权限原则

- 用户本身没有任何特殊权限
- 所有访问都通过组成员授予
- `video` 组 → 硬件编解码器
- `robot` 组 → unix socket
- `render` 组 → NPU
- 没有 `CAP_SYS_RAWIO` 或其他能力

### 3. sysusers.d 的意义

使用 `sysusers.d` 而非手动 `useradd`：
- 幂等：多次运行不产生重复用户
- 在启动时自动处理：不需要安装脚本手动调用
- 与包管理集成：包安装/卸载自动创建/移除用户
- 声明式：状态在文件中，不是脚本中的命令序列
#（注：内容由AI生成）
