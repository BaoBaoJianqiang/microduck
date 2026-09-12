# mediad.conf 文件解析

**文件位置**：`d:\microduck\mediad\systemd\sysusers.d\mediad.conf`

## 核心设计决策

创建 `mediad` 系统用户与组。安装到 `/usr/lib/sysusers.d/mediad.conf`，`systemd-sysusers` 在启动时创建，或手动运行一次。

`mediad` 以自己的非特权用户运行，这不只是卫生：整个权限故事就是为这个场景推演的。以 root 运行的 `mediad` 会无视组直接打开 `/dev/mpp_service` 和 `/dev/rga`，于是 `scripts/setup-gstreamer.sh` 的 udev 规则要防的所有问题都会在其他以用户运行的进程上才暴露——这正是那三类失败最初被发现的方式。

该用户得到的全部来自组成员资格（在 `mediad.service` 授予）：

- **`video`**：VPU（`/dev/mpp_service`）、2D 加速器（`/dev/rga`）、采集节点。
- **`robot`**：robotd、configd、updaterd 的 0660 socket，以及 padd、tofd 自行交给该组的 tap。

若此用户不存在，`mediad.service` 启动失败。

## 内容

```
u mediad - "Robot media and WebRTC gateway" - -
```

`u` 表示创建用户与同名组，无 UID 限定（`-`），描述为 "Robot media and WebRTC gateway"，无附加组（在此文件；附加组在 service 的 `SupplementaryGroups=`）。

## 关键摘要

- 非特权 `mediad` 用户是权限模型的基础：以 root 运行会掩盖设备节点权限问题。
- 实际权限全靠 `video`/`robot` 组成员资格，在 service 文件授予。
