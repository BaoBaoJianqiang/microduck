# 更新系统设计

Microduck 机器人的 OTA 更新系统。

本页面拥有更新引擎、签名、多组件、原子 symlink swap、健康门、自动回滚、golden release、boot counter、min_supported、known_bad、post-install hooks、恢复、测试和发布通道。[`restart-order.md`](restart-order.md) 拥有什么重启以及何时重启；[`boot-recovery-net.md`](boot-recovery-net.md) 拥有当启动的版本无法启动守护进程时会发生什么。

## 1. 概述

### 应用级而非 A/B 镜像

更新系统是应用级的，而不是 A/B 分区镜像。这意味着：

- 更新只更新机器人应用（守护进程、模型、配置），而不是整个操作系统
- 内核和系统库通过 `unattended-upgrades`（仅限安全口袋）或在服务时重新刷新来更新
- 回退是 symlink swap，而不是分区切换

**为什么不是 A/B：** A/B 分区需要两倍的存储空间（两个根分区），并且更新整个操作系统（包括很少更改的部分）。对于一个存储有限（Radxa Zero 3W 上的 eMMC）且应用比 OS 更改频繁得多的机器人来说，应用级更新是正确的权衡。

### 机器人从 CDN pull

更新是 pull 模式，而不是 push 模式：

1. 机器人（通过 `updaterd`）定期检查更新
2. 如果有更新，机器人从 CDN（GitHub Releases 或 Hugging Face Hub）下载 artifact
3. 机器人验证签名和 hash
4. 机器人应用更新

**Payload 永不经过 BLE。** BLE 仅用于触发更新（手机发送"检查更新"命令）。实际的 artifact 通过 wifi/以太网从 CDN 下载。这是因为 BLE 带宽太低（约 100KB/s），无法合理地传输 MB 级的 artifact。

### GitHub Releases + HF Hub

两种 artifact 来源：

| 来源 | 用途 | 示例 |
|---|---|---|
| GitHub Releases | 守护进程二进制文件 | `ORG/robot-daemon`, tag `daemon-v1.0.0` |
| Hugging Face Hub | ML 模型 | `ORG/gait-walk`, revision `main` |

守护进程二进制文件很小（几 MB），并且版本化严格。模型很大（几十到几百 MB），并且版本化较松（`main` 分支）。两者使用相同的更新引擎，但有不同的 source 配置。

## 2. 签名

### minisign

所有 artifact 都使用 minisign 签名。minisign 是一个简单的签名工具，使用 Ed25519 签名。

**为什么是 minisign 而非 GPG：**
- 更简单：minisign 只有几个命令，而 GPG 有一个复杂的密钥管理系统
- 更小：minisign 签名是 112 字节，而 GPG 签名是几百字节
- 无密钥服务器：minisign 公钥直接分发（在机器人文件系统上），不需要密钥服务器
- Rust 库：`minisign-verify` crate 是纯 Rust，没有外部依赖

### 签名验证流程

```
1. 下载 artifact（.tar.zst）
2. 下载签名（.minisig）
3. 下载 manifest（manifest.json）
4. 下载 manifest 签名（manifest.json.minisig）
5. 使用 trusted key 验证 manifest 签名
6. 从 manifest 读取 artifact 的 sha256 hash
7. 计算下载的 artifact 的 sha256 hash
8. 比较 hash
9. 使用 trusted key 验证 artifact 签名（可选，manifest 已经足够）
```

manifest 是一个 JSON 文件，列出了 release 中的所有 artifact 及其 hash。签名 manifest 意味着攻击者不能替换 artifact 而不被检测到（因为 hash 不匹配），也不能替换 manifest 而不被检测到（因为签名不匹配）。

### trusted_keys_dir

trusted keys 存储在 `/etc/robot/trusted_keys/` 中。这是一个*密钥集合*，而不是单个密钥。这允许：

- 密钥轮换：旧密钥可以在新密钥添加后被移除
- dev key：开发密钥可以在受信任的集合中，但被门控在 dev build 后面
- 多签名者：不同的组件可以由不同的密钥签名

`updaterd` 在验证时尝试集合中的每个密钥。如果任何一个密钥验证成功，artifact 被接受。

## 3. 多组件配置驱动引擎

### 组件

更新引擎是多组件的。每个组件是一个独立的可更新单元，有自己的：

- install_dir
- source（GitHub Releases 或 HF Hub）
- on_apply 动作（restart 或 reload）
- health 探针
- golden 版本
- keep_previous 数量

示例组件：

| 组件 | install_dir | source | on_apply |
|---|---|---|---|
| daemon | `/opt/robot/daemon` | GitHub Releases | restart |
| model-walk | `/opt/robot/model/walk` | HF Hub | reload (SIGHUP) |

### 为什么是多组件

守护进程和模型有不同的发布节奏：

- 守护进程：频繁更新（错误修复、新功能），小 artifact
- 模型：不频繁更新（训练新模型），大 artifact

如果它们在同一个组件中，每次模型更新都会强制重新下载守护进程（反之亦然）。多组件允许它们独立更新。

模型使用 `reload`（SIGHUP → 重新 mmap 权重）而不是 `restart`，因此模型交换时电机控制永不断开。这是两个通道保持独立的核心原因。

### 配置文件

组件在 `/etc/robot/updater.toml` 中配置：

```toml
trusted_keys_dir = "/etc/robot/trusted_keys"
hw_rev           = 1
state_dir        = "/var/lib/robot/updater"

[component.daemon]
install_dir   = "/opt/robot/daemon"
keep_previous = 1
golden        = "1.0.0"

[component.daemon.source]
type       = "github_releases"
repo       = "ORG/robot-daemon"
tag_prefix = "daemon-v"

[component.daemon.on_apply]
action = "restart"
units  = ["robotd", "mediad"]

[component.daemon.health]
probe   = "socket"
path    = "/run/robotd.sock"
timeout = "30s"
```

配置在加载时验证，`deny_unknown_fields` 开启，因此拼写错误会大声失败，而不是静默禁用它想要更改的设置。

值得注意的拒绝：
- `state_dir` 在 `install_dir` 内部（swap 会破坏更新日志）
- 两个组件共享一个 `install_dir`
- 相对路径
- `keep_previous = 0` 且没有 golden（没有回退目标）

## 4. 原子 symlink swap + health gate + 自动回滚

### 目录布局

```
/opt/robot/daemon/
├── current → releases/1.1.0/
├── golden → releases/1.0.0/
└── releases/
    ├── 1.0.0/
    ├── 1.1.0/
    └── 1.2.0/
```

- `releases/<version>/`：解压的 release 目录
- `current`：指向当前运行版本的符号链接
- `golden`：指向具有持续保证的版本的符号链接

所有守护进程通过 `current` exec。systemd 单元文件中的 `ExecStart=/opt/robot/daemon/current/bin/robotd`。

### swap 是原子的

更新通过 `rename(2)` 将临时符号链接重命名到 `current` 上。`rename(2)` 是原子的——要么 swap 完全发生，要么完全不发生。没有"半交换"状态。

```
1. 创建临时符号链接：current.tmp → releases/1.2.0/
2. rename("current.tmp", "current")  ← 原子
3. 旧的 current 被替换
```

这与 `ln -sfn` 形成对比，后者会取消链接并重新创建，留下一个守护进程无法 exec 的窗口。

### health gate

swap 之后，`updaterd` 等待健康探针通过：

```
1. swap current → 新版本
2. 重启 units_to_restart 中的单元
3. 等待健康探针通过（最长 30 秒）
4. 如果健康通过 → 提交更新
5. 如果健康失败 → 回滚到 previous
```

健康探针是 `robotd` 的 unix socket。`updaterd` 连接到 socket 并发送 `system.health` 请求。如果 `robotd` 在超时内回复 `healthy`，更新被提交。

### 自动回滚

如果健康探针失败，`updaterd` 自动回滚：

```
1. swap current → previous 版本
2. 重启单元
3. 等待健康探针通过
4. 记录回滚
5. 将失败的版本标记为 known_bad
```

回滚是自动的——不需要操作员干预。机器人在旧版本上恢复健康，更新日志记录了发生了什么。

## 5. golden release

### 什么是 golden

golden 是一个具有持续保证的版本——它被知道可以工作，并且永远不会被修剪。它是回退链的最后一道防线：

```
current → previous → golden
```

如果 current 失败，回滚到 previous。如果 previous 也失败（或缺失），回滚到 golden。

### 为什么需要 golden

previous 版本可能不工作——它可能是一个被回滚的坏版本，或者它可能已经被修剪。golden 是唯一保证工作的版本，因为它是在发布时手动验证的。

golden 在配置中指定（`golden = "1.0.0"`），并且 `Engine::refresh_golden_links` 在每次 `updaterd` 启动时将其发布为 `current` 旁边的 `golden` 符号链接。

### golden 永不被修剪

`prune` 删除旧版本，但永远不会删除 golden。这意味着 golden 始终在磁盘上可用，即使它很旧。

## 6. boot counter

### 什么是 boot counter

boot counter 是一个机制，用于捕获健康门无法捕获的故障：一个启动并运行但从不报告健康的版本。

每次 `updaterd` 启动时，它都会增加当前版本的 boot counter。如果 boot counter 达到阈值（默认 3），版本被认为是坏的，`updaterd` 自动回滚。

### 为什么需要 boot counter

健康门在更新后立即运行。但如果一个版本在更新后通过健康门，然后在稍后的启动中失败（例如，因为它只在冷启动时失败，或者因为硬件状态不同），健康门不会捕获它。

boot counter 捕获这种情况：如果一个版本无法连续启动 3 次，它就是坏的，回滚。

### 与启动恢复网络的关系

boot counter 在 `updaterd` 内部运行。但如果 `updaterd` 本身无法启动，boot counter 就无法增加。这就是启动恢复网络（`boot-recovery-net.md`）存在的原因——它在 `updaterd` 外部运行，捕获 `updaterd` 无法启动的情况。

## 7. min_supported 最低版本地板

### 什么是 min_supported

`min_supported` 是一个版本地板——机器人不能安装低于此版本的版本，即使它被签名并有效。

`min_supported` 在 manifest 中指定：

```json
{
  "version": "1.2.0",
  "min_supported": "1.0.0",
  "artifacts": [...]
}
```

### 为什么需要 min_supported

有时，新版本需要一个最低版本的配置或状态。例如，如果 1.2.0 引入了一个新的配置格式，它可能需要 1.0.0 或更高版本（因为 0.9.0 有旧的配置格式，迁移脚本可能不存在）。

`min_supported` 防止机器人安装一个与当前状态不兼容的版本。如果机器人运行 0.9.0 并且 manifest 说 `min_supported = "1.0.0"`，更新被拒绝。

### 如何升级地板

如果机器人运行 0.9.0 并且需要升级到 1.2.0，它必须先升级到 1.0.0（或 1.1.0），然后再升级到 1.2.0。这是一个两步升级，但它确保迁移脚本有机会运行。

## 8. known_bad 守卫

### 什么是 known_bad

`known_bad` 是一个版本集合，已知是坏的。`updaterd` 永远不会安装或回滚到 known_bad 版本。

### 为什么需要 known_bad

如果一个版本失败并被自动回滚，它应该被标记为 known_bad。否则，`updaterd` 可能会在下次检查更新时重新安装同一个坏版本，导致无限循环：

```
安装 1.2.0 → 健康失败 → 回滚到 1.1.0 → 检查更新 → 安装 1.2.0 → ...
```

`known_bad` 打破了这个循环：1.2.0 被标记为 known_bad，`updaterd` 不会重新安装它，直到操作员明确清除标记（或发布 1.2.1）。

### known_bad 是持久的

`known_bad` 存储在状态目录中，跨重启持久化。这意味着即使机器人重启，它也不会重新安装一个已知坏的版本。

## 9. 签名防降级但不防冻结

### 降级 — 已修复

`updaterd` 在目标是"最新"时拒绝比已安装版本更旧的候选版本：

- `Target::Latest` → 拒绝，错误 `WOULD_DOWNGRADE`
- `Target::Exact` → 允许（这是定向回退的工作方式）
- `rollback` 和 `reset-to-golden` 按设计向后移动，使用已安装的版本，因此它们根本不查询 manifest

不对称性是要点：守卫阻止*攻击者*可以导致的事情，同时留下*操作员*可以选择的事情。

### 冻结 — 开放

目前没有任何东西区分"没有更新"和"有人阻止我看到更新"。一个被无限期固定在易受攻击版本的机器人是现实的危害——它已经在运行相同的版本，因此没有完整性检查触发。

选项（最便宜的优先）：

1. **签名 manifest 过期**：在 manifest 中添加 `not_valid_after`；拒绝比这更旧的 manifest。代价：发布变得有时间限制——CI 必须按计划重新签名 `stable` manifest，即使没有发布更改。
2. **单调 manifest 计数器**：一个只增加的 `sequence`；机器人记录看到的最高值并拒绝任何更低的值。检测元数据的*回滚*，但不检测冻结（重放的当前 manifest 具有预期的 sequence）。
3. **报告陈旧而非强制执行**：记录"上次成功的 manifest 获取"并在 `status`/应用中显示。在密码学上什么都检测不到，但使被冻结的机器人对其所有者和支持可见。
4. **Phone-home**：报告的机器人让*我们*看到哪些机器人停止检查——唯一集中检测冻结的选项。需要后端。

**建议：现在用 (3)，当有发布节奏可以挂接时用 (1)。** 陈旧报告几乎是免费的，没有自己的失败模式，并将静默攻击转换为可见攻击。过期是真正的防御，但其运营成本——一个重新签名计划，如果错过，会警告整个机群——在发布管道成为常规之前不值得支付。

**v1 明确接受：** 网络敌对的机器人可以被阻止更新。它不能被*降级*、安装我们没有签名的 artifact、或安装健康门失败的版本。这些是我们实际依赖的属性。

## 10. Post-install hooks

### 与 dpkg postinst 相同的想法

Post-install hooks 与 dpkg 的 `postinst` 相同的想法，但 hook 发布在*（签名的）tarball 内部*，因此没有未签名的代码曾经运行。Rust 二进制文件或 shell 脚本——引擎只是 `exec` 它。

### 契约

- 在 symlink swap *之后*、`apply` *之前*运行。（`pre_install` hook，如果存在，在 swap 之前运行。）
- pre-install hook 比 post-install hook 有更长的上限——十分钟对两分钟。
- 非零退出 ⇒ 更新失败 ⇒ 回滚。
- 提供的环境变量（缺失的值被**省略**，而不是设为空，因此 hook 可以区分"首次安装"和"未知"）：
  - `UPDATE_COMPONENT` / `UPDATE_CHANNEL` — 组件名称
  - `UPDATE_NEW_VERSION`，以及 `UPDATE_OLD_VERSION`（当有先前版本时）
  - `UPDATE_RELEASE_DIR` — 正在安装的 release
  - `UPDATE_INSTALL_DIR` — 组件根目录
  - `UPDATE_NEW_SCHEMA_VERSION`，以及 `UPDATE_OLD_SCHEMA_VERSION`（当已知时）
- Hooks 使用**清除的环境**加上固定的 `PATH` 运行，因此行为不依赖于 systemd 如何调用 `updaterd`。
- 输出被捕获和截断，并且无论成功或失败都会到达更新日志。

### 如果全新安装做了，hook 也做

**一个 release 直到它需要的一切都在板上才被安装。** 将文件发布到 release 目录不是安装它；将脚本发布到 release 目录不是运行它。Hook 是唯一在每次更新时在每个板上运行的东西，因此"这个板在这个 release 工作之前必须有 X"属于这里——不属于在 X 存在之前运行过一次的 provisioning 脚本，也不属于人类的记忆。

这已经被搞错了四次，以四种不同的形式：

1. **Units**：一个添加了守护进程的 release 将其 `.service` 放在 artifact 内部，但没有放在 systemd 查找的任何地方。`btd` 在 release 完整且正确的板上失败，错误 `203/EXEC`。
2. **GStreamer 栈和 3A 引擎**：Provisioning 安装了它们，因此在它们存在之前被 provisioned 的板没有它们。
3. **NPU**：添加 duck detector 的分支编写了 `setup-npu.sh`，将其打包到 release 中——但从未调用它。
4. **登录 shell**：`install.sh` 添加了一个 `/etc/profile.d` 片段，将机器人名称放在主机名旁边。没有其他东西安装它，`install.sh` 不打包到 release 中，因此没有 hook*可以*安装它。

每次的形式都相同：工作*完成了*，但使工作到达板的东西被遗漏了。它在审查中是不可见的，因为添加脚本的 diff 看起来是完整的。

**推论：**

- **`install.sh` 对板做的任何事情，hook 也做。**
- **Hook 步骤必须是幂等的，并且在没有事情可做时必须是廉价的。**
- **Hook 运行的脚本必须被打包。**
- **脚本需要的东西必须随它一起携带。**
- **可选硬件永远不是致命的。** Hook 的非零退出会使更新失败并回滚，因此 hook 只能为意味着 release 不可安装的事情失败。

## 11. 可重用、配置驱动的引擎

引擎在机器人之间是相同的；每个机器人发布一个声明其组件的配置。适应新机器人 = 新配置 + 新签名密钥 +（可能）新的健康探针。

跨机器人共享：引擎、minisign 管道、phone/BLE 触发协议、回滚逻辑。每个机器人：配置文件、签名密钥、健康探针、hooks。它是一个小二进制文件 + 一个 schema，而不是一个框架。

## 12. OS 更新（故意不在 OTA 范围内）

跳过 A/B 有一个真正的约束：我们不对内核/系统库进行 OTA。缓解措施：

- 启用 `unattended-upgrades`，范围仅限于 Debian **security** pocket，用于 CVE 补丁。设置后忘记。
- 任何更大的事情（内核升级、守护进程需要的新系统库）= **在服务时重新刷新**。
- 最小化对基础 OS 的运行时依赖：近似静态链接，在 tarball 内捆绑我们能捆绑的，因此库存 OS 永远不会阻止守护进程更新。

### 外设 / MCU 固件 — 不在范围内

子板和舵机固件在**生产时**刷新，而不是在现场。不是 OTA 组件；没有为它做准备。

## 13. 考虑的替代方案：apt / dpkg（自有签名仓库）

仍然值得保留在桌面上，特别是对于 Debian 上的**守护进程**通道：

**优点**
- 签名（GPG）、原子安装、版本控制和**维护者脚本**（`preinst`/`postinst`/`prerm`/`postrm`）——我们想要的 post-install hook——全部免费且经过实战测试。
- 标准 Debian 工具；容易推理；`apt` 处理依赖。

**我们案例的缺点**
- 没有健康门控回滚，也没有开箱即用的干净"回滚到先前版本"；我们无论如何都要在上面构建。
- 将 ML 模型作为普通版本化资产发布很笨拙。
- 不携带 phone-trigger / 进度 IPC——仍然需要包装。
- **可移植性：** 在未来的非 Debian 机器人（Yocto 等）上不存在，这与"跨机器人的通用工具"目标相悖。

**建议：** 从自定义 Rust 引擎开始，以获得可移植性 + 回滚；如果 Rust 引擎的维护成本超过其收益且机群保持仅限 Debian，重新审视 apt。

## 14. Phone / BLE 触发协议（草图）

`btd` 每个通道暴露两个 GATT characteristic（或一个带通道字段的）：

- **Trigger**（write）：`{ "cmd": "check" | "apply", "channel": "daemon", "version"?: "…" }`
- **Status**（read/notify）：`{ "channel", "state": idle|checking|downloading|applying|healthy|failed|rolled_back, "progress": 0-100, "version", "error"? }`

`btd` 通过 unix socket 转发到 `updaterd` 并将其状态镜像回来。Payload 永不经过 BLE。

## 15. 实现草图

`updaterd`，一个小的 Rust 二进制文件（几百行逻辑）：

- `reqwest` — 下载（带恢复/重试）+ GitHub API 用于最新标签查找
- `minisign-verify` — 签名验证
- `tar` + `zstd` — 解压
- `serde` / `toml` — 配置；`serde_json` — manifests
- 通过 `rename(2)` 的原子 symlink swap；通过现有 unix-socket IPC 的健康轮询
- systemd 集成：`Type=notify`、`WatchdogSec`、单元模板

CI（每个通道）：
1. 构建 artifact（可选通过 `cargo-dist`）
2. `minisign -S` artifact 和 manifest
3. 创建 GitHub Release，标签 `daemon-vX` / `model-vX`，以 artifact、`.minisig` 和 `manifest.json` 作为资产

## 16. 好邻居（范围内）

廉价的添加，插入相同的 IPC / 引擎并获得回报：

- **可重用健康自检模块。** 一个探针（电机 ack、模型加载、媒体初始化）由两个调用者调用：启动时和更新后作为健康门。
- **`robotctl` 本地 CLI。** `updaterd` 的 unix socket 上的瘦客户端——与 `btd` 相同的角色，不同的传输；它不持有自己的更新逻辑。
- **Dev sideload 路径。** 接受用不同的 **dev key** 签名的 artifact（存在于受信任集合中，但被门控在标志 / dev build 后面），因此团队可以刷新本地构建而不接触 prod 签名。
- **BLE provisioning 安全。** 相邻但重要：wifi 凭据在设置期间通过 BLE 传递。该 characteristic 必须配对 + 加密，否则就是凭据泄露。

## 17. 发布测试和信心

**动机。** 在以前的机器人上，release 是手动验证的——手动回退到已知版本，手动应用新版本，手动检查。人为错误的空间太大。这里的设计目标：**使更新路径可脚本化并自动测试，因此向客户发货是一个*决定*，而不是手动程序。**

关键框架：大部分人为错误表面（回退 → 应用 → 验证）存在于更新*机制*中，这是纯软件，可以在 CI 中测试，**不需要硬件**。只有真正依赖硬件的行为需要真实的板。

### 为可测试性设计

- **针对测试 artifact 的真实代码路径。** `updaterd` 从配置读取其 source/manifest，因此测试将其指向本地目录或 `staging` 通道并驱动*精确的生产代码*——没有与现实脱节的 mock updater。
- **可脚本化、幂等的原语**（通过 `robotctl`）：`apply --version X`、`rollback`、`pin <version>` / `unpin`、`model select`、**`reset-to-golden`**。
- **`--dry-run`：** 运行 fetch / verify / compat / 空间检查，并在 swap 之前停止。
- **故障注入 hooks：** 强制失败健康探针、损坏的 artifact、非零 hook、或 mid-swap 终止的标志——因此回滚是*被测试的*，而不是假设的。

### 两个测试层级

**层级 1 — 机制测试（CI，无硬件，快速）。** 在每个 PR 上运行；仅此一项就消除了大部分手动回退风险。

**层级 1b — 针对真实 `robotd` 进程**（`robotd/tests/updater_gate.rs`，仍然是 CI，仍然没有硬件）关闭了这一点。它生成实际的二进制文件，并让真实的 `SocketRobotClient` 通过真实的 unix socket 与它对话。

**层级 2 — 设备上验收（canary 机器人）。** 依赖硬件的行为（步态、电机、媒体）需要真实的板。保留几个跟踪 `staging` 的**实验室/canary 机器人**，在每个候选上自动更新，运行脚本化验收测试，并报告 pass/fail + 更新日志。

### 通道和晋升

实现为三个 GitHub Actions 工作流加上一个 `cargo xtask` 发布器：

| | |
|---|---|
| `ci.yml` | fmt、clippy、测试，加上 `board-test.sh`——唯一证明二进制文件在 aarch64 Linux 上运行的作业 |
| `release.yml` | 在 `daemon-staging-v*` 标签上：交叉构建、打包、签名、**用机器人自己的代码路径验证**、发布预发布 |
| `promote.yml` | 手动：重新签名 *stable* manifest，将验证过的 artifact 复制到 stable release，退役 staging |

三个值得陈述的属性，因为每个都是断言而非假设：

- **晋升永不重建。** stable manifest 携带 staging `sha256`，晋升在检查该摘要后将 staging artifact 复制到 stable release，因此客户收到的字节是 canary 验证过的字节。
- **Artifact 是可重现的。** tar 中的固定 mtime 意味着相同的输入产生相同的归档，因此重建可以与已发货的进行比较。
- **`release.yml` 在发布之前验证。** 它通过真实引擎安装 release（`updaterd install --from`，通过 `LocalDir` source）并断言二进制文件落地为可执行。

- 通道：`staging` → `stable`。CI 将候选发布到 `staging`。
- canary 机器人用**每个命令一个标志**获取候选：`sudo robotctl update apply daemon --staging`
- 在绿色时，**晋升**：将 `stable` 重新指向已验证的*相同字节*——重新签名 `stable` manifest 以引用相同的 tarball + hash。没有重建，没有重新刷新，没有手动复制文件。
- 坏的 `stable` → 反向相同的操作：将 `stable` 重新指向先前的已知良好 manifest；`min_supported` 然后在修复落地后将机器人向前拉。

### 来源

`version.toml` 和更新日志记录每个 artifact 的 git SHA + hash，因此"客户正在运行的确切版本"在实验室中始终可重现。

这关闭了上次受伤的循环：**人类决定*是否*晋升；机器每次都相同地执行回退 / 应用 / 验证。**

## 18. 开放问题 / 未来

- **配置所有权。** 配置目前随模型 bundle 一起。是否有任何配置应该属于守护进程通道，或成为自己的小通道？
- **最小的"成功了吗" phone-home。** 机群管理被推迟，但每次更新一个成功/失败 ping 将让我们及早发现坏 release，并且是使 `min_supported` 在实践中可操作的原因。值得吗？
- 分阶段推出 / 遥测 / 仪表板：明确推迟；如果机群增长则重新审视。
- 增量更新：推迟；在当前节奏下 artifact 很小。

本文档与实现之间的已知差距，故意开放：

- **一个模型槽内的竞争替代方案不受支持。** 每个模型是一个具有一个版本线的组件。它不覆盖"安装了两个不同的步行模型，用户选择一个"。
- **没有办法发现可安装的模型。** 应用可以列出*已安装的*，但没有 `Source` 可以回答"可以安装什么"。
- **没有恢复模式。** 回退链是 `current → previous → golden`，全部三个都已实现。最终的"仍然可以重新获取的最小恢复模式"不存在。
- **一个 `RobotClient` 服务于每个组件。** `HealthCheck::Socket { path }` 被尊重用于*运行哪个探针*，但 socket 路径本身来自 `main` 构造的任何东西。当两个组件都探测 `robotd` 时很好；如果一个组件曾经需要不同的对等方，则是陷阱。

仍然开放：

- **Phone 交付的 artifact 作为后备** — 对于没有可用互联网的机器人：从未 provisioned、captive portal、被阻止的 CDN、离线演示站点。
- **Manifest 陈旧报告** — 在 `status` 和应用中显示"上次成功检查 N 天前"，将冻结攻击从静默转换为可见。
- **配置所有权** — 配置随模型 bundle、守护进程、还是成为自己的小通道？
- **最小成功/失败 phone-home** — 每次更新一个 ping 将让我们及早发现坏 release，并且是使 `min_supported` 在实践中可操作的原因。
- **行为/大脑层** — 如果更高级别的行为层（驱动器、情绪、习惯）曾经作为自己的 artifact 发货，它将成为第三个通道，有自己的兼容性约束。

明确**不**做：硬件变体矩阵、外设固件 OTA、分阶段推出 / 遥测、增量更新。

自初稿以来决定：拆分托管（GitHub + HF Hub）；schema 中的 `min_supported` 地板；模型*bundle 带命名槽* + `model_api` 兼容性；多个受信任签名密钥；单一硬件目标；机器人特定状态保留；配置在文件中而非 systemd 单元中。
#（注：内容由AI生成）
