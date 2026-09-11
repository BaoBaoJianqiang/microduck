# `padd.conf` 解读

## 概述

systemd-sysusers 配置文件，创建 `padd` 系统用户和组。

安装到 `/usr/lib/sysusers.d/padd.conf`；`systemd-sysusers` 在启动时创建它们，或手动运行一次 `systemd-sysusers`。

---

## 核心行

```ini
u padd - "Robot gamepad intent client" - -
```

### 格式分解

`systemd-sysusers` 的行格式：`type name id gecos home shell`

| 字段 | 值 | 说明 |
|------|-----|------|
| `type` | `u` | 创建用户*和*组（同名） |
| `name` | `padd` | 用户名和组名 |
| `id` | `-` | UID/GID 自动分配 |
| `gecos` | `"Robot gamepad intent client"` | 描述（finger 信息） |
| `home` | `-` | 无 home 目录（`/nonexistent`） |
| `shell` | `-` | 无登录 shell（`/usr/sbin/nologin`） |

---

## 为什么 padd 必须作为自己的用户运行

> `padd` runs as its own unprivileged user for the reason the crate exists: it is an ordinary intent client with no privileged access to the robot, and it exercises the same API the phone app and the SDK will use. A `padd` running as root would still work and would quietly make that claim untrue.

**这不仅仅是卫生问题**：

- padd 以自己的非特权用户运行，理由正是 crate 存在的理由——它是一个普通意图客户端，对机器人没有特权访问，它练习手机 app 和 SDK 将使用的同一个 API
- 以 root 运行的 padd *仍然会工作*——但会悄悄让"普通客户端"这个说法不成立
- 这和 mediad.conf 的设计哲学一致：非 root 运行是权限验证机制

**区别于 mediad 的表述**：mediad 的理由是"root 会绕过 udev 规则，让权限问题不可见"；padd 的理由更直接——"root 仍然工作，但让'普通客户端'的说法不成立"。padd 存在的意义就是作为普通客户端练习意图 API，以 root 运行就失去了这个意义。

---

## 用户获得的权限全部来自组成员

> What this user gets comes entirely from group membership, granted in padd.service:

| 组 | 权限来源 | 访问什么 |
|----|----------|----------|
| `input` | `/dev/input/event*`（root:input 0660） | 读取手柄事件节点 |
| `robot` | robotd 的 0660 socket | 发送意图，和其他客户端相同方式 |

---

## 替代了什么：为什么不用操作员组

> This is also what replaced adding the *operator* to the `input` group. That worked, needed a log out and back in before it took effect, and was invisible when forgotten: `padd` started, reported nothing, and saw no pad at all. A system user with the group in its unit has neither problem.

### 旧方案：把操作员加到 input 组

以前的做法是把*操作员*（真人用户）加到 `input` 组，然后操作员 ssh 进来手动运行 padd。

问题：
1. **需要登出再登入**：组变更只在新会话中生效
2. **忘记了不可见**：padd 启动了，什么都没报告，看不到任何手柄

### 新方案：系统用户 + unit 文件中的组

- padd 是系统用户，组成员在 `padd.service` 的 `SupplementaryGroups` 中
- 不需要登出再登入——systemd 在启动单元时应用组成员
- 忘记加组会导致单元启动失败（`User=padd` 不存在），而非静默不工作

**核心改进**：从"忘记了不可见"变成"忘记了响亮失败"。

---

## 与其他文件的关系

- **`padd.service`**：使用 `User=padd` 和 `Group=padd`，通过 `SupplementaryGroups=input robot` 授予额外权限
- **`/usr/lib/sysusers.d/`**：安装位置
- **`mediad.conf`**：类似的 sysusers 配置，但 mediad 的组是 `video`/`render`/`robot`
- **`configd.service`**：padd 的 tap socket 通过 `robot` 组访问，和 robotd/configd/updaterd 的 socket 相同

---

## 与 mediad.conf 的对比

| 维度 | padd.conf | mediad.conf |
|------|-----------|-------------|
| 用户名 | padd | mediad |
| 描述 | Robot gamepad intent client | Robot media and WebRTC gateway |
| 补充组 | input, robot | video, render, robot |
| 非 root 的理由 | "普通客户端"的说法必须成立 | root 会绕过 udev 规则让权限问题不可见 |
| 替代的旧方案 | 把操作员加到 input 组（需要登出再登入，忘记了不可见） | 无（直接从一开始就设计为非 root） |
| 读取设备 | /dev/input/event*（手柄） | /dev/mpp_service, /dev/rga, /dev/videoN（摄像头/编码器） |

---

## 关键设计要点

### 1. 非 root 是 crate 存在的理由

padd 存在的意义是作为普通客户端练习意图 API。以 root 运行仍然工作，但让"普通客户端"这个说法不成立——这是一种"静默说谎"，比直接失败更危险。

### 2. 从"忘记了不可见"到"忘记了响亮失败"

旧方案（操作员加 input 组）的失败模式：
- padd 启动成功（进程能跑）
- 看不到手柄（input 组没生效）
- 没有错误消息（gilrs 只是报告没有手柄）
- 用户疑惑为什么手柄不工作

新方案（系统用户）的失败模式：
- 用户不存在 → `User=padd` 失败 → 单元启动失败 → journal 中有明确错误

### 3. sysusers.d 的幂等性

- 多次运行 `systemd-sysusers` 不产生重复用户
- 在启动时自动处理，不需要安装脚本手动调用
- 与包管理集成：包安装/卸载自动创建/移除用户
#（注：内容由AI生成）
