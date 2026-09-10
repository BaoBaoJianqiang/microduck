# `/etc/robot/updater.toml` 配置解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件路径 | `/etc/robot/updater.toml` |
| 文件类型 | TOML 配置文件（生产环境） |
| 安装方式 | 由 `scripts/install.sh` 写入客户端机器人 |
| 作用对象 | 机器人固件/组件更新守护进程（updaterd） |
| 配套参考 | `updater/updater.example.toml`（带完整注释的参考模板，覆盖此处未启用的全部选项） |

**解读要点**：本文件与示例模板的差异即是重点——示例展示"可以怎么做"，本文件展示"出厂机器人实际怎么做"。文件中每个值都代表一个**决策**或**事实**，注释说明了"为什么这么选"。

---

## 二、全局配置

### 1. 信任锚：`trusted_keys_dir = "/etc/robot/trusted_keys"`

- 安装时一次性写入**全部 3 个发布公钥**，尽管当前只有 `release-1` 在签名。
- **设计意图**：机器人只能校验"烧录进自身的密钥集合"。出厂时预埋备用公钥，是让**密钥轮换**无需物理重刷设备的唯一机会（见 `docs/project/ci-setup.md`）。

### 2. 硬件与状态路径

| 配置项 | 值 | 说明 |
|---|---|---|
| `hw_rev` | `1` | 硬件版本号 |
| `state_dir` | `/var/lib/robot/updater` | 更新状态目录，位于所有 `install_dir` **之外**——这样"记录更新历史"这件事能在它正在记录的 swap/回滚操作中存活下来（§5.7） |
| `robot_socket` | `/run/robotd.sock` | 机器人守护进程套接字路径 |

### 3. 安全开关（双 false）

```toml
allow_dev_keys        = false
allow_fault_injection = false
```

- `allow_dev_keys = false`：禁止安装任何团队成员随意构建的产物。
- `allow_fault_injection = false`：禁止让更新器被指令"故意失败"。
- **设计意图**：开发者板卡可在本地覆盖这两项，但**更新操作不会触碰 `/etc`**，因此任何线上更新都无法把它们改回 true——出厂安全性不可被远程放宽。

### 4. 检查节奏：`check_interval = "6h"`

- 每 6 小时轮询一次发布源。
- **设计意图**：让 `min_supported`（最低支持版本）真正有约束力。若从不轮询，机器人只有等用户打开 App 才会得知"存在一个版本下限"，一旦发布过坏版本，主人不在身边的机器人将**无法被修复**（§8.1）。

### 5. 自动应用策略：`auto_apply = "mandatory"`

取值含义：`off`（不自动装）/ `mandatory`（仅强制版）/ `all`（全部自动装）。

- 本文件选 **`mandatory`**：仅当发布清单中的 `min_supported` 高于当前运行版本时自动安装——否则整个机群会困在一个已被撤回的版本上。
- **普通版本仍等待客户端触发**：机器人何时重启由主人决定，App 驱动的更新流程就是为把决定权交给主人而存在的。
- **`all` 不用于客户端机器人**：它只属于 canary / bench 机器人（§16.2 Tier 2），用于跟踪频道并安装每个候选版本；在此处设置会把所有客户端机器人拖入无人值守重启。
- **无论哪种策略**，无人值守应用与普通应用走同一套流程：同样的 preflight（行走中/直播中的机器人会拒绝并在下个周期重试），同样的健康门（启动不起来的版本会被回滚）。

### 6. 权限模型：`allow_users = ["btd", "mediad"]`

**只读调用（status / log / listInstalled / check / subscribe）不受此限制**——能连上套接字（mode 0660）即已要求所属组，且支持人员必须能检查无权修改的机器人。

| 授权用户 | 授权范围（刻意收窄） | 背景 |
|---|---|---|
| `btd`（蓝牙传输） | 仅可**转发**来自 App 的更新请求 | M6 的核心交付是"主人用手机更新机器人"；缺此行 `update.apply` 经蓝牙返回 `PERMISSION_DENIED`。注意措辞之窄："可转发请求"，而非"机器人组内任何人可替换固件" |
| `mediad`（媒体守护） | 仅其路由表允许的变更调用：`account.login` / `account.logout` / `policy.install` / `policy.fetch` | 控制台页面需要登录机器人与安装策略；此前 WebRTC 路由缺此行导致 `PERMISSION_DENIED`（Hub 浏览器"安装按钮"形同虚设）。该边界由测试 `mediad::route` 的 `only_these_mutating_calls_are_reachable_over_webrtc` 强制——新增变更方法必须刻意修改该列表。**`mediad` 无更新权限**：`update.apply` 在其侧被拒绝 |

**三条硬性约束（均有测试守护）**：

1. **按名字而非 uid**：systemd-sysusers 动态分配 uid，写死数字会在不同板卡上对错号；`allow_uids`/`allow_gids` 仅为 bench 覆盖保留，且测试断言出厂配置不使用它们。
2. **绝不允许 `allow_groups = ["robot"]`**：`robot` 组成员资格只应换来"能与 updaterd 对话"，再授予变更权会 collapse 两层安全模型——任何能读状态的进程都能换固件。
3. 相关测试同时存在，保证上述约束不会被无意破坏。

---

## 三、daemon 组件配置

### 1. 安装与保留策略 `[component.daemon]`

```toml
install_dir   = "/opt/robot/daemon"
keep_previous = 1
```

- `keep_previous = 1`：保留上一版本，支撑回滚。
- **`golden` 故意未设置**（注释掉的 `# golden = "1.0.0"`）：
  - `golden` 命名一个永不被清理的"已知良好版本"，`robotctl update reset-to-golden` 是**永不砖机链路**的最后一环（§8.2）。
  - 若命名一个机器人从未安装过的版本，该命令会在最需要它的时刻失败——这比它如实报告"未配置 golden"更糟。
  - **约定**：与 1.0.0 发布的同一变更中设置它。

### 2. 发布源 `[component.daemon.source]`

```toml
type           = "github_releases"
repo           = "ORG/duck-daemon"
tag_prefix     = "daemon-v"
manifest_asset = "manifest.json"
```

- 稳定频道只认 `daemon-v*` 标签。
- **staging 发布 `daemon-staging-v*` 并标记为 prerelease**，被双重排除——客户端机器人不可能漂移到候选构建。
- 两个注释掉的默认值（无需配置即生效）：
  - `ref_tag_prefix = "daemon-dev-"`：按分支的 dev 构建标签。dev 标签是**可移动的**（指向该分支最近一次构建），发布标签**不可变**，`latest` 永不考虑 dev 标签；且安装 dev 构建还需 `allow_dev_keys` + dev 公钥，客户端机器人即使解析到标签也无法安装。
  - `staging_tag_prefix = "daemon-staging-v"`：`--staging` 的候选标签。能触达候选仍需"有 root 权限的人手动敲 `--staging`"，且 prerelease 会被 `latest` 跳过——配置本身不会让机器人跟踪候选。

### 3. 应用后动作 `[component.daemon.on_apply]`

```toml
action = "restart"
units  = ["robotd", "configd"]
```

- **`updaterd` 与 `btd` 刻意缺席，且理由相同：不能重启正在执行/承载该操作的服务。**
  - `updaterd`：它就是执行更新的进程，重启自身等于在更新中途杀死执行者（updater-design.md §4.1）。
  - `btd`：它可能是本次更新请求的**传输通道**。重启会断开承载 `update.subscribe` 的 BLE 连接——发起更新的手机丢失进度流，永远不知道结果，整个 App 驱动流程即告失败。`updater/src/config.rs` 有测试拒绝 btd 出现在此列表。
  - 两者不等待重启：更新应答后 5 秒被重启，下次 updaterd 启动会检查该步骤是否完成并补做（docs/design/restart-order.md §1、§5）。
- **`configd` 反而被重启**：它不持有长会话状态，客户端观察的进度流来自 updaterd 而非它；配置变更迟迟不生效是另一种意外。
- **该列表是"附加性"而非"权威性"**：重启集合实际由发布物所带的 unit 推导，这里只需列出发布物**不**带的 unit。两条都是冗余的，保留只因"显式声明期望比空列表更易读"。
- **历史教训**：它曾是权威列表，导致 `install-path-gap.md` §4 记载的 bug——在 `configd` 存在前配好的板卡永远保留 `units = ["robotd"]`，每次发布都换掉 configd 二进制却让旧进程继续运行。`install.sh` 保留本文件，故该 bug 无法靠更新自愈。

### 4. 健康门 `[component.daemon.health]`

```toml
probe   = "socket"
timeout = "30s"
```

- 这是**自动回滚成真**的门：应用新版本后探测失败/超时即回滚。
- `30s` 仍是 M4 计划的占位值，将替换为"实测启动时间 + 余量"；过短会误回滚一个只是启动慢的健康版本。
- **只对 `unhealthy` 回滚，不对 `degraded` 回滚**：看不到舵机的机器人报 degraded 并通过——它交换前也这么说，回滚修不好它，反而会把每个已发布版本都回滚到 bench 板；而破坏控制回路的版本照常回滚。`robotctl health` 会打印当前处于哪种状态。

---

## 四、models 组件：刻意的缺席

- 文件里**没有** `[component.models.*]` 配置块，这不是遗漏，而是有意的设计决策。
- **原因**：示例模板配置的 `model-walk` / `model-jump` 指向尚不存在的 HF 仓库。一个 source 404 的组件会让**每次 `check`（含上述 6 小时周期检查）都报失败**——这会把"读机器人状态的人"训练成习惯性忽略失败。
- **约定**：每个模型组件在"首次发布其 bundle 的那次变更"中才落地配置，在此之前不存在于配置中。

---

## 五、核心设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **安全分层** | 套接字组权限（对话）与 `allow_users` 变更权限（改固件）严格分离；`btd`/`mediad` 权限刻意收窄并以测试锁定边界 |
| **出厂安全不可逆** | 更新不触碰 `/etc`，`allow_dev_keys`/`allow_fault_injection` 只能本地覆盖 |
| **永不砖机** | `keep_previous` + 健康门（unhealthy 才回滚）+ 未来 `golden`（1.0.0 随发布启用）三重保障 |
| **失败可见性优先** | 模型组件不存在的配置宁缺毋滥，避免"永报失败"训练操作员忽视告警 |
| **密钥轮换可行** | 出厂预埋全部 3 个公钥，避免物理重刷 |
| **执行者自保** | 重启集合排除 updaterd 与 btd，由测试和延迟重启机制兜底 |

---

## 六、配置速查表

| 配置项 | 值 | 一句话含义 |
|---|---|---|
| `trusted_keys_dir` | `/etc/robot/trusted_keys` | 信任锚，预埋 3 个公钥以支持未来轮换 |
| `hw_rev` | `1` | 硬件版本 |
| `state_dir` | `/var/lib/robot/updater` | 状态目录（位于 install_dir 之外，回滚后仍存活） |
| `robot_socket` | `/run/robotd.sock` | 守护进程套接字 |
| `allow_dev_keys` | `false` | 禁止开发密钥 |
| `allow_fault_injection` | `false` | 禁止故障注入 |
| `check_interval` | `"6h"` | 每 6 小时轮询发布源 |
| `auto_apply` | `"mandatory"` | 仅自动安装 min_supported 强制版本 |
| `allow_users` | `["btd", "mediad"]` | 允许 btd 转发更新请求、mediad 执行其路由表内变更 |
| `component.daemon.install_dir` | `/opt/robot/daemon` | 安装目录 |
| `component.daemon.keep_previous` | `1` | 保留上一版本 |
| `component.daemon.golden` | （未设置） | 1.0.0 发布时启用 |
| `component.daemon.source.type` | `github_releases` | 发布源类型 |
| `component.daemon.source.repo` | `ORG/duck-daemon` | 发布仓库 |
| `component.daemon.source.tag_prefix` | `daemon-v` | 稳定版标签前缀 |
| `component.daemon.source.manifest_asset` | `manifest.json` | 清单文件名 |
| `component.daemon.on_apply.action` | `restart` | 应用后重启 |
| `component.daemon.on_apply.units` | `["robotd", "configd"]` | 附加重启单元（排除 updaterd/btd） |
| `component.daemon.health.probe` | `socket` | 健康探测方式 |
| `component.daemon.health.timeout` | `"30s"` | 健康门超时（M4 占位值） |
| `component.models.*` | （不存在） | 刻意缺席，随首次发布落地 |
