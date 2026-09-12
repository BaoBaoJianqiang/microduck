# `btd.conf` 文件解析 —— 创建 btd 系统用户与组

## 1. 文件定位

- 路径：`systemd/sysusers.d/btd.conf`
- 安装位置：`/usr/lib/sysusers.d/btd.conf`
- 角色：systemd-sysusers 的声明式配置，用于创建 `btd` 系统用户与同名组。
- 生效方式：`systemd-sysusers` 在开机时创建，或手动执行一次 `systemd-sysusers`。

## 2. 唯一的有效配置行

```sysusers
u btd - "Robot BLE transport adapter" - -
```

字段含义（sysusers.d 格式 `类型 名称 ID 昵称 主目录  Shell`）：

- `u`：创建一个用户（同时自动建立同名组）；
- `btd`：用户名；
- 第一个 `-`：不指定固定数值 ID，由系统自动分配；
- `"Robot BLE transport adapter"`：用户描述/昵称（GECOS）；
- 后两个 `-`：不指定主目录与登录 shell——这是一个不登录的服务账户。

## 3. 注释中的设计说明

### 3.1 为什么用专用非特权用户

btd 以自己的非特权用户而非 root 运行，因为它是解析"无线电范围内任何人发来的字节"的进程（docs/architecture.md §1.1）。它的访问权完全来自组成员关系（在 `btd.service` 中授予）：

- **`robot` 组**：能到达 updaterd、robotd、configd 的 `0660` socket；
- **`bluetooth` 组**：应能经 D-Bus 到达 org.bluez（需要在镜像上验证）。

### 3.2 组成员关系"不"授予什么（最小权限）

注释特别澄清：在 `robot` 组中只让 btd 能与这些守护进程**交谈**，并不授予变更任何东西的权力。变更权限是"逐服务、按名字"授予的：

- updater：`updater.toml` 中的 `allow_users = ["btd"]`；
- configd：`btd.service` 里的 `--allow-user btd`。

按服务名授权是一个很窄的声明，含义仅为"btd 可以中转来自 App 的请求"。若改为授予整个 `robot` 组，会把"可以读状态"与"可以替换固件"压缩成同一项权限；两个服务都有测试明确拒绝那种情况（§2.2，见 `btd/src/route.rs`）。

### 3.3 与服务单元的耦合

注释末尾提醒：如果该用户不存在，`btd.service` 将无法启动。因此本文件必须与单元文件一起安装。

## 4. 本文件要点小结

1. 一行 sysusers 声明创建无登录、无主目录、自动分配 ID 的系统用户 `btd`（及同名组）；
2. 专用账户是"解析不可信无线电字节的进程不持权"这一安全边界的落地；
3. 访问能力仅来自附加组（robot 用于 unix socket，bluetooth 用于 D-Bus）；
4. 组身份只给"连通性"，变更权限按服务名单独授予，避免把只读与刷固件并为一项；
5. 必须与 `btd.service` 同时部署，否则单元启动失败。
