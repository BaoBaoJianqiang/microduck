# robot.conf 文件解析

## 文件位置

`d:\microduck\updater\systemd\sysusers.d\robot.conf`

## 内容

```
g robot -
```

创建 `robot` 系统组。

## 定位

更新访问控制的第一层：updaterd socket 是 root:robot 模式 0660，所以组成员资格是让进程能与 daemon 对话的条件。在其中**不**赋予变更机器人的权利——那需要 updater.toml 的 allow_uids/allow_gids。

updaterd.service 若此组不存在将启动失败。

## 关键摘要

robot.conf 创建 robot 系统组，是 updaterd socket（root:robot 0660）访问控制的第一层（对话权）；变更权需另行在 updater.toml 授权。
