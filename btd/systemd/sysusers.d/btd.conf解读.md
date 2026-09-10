# `btd.conf` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | systemd-sysusers 配置文件 |
| 行数 | 23 行（含 21 行注释 + 1 行有效配置 + 1 空行） |
| 安装路径 | `/usr/lib/sysusers.d/btd.conf` |
| 作用 | 创建 `btd` 系统用户和组 |
| 关联文件 | `btd.service`（授予组成员资格）、`updater.toml`（`allow_users`）、`configd.service`（`--allow-user`） |

## 二、有效配置行

```
u btd - "Robot BLE transport adapter" - -
```

这是 systemd-sysusers 的 `u`（user）类型行，字段依次为：

| 字段位置 | 值 | 含义 |
|---|---|---|
| 类型 | `u` | 创建系统用户和同名组 |
| 名称 | `btd` | 用户名/组名 |
| UID | `-` | 自动分配（不固定 UID） |
| GECOS | `"Robot BLE transport adapter"` | 用户描述（机器人 BLE 传输适配器） |
| 主目录 | `-` | 不创建主目录（默认 `/`） |
| Shell | `-` | 不设置登录 shell（默认 `/sbin/nologin` 或空） |

### 安装与生效

- 安装到 `/usr/lib/sysusers.d/btd.conf`。
- `systemd-sysusers` 在启动时创建它们，或手动运行一次 `systemd-sysusers`。
- **`btd.service` 如果此用户不存在将无法启动。**

## 三、为什么 btd 以非特权用户运行

### 核心原因

`btd` 以自己的非特权用户而非 root 运行，因为它是**解析来自无线电范围内任何人的字节**的进程（`docs/architecture.md` §1.1）。

- 与 `configd` 形成对比：`configd` 只看到来自对等凭据本地 socket 的类型化 JSON，因此以 root 运行。
- 把解析器放在权限边界的安全侧，比加固分发器更重要。

### 访问完全来自组成员资格

btd 的访问完全来自组成员资格，在 `btd.service` 中授予：

| 组 | 授予的访问 |
|---|---|
| `robot` | 到达 updaterd、robotd 和 configd 的 `0660` socket |
| `bluetooth` | 应该通过 D-Bus 到达 `org.bluez`；需在镜像上验证 |

## 四、组成员资格故意不授予的东西

### 关键区分：能对话 ≠ 能改变

- 属于 `robot` 组让 btd *能与*那些守护进程对话，**不是*改变任何东西。
- 改变权限按服务、按名称授予：
  - `updater.toml` 中的 `allow_users = ["btd"]`
  - `configd.service` 中的 `--allow-user btd`

### 命名此服务是窄主张

- 命名 `btd` 服务是窄主张："btd 可以中继来自应用的请求"。
- 命名 `robot` 组反而会把"可以读状态"和"可以替换固件"合并成一个权限。
- 两个服务都有测试精确拒绝这种合并（§2.2，`btd/src/route.rs`）。

## 五、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `btd.service` | 授予 `robot` 和 `bluetooth` 组成员资格；如果 `btd` 用户不存在则启动失败 |
| `updater.toml` `allow_users = ["btd"]` | 窄权限：btd 可中继 App 的更新请求，而非整个 `robot` 组 |
| `configd.service` `--allow-user btd` | configd 侧的按名称授权 |
| `btd/src/route.rs` | 方法白名单，穷举 match 防止新方法静默到达无线电；有测试验证拒绝越权 |
| `architecture.md` §1.1 | 定义 btd 作为解析不可信字节的进程的非特权定位 |
| `architecture.md` §2.2 | 定义按名称授权而非按组授权的权限模型 |

## 六、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **最小权限** | btd 非特权运行，访问完全来自组成员资格，不直接以 root 运行 |
| **按名称授权而非按组** | `allow_users = ["btd"]` / `--allow-user btd`，窄主张，避免组权限过度授权 |
| **能对话 ≠ 能改变** | 组成员资格只授予 socket 访问权，改变权限需服务单独按名称授予 |
| **防御纵深** | 权限边界多层：非特权用户 + 组成员资格 + 按名称服务授权 + 方法白名单（route.rs） |
| **可验证** | 两个服务都有测试精确拒绝"按组授权会导致的越权" |
#（注：内容由AI生成）
