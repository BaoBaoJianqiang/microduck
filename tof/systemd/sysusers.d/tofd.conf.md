# tofd.conf 文件解析

## 文件位置

`d:\microduck\tof\systemd\sysusers.d\tofd.conf`

## 内容

```
u tofd - "Robot ToF sensor daemon" - -
```

sysusers.d 语法 `u` 创建系统用户；用户名 `tofd`；无显式 uid（自动分配）；描述 "Robot ToF sensor daemon"；无主目录、无 shell。

## 设计说明

`tofd` 非特权运行的原因与 `padd` 相同：它只需要操作者账户没有的一样东西——I²C 总线——而这通过组身份授予（在 tofd.service 的 `SupplementaryGroups` 中），而非以 root 运行：

- `i2c` 组：`/dev/i2c-*` 为 root:i2c 模式 0660
- `robot` 组：使其能把自己的 socket 交给可看机器人的组

若该用户不存在，tofd.service 将启动失败。

## 安装

安装到 `/usr/lib/sysusers.d/tofd.conf`；systemd-sysusers 在启动时创建，或手动运行 `systemd-sysusers` 一次。

## 关键摘要

一行配置创建非特权 `tofd` 系统用户，权限完全来自 i2c（读总线）与 robot（发 socket）两个补充组，与 padd 的权限模型一致。
