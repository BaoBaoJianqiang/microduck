# padd.conf 文件解析

**文件位置**：`d:\microduck\padd\systemd\sysusers.d\padd.conf`

## 核心设计决策

创建 `padd` 系统用户与组。`padd` 以自己的非特权用户运行——这是 crate 存在的理由：它是普通意图客户端，对机器人无特权访问，锻炼与手机 app 和 SDK 相同的 API。以 root 运行的 padd 仍能工作，但会悄悄让这一声称为假。

## 配置

```
u padd - "Robot gamepad intent client" - -
```

创建系统用户 `padd`，无密码，描述为"Robot gamepad intent client"，无主目录、无 shell。

## 权限来源

全部来自组成员（在 padd.service 授予）：
- `input`——经 `/dev/input/event*` 读手柄（root:input 0660）。
- `robot`——达 robotd 的 0660 socket，与其他客户端一样。

这也替代了把*操作者*加入 input 组的旧做法——那需要登出再登入才生效，且忘记时不可见：padd 启动、不报错、完全看不到手柄。系统用户在 unit 里带组则无此两问题。

## 关键摘要

一行 sysusers 声明创建非特权 padd 用户。其权限完全来自 input（读设备）与 robot（达 robotd socket）两个补充组。padd.service 若用户不存在会启动失败。
