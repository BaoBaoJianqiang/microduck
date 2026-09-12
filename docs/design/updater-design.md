# 机器人 Daemon 更新系统 —— 设计

状态：草稿 · 日期：2026-07-22 · 负责人：pierre

## 1. 背景

机器人在 Raspberry-Pi 级别的板子（Raspberry Pi OS / Debian）上跑一个 Rust daemon。daemon 处理电机控制、运动学、一个小型 ML 步态模型，并通过 HTTP/WebRTC 推流视频/音频。蓝牙（BLE）服务暴露配置和 provisioning。机器人发货给非开发者客户；更新必须能从配套手机 app 点几下触发。

我们需要一种在现场安全更新已发货软件的方式。

排期和里程碑见 [`roadmap.md`](../project/roadmap.md)。

周围的系统——服务划分、IPC、状态归属、机器人 API 和远程/WebRTC 访问——见 [`architecture.md`](architecture.md)。本文只是更新系统。

## 2. 目标 / 非目标

**目标**
- 独立更新 **daemon 二进制**和 **ML 步态模型 + 配置**（它们实际上从不一起发货）。
- 支持**多个模型**（走、跳、站……）各自独立版本化和更新且全部同时加载，每个由 daemon 兼容性门控（§5.5）。
- 从手机 app 触发 + 监控更新。
- 通过最低版本下限**强制升级已知坏 release**。
- 永不变砖：原子 apply、自动回滚、签名 artifact。
- 保持简单；无机群后端。
- 能**在团队其他机器人上复用**，最小适配。
- 支持逐组件的**安装后钩子**用于迁移 / 一次性步骤。

**非目标（目前）**
- OS / 内核 / 驱动的 OTA（带外处理，见 §11）。
- 机群管理：分阶段推出、逐设备遥测、仪表盘。模型是"最新版本，所有机器人"。
- Delta/diff 更新。在我们节奏下完整 artifact 足够小。

## 3. 关键决策与理由

| Decision | Choice | Why |
|---|---|---|
| Update granularity | Application-level (not A/B image) | Only daemon + model change; OS is static. A/B (RAUC/Mender) would be over-engineering. |
| Transport | Robot **pulls** from CDN over its own wifi | Robot has internet; phone is trigger + progress only. Big payloads never touch BLE. |
| Hosting | **GitHub Releases** (daemon) + **HF Hub** (model) | Zero backend; each is a natural home for its artifact. Engine treats "source" as pluggable. |
| Signing | **minisign** (CI signs, robot verifies) | Battle-tested format, tiny verify crate, simple key management. |
| Channels | One per independently-versioned thing: `daemon`, plus one per model | Different cadences; models reload without a full restart. |
| Rollback | Versioned dirs + atomic symlink swap + health gate | Simple, robust, no partitioning needed. |
| Reuse | Config-driven generic engine | Same engine everywhere; each robot declares its components. |

*面向客户的 release* 的节奏预期：头几个月每约 2–3 周一个 daemon release，之后每约 2–3 个月。模型按自己独立的节奏更新。

但**内部**迭代热得多（原型 3 天约 8 个 tag），所以系统必须服务两种节奏：快的内部 `staging` 流和慢的、策展的 `stable` 流（§16.3）。这就是 staging 通道和脚本化回滚成为关键而非可选润色的原因。

### 3.1 前置条件：机器人需要自己的互联网接入

上面的传输决策有一个值得明说的设置后果：**更新要求机器人在自己的网络接口上有可用的互联网。** Wifi 配置因此是更新的前置条件，不是独立功能——引导顺序是*通过 BLE 配对 → 配置 wifi → 更新*——且从未配置过的机器人根本无法更新。

手机自己的连接不是替代。一旦计入连接间隔和 iOS 的节奏，真实持续的 BLE 应用吞吐约 10–30 kB/s，这让 daemon artifact 要数十分钟，模型 bundle 要数小时。这就是为什么 payload 从不经过 BLE（§13）：限制是链路，不是协议，所以再多帧定界工作也找不回来。

完全没网络时仍工作的：`rollback` 和 `reset-to-golden`（§8.2）纯对磁盘上已有的 release 操作。它们不需要 manifest 和源，不移动字节，所以它们轻松适合 BLE 且 `btd` 即使 `robotd` 死了也服务它们（§4.1）。**未配置或离线的机器人因此可恢复，只是不能升级**——该权衡正确的失败一侧。

通过手机交付 artifact 作为回退是可能的，但不是正常路径且仍需要真实链路（wifi 或 USB，绝不是 BLE）。见 §17。

## 4. 架构

`systemd` 是监督者（生命周期、崩溃重启、顺序、看门狗）。

```
systemd
├── robotd     motor control · kinematics · gait ML · sensor loop   [RT-ish core]
├── mediad     video/audio capture · WebRTC/HTTP                     [isolated: media crash ≠ dead motors]
├── btd        BLE GATT server (bluer): wifi provisioning, naming, update trigger/progress
└── updaterd   the update engine (this document)
                    ↕ local IPC (unix socket) with robotd + btd
```

**结构规则：** updater 是与它更新的东西*分开的 unit*。进程不能干净地替换自己运行的二进制，且 updater 必须在 daemon 崩溃中存活以执行回滚。

**推论 —— `updaterd` 必须常驻，且必须把自己排除在重启集外。** `updaterd` 和 `btd` 都发货在 daemon artifact *内部*，所以天真的"重启一切"会在交换中途或健康门控中途杀死执行者。`on_apply` 因此重启 release 发货的一切**除了**这两个。

**排除在进行中重启外不等于跳过**，那样读是 bug。把它们推迟到"下次启动或显式稍后重启"让两者无限期跑旧二进制：常驻的 `updaterd` 用"client speaks API v4, daemon speaks v3"拒绝较新的 `robotctl`（那个拒绝没了——过时二进制是错，不是不一致），且 `btd` 修复是针对从未运行过的二进制测试的。所以 `RESTART_AFTER_REPLYING` 通过 systemd 瞬态 timer 在结果上线 5 秒后调度每个——足够一次写入，短到没人在等。每个排除的理由恰在那一刻过期：更新完成，`btd` 携带的回复已交付。

这里没有东西等重启，`robotctl health` 的 unit 块是确认它的地方：它打印每个进程*运行自*的 release，那是失败的延迟重启会显示的唯一地方。

集合从 release 自己的 `systemd/*.service` 文件推导而非从板子配置读取，两个排除住在代码里（`NEVER_RESTART`）而非配置中：它们是那些 daemon *是什么*的性质，不是操作员该能搞错的选择。早期设计把列表放在 `/etc/robot/updater.toml`，`install.sh` 保留它——所以在某个 daemon 存在之前配置的板子从不重启它，更新仍说成功（`install-path-gap.md` §4）。

**这不覆盖什么，以及答案的形态。** 不重启 `updaterd` 保护*进行中*的更新。它没说新 `updaterd` 是否工作，且把发现推迟到下次启动——那时更新已提交、没人在看、恢复住在 `Engine::recover_on_start`，即**在启动失败的进程内部**。`Restart=on-failure` 然后崩溃循环几次放弃，留下没有更新 daemon 且无法更新的机器人。那与恢复在机器人已坏时工作的承诺矛盾。

两层关上它，且它们抓不同的东西。**第一层已实现**；第二层是它自己的 PR。

1. **提交前自检新二进制**，并在发送回复后重启 `updaterd`。只读模式——配置加载、引擎构造、在恢复*之前*退出——抓错误架构、缺失库、立即 panic，以及最可能的一个：拒绝板子已有 `updater.toml` 的新 `updaterd`，那是操作员的文件且跨安装保留。在那里失败廉价回滚，无重启。重启必须是**分离的**，因为引擎在 `update.apply` RPC 内运行，否则会给客户端一个坏管道而非结果。同样的"回复后重启"推理延伸到 `btd`。

   注意*现有*的 `--check-only` 不是什么：它在尊重标志之前真实运行 `recover_on_start`，所以它递增每个已武装 trial 的启动计数且能回退更新。它是操作员工具，不是探针，在更新中途用它会让第二个引擎变异第一个正在工作的存储。

   构建时：`self_test_updaterd` 在已发货 units 重启之后、健康门控之前运行，仅在 `on_apply` 是重启的地方——模型组件没有 `updaterd`，且引导安装强制 `on_apply=none` 正因为还没装任何东西。`Error::SelfTest` 携带二进制最后一行 stderr，所以"config error: unknown field `foo`"到达回滚原因而非被压平成"不可达"。

   重启通过 `systemd-run --on-active=5s`，那里两个细节是关键的。*子进程*会坐在 `updaterd` 的 cgroup 里并在重启自己父进程中途被杀死；瞬态 unit 不会。且更新锁在派生任何东西**之前**释放，因为 fork 复制进程中每个打开的描述符——包括同一进程中其他引擎持有的锁，那在测试套件中表现为不相关操作以 `Busy` 失败。

2. **`updaterd` 之外的启动时网**，针对漏过的东西：每次启动三分钟后的 timer 问 release 是否把它的 daemon 带起来了，已安装基础中的 `/bin/sh` 救援在没有时把 `current` 交换到 golden。它必须住在这个进程之外和 release 之外，因为它存在要应对的失败是不启动的 `updaterd`——那时 `recover_on_start` 和下面的启动计数器都不可达。它不能修硬件：因缺舵机电源而失败的 `robotd` 在 golden 上同样失败，与健康门控在不健康和降级之间画的区分相同。[`boot-recovery-net.md`](boot-recovery-net.md) 负责它。

这也是为什么更新逻辑不能住在 `btd`：作为更新的客户端，`btd` 不能是执行它的东西——它会在中途杀死自己。`btd` 保持薄传输，常驻的 `updaterd` 拥有状态机、single-flight 锁、启动恢复，以及任何完全无客户端时的定期/强制检查（§8.1）。

`btd` 是薄中继：它把"check" / "apply"转发到 `updaterd` 并把进度流回手机。它从不自己移动 payload。

### 4.1 不变量：`btd` 和 `updaterd` 在 `robotd` 死时存活

**`btd` 和 `updaterd` 必须即使 `robotd` 崩溃、崩溃循环或不存在也能启动、运行并保持完全可用。** 它们是恢复路径——如果它们只能在机器人健康时工作，那么客户端真正需要它们的那个情况就是它们不工作的情况。这是不让现场机器人变砖的最重要结构性质。

具体地：
- **对 `robotd` 无 systemd 依赖。** 无 `Requires=`/`After=robotd`，无共同命运。`btd` 和 `updaterd` 启动时独立起来并保持运行。
- **无硬 IPC 依赖。** 每次进入 `robotd` 的调用（"safe to restart?"检查、健康探测、模型重载）是*可选且有超时上限*的。死掉的 socket 是正常、预期的应答——从不挂起、从不 panic、从不拒绝服务手机。
- **降级模式语义。** `robotd` 宕机时，safe-to-restart 检查平凡通过（没东西在动），`updaterd` 继续。健康门控回退到"新 `robotd` 是否起来并报告健康"——那正是从坏 release 恢复时重要的检查。
- **`btd` 报告真相。** app 必须能看到"daemon 不健康、版本 X、上次更新失败"并从该状态提供*更新* / *回滚* / *重置到 golden*（§8.2）。不会走的机器人必须仍能解释自己并接受修复。
- **最小依赖面。** 这两个是最后防线，所以让它们无聊：少依赖、无 ML 运行时、无媒体栈、无 GPU/摄像头访问。有趣代码里的 bug 绝不能带走恢复路径。
- **可分别更新，保守。** `btd`/`updaterd` 发货在 daemon artifact 里，所以坏 release 原则上也能破坏它们——那正是 golden release（§8.2）和启动计数器回退（§8）存在要抓的。把对这两个的改动当作比对 `robotd` 的改动更高风险。

`updaterd` 的推论：它也不应该*要求* `btd`。本地 `robotctl`（§15）通过 unix socket 必须能驱动恢复，即使 BLE 本身坏了。

## 5. 更新模型

### 5.1 通道

两个逻辑通道，各有自己的 manifest 和版本线：

- `daemon` —— `robotd` / `mediad` / `btd` 二进制 + 支持文件。
- `model` —— 步态模型权重 + 行为/配置 bundle。

两个通道住在**不同主机**上，引擎通过每组件可插拔的 `source` 处理：
- `daemon` → **GitHub Releases**，tag `daemon-v1.4.2`，assets = artifact + `.minisig` + manifest。
- `model` → **HF Hub** 仓库，按 git revision/tag 版本化；文件在 `https://huggingface.co/ORG/MODEL/resolve/<rev>/<file>` 解析。"Latest" = 移动的 tag/分支或最新 tag。我们仍自己 `minisign` 模型 artifact 并把 `.minisig` 存旁边——HF 不为我们签名。

### 5.2 Artifact

每个 release 一个压缩 tarball，例如 `daemon-1.4.2.tar.zst`，包含：

```
bin/robotd  bin/mediad  bin/btd        # (daemon channel)
model/gait.onnx  config/…              # (model channel)
version.toml                           # semver, min_hw_rev, schema_version
hooks/postinstall                      # optional, executable, versioned+signed
```

### 5.3 Manifest

作为 release asset 发布的小 JSON（并由稳定 URL 引用，见 §6）。用 minisign 签名。

```json
{
  "channel": "daemon",
  "version": "1.4.2",
  "url": "https://github.com/ORG/REPO/releases/download/daemon-v1.4.2/daemon-1.4.2.tar.zst",
  "sha256": "…",
  "sig_url": "https://github.com/ORG/REPO/releases/download/daemon-v1.4.2/daemon-1.4.2.tar.zst.minisig",
  "min_hw_rev": 3,
  "schema_version": 2,
  "min_supported": "1.5.1",
  "changelog": "…"
}
```

`schema_version` 是**迁移上下文，不是兼容性门控**。基于它门控会自败：判断 manifest 的引擎总是*前一个* release 的引擎，所以拒绝 `schema_version > supported` 会让每个 schema 升级都无法交付——包括携带理解它的引擎的 release。安装后钩子执行迁移；引擎把数字透传（§9）。

`min_supported` 是**最低版本下限**（§8.1）。从 v1 起包含在 schema 中即使起初未用——它不能被后装到从未学会找它的机器人上。

刻意目前**没有** `not_valid_after` 字段：manifest 过期是对冻结攻击的防御，承担其运营成本的决定在 §8.4.2 推迟。以后加它是附加的（缺失 = 无过期），不像 `min_supported`，所以现在不需要预留。`model` manifest 另外携带 `model_api`（§5.5）。

### 5.4 签名 / 信任

- CI 用私有 minisign 密钥（存在 CI 机密中，理想是离线主密钥）同时签名 **artifact** 和 **manifest**。
- 机器人发货时带**一组受信任的 minisign 公钥**（`/etc/robot/trusted_keys/`），不是单个密钥。签名若能针对*任一*受信任密钥验证即有效。这是信任锚——无需 PKI。
- **密钥轮换：** 从第一天起发货*一组*意味着丢失/泄露的密钥可存活——发布一个由现有密钥签名的更新添加新密钥并（稍后）退役旧密钥。单个内置密钥会让丢失密钥成为不可恢复的死更新路径。
- 可选在组中预留一个独立的**dev 密钥**，由标志门控，用于团队本地侧载路径（§15）。
- 机器人上的验证顺序：验证 manifest 签名 → 下载 artifact → 验证 sha256 → 验证 artifact 签名。**无符号字节从不被执行或解压到活动路径。**

#### 密钥保管

用 `cargo xtask keygen` 生成密钥，它拒绝写入仓库并把密钥写成 `0600`。

| key | encrypted | private half lives | in trusted set of |
|---|---|---|---|
| `release-1.pub/.key` | yes | password manager **and** CI secrets | every robot |
| `release-2.pub/.key` | yes, **different passphrase** | password manager only — **never CI** | every robot |
| `release-3.pub/.key` | yes, **different passphrase** | offline; ideally never on a networked machine | every robot |
| `team.dev.pub/.key` | no | team store + CI | developer boards only |

这个表编码的三件事，每件都容易搞错：

1. **备用必须从第一个镜像起发货。** 机器人针对内置的密钥集验证，所以替换不能以后通过空中引入——丢失唯一密钥意味着手动重刷每台机器人。
2. **备用必须在密码*和*暴露上不同。** 它的目的是在第一个被攻破时存活；`release.key` 因必要在 CI 中，所以 `release-2.key` 不能在。共享密码把两者塌成一个。
3. **密码是生成的，不是选的。** 它们从不被打——住在密码管理器和机密存储里——所以可记忆性换不来什么，而 minisign 的 scrypt 参数（32 MiB，opslimit 2²⁰）对针对泄露密钥文件的离线攻击只是适度刹车。≥128 位，例如 `openssl rand -base64 24`。

Release 在 **CI** 中签名，不是本地——刻意选择，补偿它的审批门在 [`ci-setup.md`](../project/ci-setup.md) 文档化。

dev 密钥刻意不加密：CI 非交互签名，存在密钥旁的密码加不了多少。它真正的保护是结构性的——它不在客户机器人的受信任集中，且在那里由 `allow_dev_keys = false` 门控。

### 5.5 模型：每个一个组件

有几个模型——走、跳、站、地面拾取——**各自独立版本化**，且全部同时加载而非从多个中选一个。

那正是*组件*是什么：有自己版本线的东西。所以每个模型在 `updater.toml` 里是自己的条目，不需要特殊存储布局：

```toml
[component.model-walk]
install_dir = "/opt/robot/model/walk"
source      = { type = "hf_hub", repo = "ORG/gait-walk", revision = "main" }
on_apply    = { action = "reload", unit = "robotd", signal = "SIGHUP" }

[component.model-jump]
install_dir = "/opt/robot/model/jump"
source      = { type = "hf_hub", repo = "ORG/gait-jump", revision = "main" }
on_apply    = { action = "reload", unit = "robotd", signal = "SIGHUP" }
```

每个然后免费且独立地得到：自己的回滚目标、golden release、pin、启动计数器 trial 和已知坏历史。`robotctl update apply model-walk` 更新一个模型不碰其他。尝试某个的旧版本是 `robotctl update select model-walk 1.1.0`——`select` 重新指向任何已安装 release 而不下载。

模型用 `reload`（SIGHUP → 重新 `mmap` 权重）而非 `restart`，所以权重交换从不丢掉电机控制。逐组件 `on_apply` 是让这自然的原因，也是模型和 daemon 保持独立组件的核心原因。

**"库布局"曾被尝试并移除。** 早期草稿有一个 `Layout::Library` 模式（`library/<ver>/` + 一个 `active` 符号链接）意在容纳几个命名 bundle 选一个。它实际做不到——它只变两个目录名，所以能力上与正常布局相同却*看起来*像功能。更糟的是，"多个已安装，一个活动"不是模型的形态：它们同时全部活动。删除。

如果*单个*槽将来需要竞争替代（两个不同走模型，用户选一个），那是真正的 `(name, version)` 存储键和真正的变更——见 §17。当前设计中没有东西假装支持它。

**兼容性（保持轻量）。** Daemon 广告一个**模型 API 版本**（`model_api`，例如 `1`），描述它实现的传感器输入 / 执行器输出契约。每个模型 manifest 声明它要求的 `model_api`。规则：模型仅在 `model.model_api <= daemon.model_api` 时加载。

API 版本在接口变化时升级——例如模型开始消费**新传感器输入**或发出**新输出**。这意味着：
- 需要 API `2` 的模型在只说 `1` 的 daemon 上被拒绝 → app 提示先更新 daemon。
- 旧模型在较新 daemon 上继续工作（向后兼容），所以 daemon 升级不会破坏东西。

不可达的 `robotd` 让这*未知*而非不兼容，那是等模型但不等 daemon 的理由——见 §5.4 的 `Compatibility` 注释。

### 5.6 硬件：单一目标

**当前目标：Radxa Zero 3**（RK3566、Cortex-A55 → `aarch64`）跑 **Armbian 26.2.x** 带 **Debian 13 (Trixie)** 用户态、Rockchip BSP 内核 6.1.115。暂定——手边最近的板子而非最终选择——但它是构建和测试工具瞄准的。

构建和验证：`scripts/board-test.sh` 用 `cargo-zigbuild` 交叉编译（`zig cc` 提供 `zstd` 需要的交叉 sysroot 和 C 编译器）并在容器中的真实 ARM64 Linux 上运行二进制。

glibc 下限钉在 **2.31**（实际下限 2.30）而非 Trixie 的 2.41，脚本仅在 **Trixie** 上运行二进制——我们发货的用户态。Armbian 为此板提供其他；测试没人跑的配置花作业时间捍卫一个我们不需要的主张，`BOARD_IMAGES=` 为一次性检查覆盖它。

钉住不是关于那些替代。它防御*构建主机*：不钉的话，构建链接到开发者机器或 CI runner 上的任何 glibc，那比板子的新——它链接干净然后在机器人上加载失败，构建输出里没有东西指向原因。2.31 就是远低于目标，不花代价。

内核版本不是约束：`flock`、`SO_PEERCRED` 和 `statvfs` 都远早于 6.1。

容器覆盖不了、硬件必须做的：真实 `systemctl restart`（`on_apply` 从未对真实 systemd 运行过）、eMMC 写入行为和时序、对真实文件系统的 `statvfs`、以及健康门控超时是否适合需要数十秒站起来的机器人。

**v1 面向一个明确规格的硬件配置。** 无板子 / 变体 / IMU / 摄像头矩阵、无逐设备硬件 profile、无 artifact 选择逻辑。原型的硬件方差是探索的产物，不是要带下去的需求。

我们保留的刻意最小：
- manifest 中的 `min_hw_rev`（§5.3）作为单个前向兼容守卫，让未来板子修订能拒绝不兼容的 artifact。一个整数，无矩阵。
- 更新日志（§8.3）和任何未来电话回的**稳定设备 ID**。SoC 序列号（`/proc/device-tree/serial-number`）可用且在重刷后存活——无需 provisioning 步骤获取。

如果第二个硬件目标出现，manifest 加约束，引擎加匹配步骤。在此之前——这正是要避免的投机复杂度。

### 5.7 机器人特定状态必须在更新中存活

*确实*带下去的要求：有些文件属于**这台**机器人，更新或回滚绝不能销毁它们。三类，区别对待：

| Class | Examples | Lifecycle |
|---|---|---|
| **Shipped** (replaced) | binaries, policy bundles, default config, static systemd units | Under `releases/<ver>/`; swapped atomically |
| **Robot-specific** (never touched) | calibration data, per-device generated assets, learned/persisted state (maps, personality, habits) | Outside release dirs; **preserved across update *and* rollback** |
| **User preferences** (preserved, migrated) | robot name, wifi credentials, active model selection, app-set tunables | Own config file; migrated by hooks (§9) on `schema_version` bumps |

两条规则：
1. Release 目录是**可丢弃的**。任何必须存活的东西住在 `/etc/robot/` 或 `/var/lib/robot/`。回滚不能丢用户数据或迫使客户端重新校准。
2. **运行时配置住在配置文件里，不在 systemd unit 里。** Units 发货且静态；它们读 updater 从不覆盖的配置文件。（原型把 CLI 标志在安装时烤进生成的 unit，所以重装静默重置设置——一个值得点名的陷阱。）

推论：逐设备资产的再生必须基于稳定种子且仅在真正需要时做，绝不在每次更新无条件做。

## 6. 托管在 GitHub Releases

- Artifact + `.minisig` + `manifest.json` 是每个 release 的 assets。
- "最新是什么？"通过以下之一解析：
  - **稳定重定向 URL** `…/releases/latest/download/<asset>`——但 GitHub 的"latest"是仓库级的，所以这只在**每仓库一个通道**时干净工作。
  - **GitHub API** 查询匹配通道前缀的最新 tag（`daemon-v*`）——适用于单一共享仓库。如果两个通道在一个仓库里首选。
- CI 侧：`cargo-dist` 可构建并发布签名 artifact 到 GitHub Releases；我们把 manifest 作为额外 asset 托管。

### 6.1 私有仓库不能服务机群——所以这个不留私有

**已定（2026-08-26）：发布 `pollen-robotics/microduck`。** 私有期间现场机器人下载不了任何东西，原因值得保留因为不明显：私有仓库的 `releases/download/<tag>/<asset>` URL **带或不带 token 都返回 404**。直接验证：

| URL | private repo |
|---|---|
| `https://github.com/<repo>/releases/download/<tag>/<asset>` | 404, authenticated or not |
| `https://api.github.com/repos/<repo>/releases/assets/<id>` + `Accept: application/octet-stream` | 200 with a token |

所以引擎通过 API 端点解析每个 asset，这对**开发者的板子**有效——`GITHUB_TOKEN` 在环境中且 `--ref` 安装分支构建——和公开仓库，但对客户机器人不行，它没有 token 也不该有：烤进镜像的机群凭据会泄露且无法轮换，与签名密钥分层要避免的同样问题。

桌上还有三个其他选项，记录下来因为如果源需要再次关闭决定可重审：

| option | keeps zero-backend | notes |
|---|---|---|
| A **public repo holding only release artifacts** | yes | signatures are what make an artifact safe, not obscurity — an artifact repo leaks build metadata and nothing else. Source stays private. The fallback if this repo ever goes private again. |
| An object store or CDN with a plain HTTP source | mostly | one more thing to own and pay for; the source trait already abstracts it. |
| A read-only token in the image | yes | rejected: an unrotatable fleet credential. |

**公开不改代码。** API 路径保持正确——它是私有和公开共有的路径，这就是那样写的原因。

**它改的是预算。** 未认证 GitHub API 请求限每小时每 IP 60 次，token 提升它；别人家里的机器人没有 token，所以它的检查从该地址背后一切共享的匿名池里花。在 `check_interval = "6h"` 和每次检查少数调用下，一只鸭子远碰不到它——同一 wifi 上一屋子二十只一起更新可能。这不是正确性问题：`http.rs` 已把 403 和 429 读为"稍后回来"并在消息中说明。这是仓库公开后偏好 `browser_download_url` 取字节的理由，因为从 `objects.githubusercontent.com` 下载对象不从那个池花任何东西，且保持 API 路径用于私有仓库和开发板。值得在一屋子鸭子存在之前做，而非第一只发货之前。

## 7. `updaterd` 状态机

```
receive trigger (from btd)  ── check | apply(version)
        │
        ▼
PREFLIGHT (§7.2):  single-flight lock · clock/NTP sane · robot stopped · disk space free
        │  (any fail → report + exit, no changes)
        ▼
fetch manifest ──► verify manifest signature ──► compare version, check min_hw_rev / model_api / pin / downgrade
        │                                                    │ (nothing to do / incompatible)
        ▼                                                    └─► report + exit
download artifact ──► verify sha256 ──► verify artifact signature
        │
        ▼
extract to releases/<ver>.tmp/  ──► orphaned-unit check (§7.3) ──► [pre_install hook]
        │                                    │ (an installed unit execs a binary this release lacks)
        │                                    └─► report + exit, nothing swapped
        ▼
atomic symlink swap:  current → releases/<ver>        (rename(2), same fs)
        │
        ▼
[post_install hook]  ──► apply: restart (daemon) | reload+SIGHUP (model)
        │
        ▼
HEALTH GATE (poll robotd over unix socket, timeout per component)
        │
   ┌────┴─────────────────────────┐
 healthy                        not healthy / hook failed / timeout
   │                                │
 prune old (keep_previous)      ROLLBACK: swap symlink back, re-apply, mark failed
   │                                │
 report success                 report failure
```

任何非零钩子退出、失败的健康探测或超时被同样对待：**中止并回滚**。

### 7.1 磁盘布局

```
/opt/robot/daemon/
├── releases/1.4.1/     ← previous (kept for rollback)
├── releases/1.4.2/     ← new
├── current → releases/1.4.2     ← systemd units point at current/
└── golden  → releases/1.4.1     ← never pruned; what the rescue reads
```

原子性是同一文件系统上符号链接的单次 `rename(2)`。从不有半写状态是活动的。

`golden` 由 `Engine::refresh_golden_links` 在每次启动时从 `ComponentConfig::golden` 写入。它存在以便此进程外的恢复不需要解析器——见 [`boot-recovery-net.md`](boot-recovery-net.md)。

### 7.2 Preflight 前置条件

在任何下载或变更前检查；任何失败干净中止无副作用：

- **Single-flight 锁。** `updaterd` 持有锁文件 / socket，所以两个触发（或重试）不会冲突。进行中的更新报告"busy"，从不启动第二个。
- **手机无关。** 机器人拉取，所以即使 BLE 断开更新也跑到底。状态被持久化并在重连时回放给手机——手机从不需要保持连接。
- **时钟健全。** Pi/eMMC 板没有电池供电的 RTC；错误时钟在任何下载前让 HTTPS 证书日期验证失败。要求 NTP 同步（或健全时钟检查 + 有界重试）作为前置条件。minisign 本身与时间无关。
- **机器人停止。** 我们假设 app 只在机器人停止/停放（电机安全）时提供更新。`updaterd` 仍做一个轻的"safe to restart?"查询到 `robotd` 并不满足时拒绝——防止在运动中重启电机控制的廉价保险。
- **无活动远程会话。** 在远程临场中途重启 `robotd`/`mediad` 是坏意外；拒绝，或警告并要求显式确认。见 [`architecture.md`](architecture.md) §5。
- **磁盘空间。** 存储是 eMMC（有限，磨损比 SD 不是问题，但空间仍是）。开始前验证下载 + 解压 + `keep_previous` 的空闲空间。

### 7.3 这个 release 会让已安装 unit 成为孤儿吗？

`hooks/postinstall` 安装 release 发货的 units 并在回滚时留下它们（§9）。对回滚那是对的——下一次成功更新重新安装它发货的任何东西。对**降级越过引入某个 daemon 的 release** 不是：unit 留下，它的 `ExecStart` 指向旧 release 不含的二进制，systemd 以 `203/EXEC` 失败它，且因为那个 daemon 在派生的重启集中，失败的重启使更新失败，后者回退。在一块解析到早于 `configd` 的 stable `0.2.0` 的板子上观察到。

所以缺少某个已安装 unit exec 的二进制的候选被拒绝——`updater/src/orphan.rs`，`Error::WouldOrphanUnit`。它读 `/etc/systemd/system/*.service` 过滤到 `Exec*=` 指向组件 `current` 符号链接的 units：活动目录是孤儿出现的唯一地方，因为它比安装它的 release 活得久，过滤是把我们任何 release 都没发货的 units 挡在外面的东西。它解析不了的任何东西不产生发现——因解析器只是不懂的 unit 文件拒绝更新是更糟的失败。

三个值得保留的放置决策：

- **不是 preflight（§7.2）。** 两次 preflight 都在 artifact 下载前运行，所以候选的文件列表还不存在。这在解压后交换前运行，那仍是"无副作用"：staging 可丢弃、启动计数器未武装、`current` 没动。它花一次下载才知道。
- **在 dry run 返回之前**，因为"这个降级能工作吗？"正是 dry run 被问的。
- **无目标豁免**，不像 `WouldDowngrade` 守卫只在 `Latest` 触发。那个是关于镜像提供过时 manifest；这个是关于不会启动的 unit，它不在乎目标怎么命名——且 `Ref` 是案例被观察到的方式。

不在回滚、reset-to-golden 或 `select` 上：那些故意向后移动且是板子脱离坏 release 的方式，所以能拒绝的东西不属于恢复路径（[`architecture.md`](architecture.md) §1.1）。

拒绝点名 unit、缺失的二进制和绕过方式——移除 unit：

```
systemctl disable --now configd.service && rm /etc/systemd/system/configd.service
```

刻意不是 `--force` 标志。移除 unit 是操作员反正的意思，因为低于引入 daemon 的 release 的板子不该跑那个 daemon；它让情况成真而非覆盖说它不真的检查，且下次发货该 unit 的更新会重新安装它。

## 8. 健康门控与回滚

- Apply 后，新 `robotd` 必须在逐组件超时内清掉**健康标志**（电机 ack、模型加载、媒体流水线初始化）。
- 对硬挂起（不只是干净失败）的双保险：
  - `robotd` 上的 `systemd` `WatchdogSec` 让挂起触发恢复。
  - `updaterd` 启动时检查的**启动计数器**文件：如果上次更新在 2 次启动内从未到达"健康"，问机器人然后决定。覆盖"启动但病态"和"根本起不来"。

    **预算决定何时问；机器人决定是否回退。** 到达预算末尾意味着没有 apply 确认过 trial——通常原因不是坏 release 而是 apply 在其门控运行前被杀，那正是 release 自己的 `hooks/postinstall` 重启 `updaterd` 时发生的。所以健康门控问的同样三选一问题：*健康*提交，*降级*提交，其他回退。

    降级提交出于 §4 启动网给的不能修硬件的同样理由：没有舵机电源的 `robotd` 在 golden 上同样失败。在那里回退把硬件故障藏在软件变更后面，也回退下一个 release，且——最糟——在不告知的情况下替换扶着机器人的人脚下的代码。那不是假设：一块装了分支构建并在上面配对手柄的板子两次启动后回来跑 stable release，之后每条命令都针对没人要求的代码运行。

    仍回退的是这个网要做的：`robotd` 不健康、不可达，或以这个 `updaterd` 读不了的形态应答。
- `keep_previous`（默认 1）保留 release 目录限制磁盘使用同时总留下一个已知良好的回滚目标。

### 8.1 最低版本下限 / 终止开关

**已实现**，包括让它工作的部分：配置中的 `check_interval` 让常驻 `updaterd` 按时钟轮询每个源，`auto_apply` 决定那个时钟可以**不等客户端**安装什么。一次调度的扫描在另一个更新进行中时完全跳过，首次检查在启动后延迟（网络通常还没起来，且一起重启的机群会成群到达）。

不设 `check_interval` 禁用轮询——也禁用下限，因为没人点更新的机器人永远学不到下限存在。`updaterd` 在那种情况启动时警告，且在 `auto_apply` 设了但无时钟运行它时明确警告。

#### 8.1.1 `auto_apply` —— 无人值守更新策略

| | |
|---|---|
| `off` | never; availability is logged, a mandatory release loudly |
| `mandatory` | **default** — only a release the floor marks mandatory. Ordinary ones wait for a client |
| `all` | every available release |

一个有序设置而非每紧急度一个标志，所以配置不能表达"自动应用普通更新但不自动应用强制的"——自动更新除了发布来救坏机群的 release 之外的一切。

`mandatory` 是默认因为替代是机群卡在我们已撤回的 release 上。普通 release 等待因为*机器人何时重启是其主人的决定*，那正是整个 app 驱动更新流程要给他们的决定。

`all` 是 canary 和实验台设置，且它是让 §16.2 的 Tier 2 可配置的：跟踪 `staging` 并安装每个候选的实验室机器人。把它发货给客户机器人会把重启决定从他们手里拿走。

无论策略如何，无人值守 apply 是**普通** apply。它跑同样的 preflight——所以正在走或有远程会话的机器人拒绝并在下一个间隔重试而非在人手底下重启——和同样的健康门控，所以起不来的 release 被回滚。这也是为什么不需要维护窗口：`safeToRestart` 是对"现在是坏时机吗"比时钟更好的回答，且它在任何网络访问*之前*运行，所以忙机器人跳过不花代价。

**这属于 daemon，不属于 cron。** 调用 `robotctl update apply` 的 systemd timer 或 crontab 看起来等价且不是：显式 apply 刻意绕过下面的已知坏守卫，因为重试 release 的操作员可能已修了原因。外部 timer 会继承那个绕过并失去保护——每个的错的那一半——并重新引入下一段要防止的循环。

**无人值守路径拒绝这台机器人已经从其回滚的候选**，无论策略且即使 release 是强制的。没有那个守卫，坏 release 是无出口的机群陷阱：check 说可用 → apply → 门控失败 → 回滚 → 等 `check_interval` → 重复。每个周期重新下载 artifact、重写 eMMC 并重启 `robotd`，所以每台机器人既不可用又在磨损，在电池上，无客户端参与察觉。循环里没有东西收敛。

守卫是 `known_bad`，从 journal 的每个版本*最新*结果派生，所以如果 release 真的成功它会自清除。它**仅**适用于无人值守路径：显式 `robotctl update apply` 仍重试，因为操作员可能已修原因且拒绝他们会移除检查的明显方式。守卫触发时以 `error` 记录——卡在强制下限以下的机器人需要修过的 release，那该大声。

对此的测试在 journal 里计数尝试而非检查实时版本：每个周期后符号链接回到好 release，所以即使循环运行时机器人的*状态*看起来正确。

manifest 的 `min_supported` 让我们即使在"全部最新"（否则机器人只在被点时更新）下也强制机器人离开已知坏 release：

- 在 check/start 时，如果运行版本 `< min_supported`，更新是**强制的**——`updaterd` 不等点击就应用它（且运行的坏版本可被拒绝）。
- 更生硬的**终止开关**（签名的"版本 X 已撤销"声明）是同一机制的升级。被签名意味着攻击者不能*伪造*一个——但见 §8.4：签名说 artifact 是**我们的**，不是说它是**当前的**，所以仅签名不阻止重放旧的真 manifest。
- 现在把字段放进 schema（§5.3）；以后不能后装。
- `min_supported` 仅在机器人成功获取携带它的 manifest 后生效。它因此是*补救*工具，不是防御：被切断更新的机器人永远学不到下限存在。

### 8.2 Golden release 与恢复

- 保留一个已知良好的 **golden** release，*从不被修剪*（与 `keep_previous` 轮换分开）。
- 如果 current 和 previous 都无法健康起来，回退到 golden，失败的话一个仍能联网并重新获取的最小**恢复模式**。机器人必须总能打电话回家而非需要 RMA / 上门。

### 8.3 设备上更新日志，以及运行记录

把最后 N 次更新尝试（时间戳、通道、from→to 版本、结果、错误）持久化到磁盘，可通过 BLE/HTTP 检索。这是客户报告"更新失败"时支持需要的第一件事。

日志回答*一次尝试发生了且如何结束*。它回答不了的是尝试**做了**什么，那个问题也有一个诚实的家。

**为什么不是 journal。** `updaterd` 把它的显著事件记到 systemd，对看更新发生那是对的地方——`journalctl -f` 不需要机制。它是*保留*它们的错地方。这块板上的 `/var/log` 是 zram 设备，所以 `Storage=persistent` drop-in 换来的是干净重启的存活而非断电的存活——而任何人需要这个的更新不成比例地是以断电结束的那些。这与最初把更新日志放在 journal 外的论据相同；记录是它的第二次应用。

更糟的是，阶段时间线根本从未在 journal 里。阶段只作为 `update.progress` 通知发出，所以状态机自己的记述恰好存在于客户端保持 socket 打开的时长内，没人看的 `robotctl update apply` 留下缺了中间的大纲。

**一次运行记录什么。** 每次运行一个文件，`state_dir` 中的 `runs/<id>.jsonl`，每行按与日志相同的规则 `fsync`——在每个 `install_dir` 之外，所以交换或回滚不能销毁交换或回滚的记述（§5.7）：

- 运行的开头：调用者点名的目标、源、活动的是什么、谁问的，来自 `SO_PEERCRED`；
- 通过签名检查的 manifest——版本、哈希、大小、URL、**哪个**受信任密钥接纳它、以及它构建自的 revision；
- 每个阶段边界，有细节的地方带细节，因此每阶段花的时间；
- 钩子输出逐字，带退出码，成功和失败都记；
- 每个被重启、重载或延迟的 unit，以及 systemd 说的；
- 健康门控的判定，那是决定提交还是回滚的；
- 它如何结束。

日志条目携带运行号，所以两者像 `git log` 和 `git show` 那样组合：`robotctl update log` 是索引，`robotctl update show` 是细节。

**记录从不使更新失败。** 每次写都是尽力而为并把失败报告到 journal。完成了却丢了日记的更新严格好过为保持日记诚实而放弃的更新。

**在别处结束的运行。** Daemon 更新重启 `updaterd` 自己，所以它的记录按设计在中途停止，判定在数分钟和一次重启后从启动计数器 trial 到达。那次回退开一个**自己的运行**而非重开第一个——第一个进程没了不能追加。配对是可读的：没有 `ended` 的记录是判定在别处的运行，`show` 在本该有判定的地方说明，下一次运行就是那个判定。rescue-to-golden 路径，由 shell 脚本在 `updaterd` 启动前执行，得到一个从它留下的面包屑重构的薄记录，标记为二手。

**边界。** 保留二十次运行，每次 4000 事件和 2 MiB，每事件 64 KiB。超过上限写入停止并记录丢了多少，且无论如何写结尾——缺尾的记录不能读成停在那的运行。`hooks::MAX_OUTPUT` 已把最大贡献者限制在每钩子 8 KiB，所以普通 daemon 运行是几 KB。

**journal 由客户端拼接，不由 daemon。** `robotctl update show` 在运行窗口上跑 `journalctl --utc`，限定到运行自己说它触碰的 units，并追加。读系统 journal 是 `updaterd` 拥有且不该通过读侧刻意不加门控的 socket 借出的特权——且它是覆盖*其他* daemon 的那一半，记录从不见。没有 journal 访问的操作员得到记录加跑余下部分的命令。

### 8.4 签名能和不能换来什么：降级和冻结

minisign 签名证明 artifact **来自我们且未被修改**。它不说*何时*发布或是否仍当前。两个攻击在完全有效签名下存活，它们是任何签名 artifact 方案的标准一对：

| | What it is | Status |
|---|---|---|
| **Downgrade / rollback** | Serve an *older, genuinely signed* manifest so robots walk backwards onto a version we withdrew | **Fixed** |
| **Freeze** | Serve the *current* manifest forever, so robots never learn a fix exists | **Open** (§8.4.2) |

两者都能被控制机器人获取什么的任何人触达：过时或被回退的 CDN/镜像、缓存代理、DNS 拦截或敌对本地网络。都不需要被盗密钥。

#### 8.4.1 降级 —— 已关闭

当目标是"无论最新是什么"时，`updaterd` 拒绝比已安装旧的候选：

- `Target::Latest` → 以 `WOULD_DOWNGRADE` 拒绝。`check` 把它报告为不兼容而非作为可用更新提供。
- `Target::Exact` → **允许**。那是定向回退如何工作，且是镜像不能诱发的刻意操作员动作。
- `rollback` 和 `reset-to-golden` 按设计向后移动并使用已安装的 release，所以它们根本不查 manifest。

不对称是要点：守卫挡住*攻击者*能造成的，同时留下*操作员*能选择的。

#### 8.4.2 冻结 —— 开放，需要决定

目前没有东西区分"没有更新"和"有人阻止我看到更新"。被无限期钉在脆弱版本的机器人是现实危害——它已在跑的同一版本，所以没有完整性检查触发。

选项，从最便宜起：

1. **签名 manifest 过期。** 给 manifest 加 `not_valid_after`（时间戳）；拒绝比它旧的 manifest 并在 app 中提示"更新元数据过时"。代价：发布变成时间绑定——CI 必须按计划重新签名 `stable` manifest 即使没有 release 变化，否则每台机器人开始警告。也依赖机器人时钟，那正是 §7.2 时钟检查要不信任的。
2. **单调 manifest 计数器。** 一个只增的 `sequence`；机器人记录见过的最高并拒绝任何更低的。无时钟依赖稳健地检测元数据*回退*，但**不**检测冻结（重放的当前 manifest 有预期的 sequence）。
3. **报告过时而非强制执行。** 记录"上次成功 manifest 获取"并在 `status`/app 中提示："上次检查 47 天前"。密码学上检测不到什么，但让被冻结的机器人对其主人和支持*可见*。无发布负担，无时钟依赖。
4. **电话回**（§17）。报告的机器人让*我们*看到哪些机器人停止检查——唯一集中而非逐机器人检测冻结的选项。需要后端。

**建议：现在 (3)，有发布节奏可挂时 (1)。** 过时报告几乎免费，无自己的失败模式，把静默攻击转成可见攻击。过期是真正的防御但其运营成本——若错过会警告整个机群的重签计划——在发布管线常规化之前不值得付。(2) 便宜但解决 §8.4.1 已在版本层覆盖的问题。

**v1 明确接受：** 网络敌对的机器人可被阻止更新。它不能被*降级*、安装我们没签名的 artifact、或安装健康门控失败的 artifact。那些是我们真正依赖的性质。

## 9. 安装后钩子

与 dpkg `postinst` 同思路，但钩子发货在**（签名的）tarball 内部**，所以无符号代码从不运行。Rust 二进制或 shell 脚本——引擎只是 `exec` 它。

**契约**
- 在符号链接交换*之后*、`apply` *之前*运行。（`pre_install` 钩子如果存在，在交换之前运行。）
- 安装前钩子比安装后钩子有更长上限——十分钟对两分钟。它是安装 release 需要而板子可能没有的东西的钩子（ONNX Runtime；`mediad` 的 GStreamer 栈，在从未有过的板子上约 100 MB apt），且它能花几分钟正因为什么都没交换：旧 release 仍活动并服务，所以长安装前是慢更新而非状态不明的机器人。
- 非零退出 ⇒ 失败更新 ⇒ 回滚。
- 提供的环境（缺失值被**省略**，不设空，所以钩子能区分"首次安装"和"未知"）：
  - `UPDATE_COMPONENT` / `UPDATE_CHANNEL` —— 组件名；两者都设，同值
  - `UPDATE_NEW_VERSION`，以及有前一个 release 时的 `UPDATE_OLD_VERSION`
  - `UPDATE_RELEASE_DIR` —— 正在安装的 release，`<install_dir>/releases/<ver>/`。**这通常是迁移想要的**，且它也是钩子的工作目录。
  - `UPDATE_INSTALL_DIR` —— 组件*根*，例如 `/opt/robot/daemon`
  - `UPDATE_NEW_SCHEMA_VERSION`，以及已知时的 `UPDATE_OLD_SCHEMA_VERSION`
- 钩子以**清空的环境**加固定 `PATH` 运行，所以行为不取决于 systemd 如何调用 `updaterd`。
- 输出被捕获并截断，无论如何到达更新日志：失败时在错误内，成功时逐行（`updater/src/hooks.rs`）。成功时记录因为安装前钩子的报告——哪些插件、哪个运行时、这块板有没有 NPU——是"这台机器人在这个 release 实际能做什么"的答案，而 `journalctl -u updaterd` 曾在钩子明明跑了的板子上 grep 出空。
- 典型用途：安装 release 需要而板子可能没有的东西（§9.1）、配置 schema 迁移、`udev`/权限调整、跨 `schema_version` 升级的数据转换。

### 9.1 如果全新安装做了它，钩子做它

**release 直到它需要的一切都在板上才算安装。** 把文件发货进 release 目录不是安装它；把脚本发货进 release 目录不是运行它。钩子是唯一在每次更新每块板上运行的东西，所以"这块板在这个 release 工作前必须有 X"属于它——不在 X 存在前跑过一次的 provisioning 脚本里，不在人的记忆里。

**问题不是 release 是否需要它。是全新安装是否做它。** "需要"是判断，且它已经放过一个案例：把机器人名字放进 shell 提示符的片段不是 release *需要*的，所以它进了 `install.sh` 且没去别处，只更新过的板子没有它。改问机械问题——**`scripts/install.sh` 写这个文件或运行这个命令吗？**——因为那个可以通过 grep 单个文件回答，且因为不是全新的板子得到恰好钩子做的东西而没有别的。

这现在已经以四种不同形态搞错四次了：

1. **Units。** 加了 daemon 的 release 把它的 `.service` 放进 artifact 且没放到 systemd 看的地方。`btd` 在 release 完整正确的板子上以 `203/EXEC` 失败，且 `on_apply` 不能重启还不存在的 unit。`docs/project/install-path-gap.md` 是记述；`hooks/postinstall` 是修复。
2. **GStreamer 栈和 3A 引擎。** Provisioning 安装了它们，所以在它们存在前 provisioned 的板子没有，插件比 release 构建针对的旧的板子也没有。`hooks/preinstall` 在每次更新上运行 release 自己的 `setup-gstreamer.sh` 和 `setup-rkaiq.sh`，这是堵住它的。
3. **NPU。** 加鸭子检测器的分支写了 `setup-npu.sh`，把它打包进 release 放在模型旁——且从不调用它。每块板都会发货一个到不了 NPU 的检测器直到有人 SSH 进去，而你发现的方式是 `rknn_init` 返回一个数字。
4. **登录 shell。** `install.sh` 加了一个 `/etc/profile.d` 片段把机器人名字放在主机名旁——`microduck@radxa-zero3 (coincoin):~$`——这样到三只鸭子的三个 ssh 窗口不是三个相同提示符。没别的东西安装它，且 `install.sh` 不打包进 release，所以钩子*不能*。在它被写之前 provisioned 的板子接受了此后每次更新却从没得到它；你发现的方式是在 release 包含该特性的板子上 `ls: cannot access '/etc/profile.d/robot-name-prompt.sh'`。

形态每次都一样：工作*做了*，让工作到达板子的东西被漏掉了。它在 review 中不可见，因为加脚本的 diff 看起来完整。

第四个加了一种它能看起来*对*的方式。它在 `install.sh` 中的两个邻居——登录横幅和 `robotctl` 补全——有同样的缺口且更老，所以周围代码读起来像要遵循的模式。连续三个只有全新安装运行的函数不是先例；是三个实例。

**推论。**

- **`install.sh` 对板子做的任何事，钩子也做。** 其他三个是这一个的实例，且它的方向是错误实际发生的方向：一个步骤被加到 `install.sh`，在那里容易在反正要 provisioned 的板子上测试，更新路径永远不知道。`install.sh` 不在 artifact 里且钩子不能调用它，所以共享步骤属于两者都跑的 `scripts/setup-*.sh`——那正是 `setup-gstreamer.sh` 和 `setup-login.sh`。`every_install_sh_step_reaches_an_updated_board` 读 `install.sh` 的调用点并在每个要么也由钩子执行要么写下来（带理由，作为只有全新安装做的东西）时失败。它是强制函数而非证明——逃生口是表中的一行——但四个实例都得为那行争论，且没有一个能。
- **钩子步骤必须幂等，且没事做时必须便宜。** 第一个因为它在每次更新每块板上运行而非一次，所以"已做"是正常情况不是例外。第二个因为代价在每次更新上永远付：在已经什么都有的板子上花十秒的步骤是给每个未来 release 加十秒，且钩子阶段每功能长一步而预算不动。预算是安装前 600 秒（`UPDATE_MAX_SILENCE_SECONDS`，与每个客户端的契约——是 apply 可能静默的最长时间）和安装后 120 秒，与 units、账户和重启共享。能用的形态是戳记：`setup-npu.sh` 把运行时版本写到 `/usr/lib/librknnrt.version` 并比较，`setup-gstreamer.sh` 比较一个戳记和两次 `dpkg -s` 调用，两者在已设置的板子上做不可测量的事。
- **钩子跑的脚本必须被打包。** `every_script_the_hooks_run_is_packaged` 从两个钩子读 `script=scripts/…` 赋值并在任何打包点漏一个时使构建失败。
- **脚本需要的东西必须跟它走。** `setup-rkaiq.sh` 从旁边的 C 文件编译 LD_PRELOAD shim；`setup-npu.sh` 从旁边的 `.dts` 编译设备树覆盖。每个有测试说明，因为失败是静默的：脚本运行、找不到它的源、更新成功只在日志里一个警告。
- **可选硬件从不致命。** 钩子非零失败使更新失败并回滚，所以钩子只能为意味着 release 不可安装的事失败。没有摄像头、没有蓝牙适配器或没有 NPU 的板子仍是机器人；那些安装步骤说明丢了什么、点名重试命令、返回成功。

**这就是 provisioning 和 updating 分离的目的。** `provision.sh` 和 `install.sh` 设置新板子，且它们跑一次。每个在某东西存在前 provisioned 的板子由普通更新修复——这意味着 release 不能假设它的板子何时 provisioned，且只有 provisioning 执行的设置步骤是半个机群永远得不到的步骤。

## 10. 可复用、配置驱动的引擎

引擎跨机器人相同；每个机器人发货声明其组件的配置。适配新机器人 = 新配置 + 新签名密钥 +（也许）新健康探测。

权威的、解析测试过的例子是 [`updater/updater.example.toml`](../../updater/updater.example.toml)——一个单元测试解析它，所以它不能从代码漂移。此处节略：

```toml
# /etc/robot/updater.toml
trusted_keys_dir = "/etc/robot/trusted_keys"   # a *set* of keys (§5.4)
hw_rev           = 1
state_dir        = "/var/lib/robot/updater"    # must be outside every install_dir

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
units  = ["robotd", "mediad"]                  # never updaterd or btd — see §4

[component.daemon.health]
probe   = "socket"
path    = "/run/robotd.sock"
timeout = "30s"

# One component per model — each versions independently (§5.5).
[component.model-walk]
install_dir   = "/opt/robot/model/walk"
keep_previous = 3

[component.model-walk.source]
type     = "hf_hub"
repo     = "ORG/gait-walk"
revision = "main"

[component.model-walk.on_apply]
action = "reload"                              # never drops motor control
unit   = "robotd"
signal = "SIGHUP"
```

配置在加载时验证且 `deny_unknown_fields` 开，所以拼写错误大声失败而非静默禁用它本要改的设置。值得注意的拒绝：`state_dir` 在 `install_dir` 内（交换会销毁更新日志）、两个组件共享 `install_dir`、相对路径、以及无 golden 的 `keep_previous = 0`（无回滚目标）。

注意模型用 `reload`（SIGHUP → 重新 mmap 权重）而非 `restart`，所以模型交换从不丢掉电机控制。逐组件 `on_apply` 是让这自然的原因，也是两个通道保持独立的核心原因。

跨机器人共享：引擎、minisign 管线、手机/BLE 触发协议、回滚逻辑。逐机器人：配置文件、签名密钥、健康探测、钩子。它是小二进制 + schema，不是框架。

## 11. OS 更新（刻意不在 OTA 范围）

跳过 A/B 有一个真实约束：我们不 OTA 内核/系统库。缓解：
- 启用范围仅限 Debian **安全**口袋的 `unattended-upgrades`，用于 CVE 补丁。一劳永逸。
- 任何更大的（内核升级、daemon 需要的新系统库）= **服务时重刷**。
- 最小化对基础 OS 的运行时依赖：近似静态链接、能打包进 tarball 的打包，所以 stock OS 从不阻塞 daemon 更新。

### 11.1 外设 / MCU 固件 —— 不在范围

子板和舵机固件在**生产时**刷写，不在现场。不是 OTA 组件；未做准备。（如果将来变了，它会是一个 `on_apply` 通过电机总线刷写的组件——实质更危险，因为回滚意味着重刷而非符号链接交换。）

## 12. 考虑的替代：apt / dpkg（自有签名仓库）

仍值得放在桌上，特别是 Debian 上的 **daemon** 通道：

**优点**
- 签名（GPG）、原子安装、版本化和**维护者脚本**（`preinst`/`postinst`/`prerm`/`postrm`）——我们想要的安装后钩子——全都免费且经过实战检验。
- 标准 Debian 工具；容易推理；`apt` 处理依赖。

**对我们情况的缺点**
- 开箱即用无健康门控回滚和无干净"回滚到前一版本"；我们反正得在上面构建。
- 把 ML 模型作为普通版本化 asset 发货别扭。
- 不带手机触发 / 进度 IPC——仍需包装。
- **可移植性：** 未来非 Debian 机器人（Yocto 等）上不存在，违背"跨机器人通用工具"目标。

**apt 何时合理：** 如果我们决定机群在可预见未来仅 Debian 且愿意放弃自动健康门控回滚（或通过 pinning `apt install pkg=old` 驱动回滚），自有签名 apt 仓库对 daemon 二进制是合理更简单的路径。模型仍想要侧通道。建议：先用自定义 Rust 引擎以获可移植性 + 回滚；如果 Rust 引擎的维护成本超过其收益且机群保持仅 Debian，重审 apt。

## 13. 手机 / BLE 触发协议（草图）

`btd` 每通道暴露两个 GATT 特征（或一个带通道字段的）：
- **Trigger**（写）：`{ "cmd": "check" | "apply", "channel": "daemon", "version"?: "…" }`
- **Status**（读/通知）：`{ "channel", "state": idle|checking|downloading|applying|healthy|failed|rolled_back, "progress": 0-100, "version", "error"? }`

`btd` 通过 unix socket 转发到 `updaterd` 并把它的状态镜像回去。Payload 从不经过 BLE。

## 14. 实现草图

`updaterd`，小 Rust 二进制（~几百行逻辑）：
- `reqwest` —— 下载（带续传/重试）+ GitHub API 做最新 tag 查找。
- `minisign-verify` —— 签名验证。
- `tar` + `zstd` —— 解压。
- `serde` / `toml` —— 配置；`serde_json` —— manifest。
- 通过 `rename(2)` 原子符号链接交换；通过现有 unix-socket IPC 健康轮询。
- systemd 集成：`Type=notify`、`WatchdogSec`、unit 模板。

CI（每通道）：
1. 构建 artifact（可选通过 `cargo-dist`）。
2. `minisign -S` artifact 和 manifest。
3. 创建 GitHub Release，tag `daemon-vX` / `model-vX`，带 artifact、`.minisig` 和 `manifest.json` 作为 assets。

## 15. 好邻居（范围内）

能塞进同一 IPC / 引擎且有回报的廉价添加：

- **可复用健康自检模块。** 一个探测（电机 ack、模型加载、媒体初始化）由两个调用者调用：启动时和更新后作为健康门控。"机器人 OK 吗"的唯一真相来源。
- **`robotctl` 本地 CLI。** 通过 `updaterd` unix socket 的薄客户端——与 `btd` 同角色，不同传输；它不持有自己的更新逻辑。用于支持、现场恢复和 CI。
  命令有命名空间（`robotctl update …`）所以以后命名空间是附加的：`check`、`apply [--version|--dry-run]`、`rollback`、`reset-to-golden`、`select`（切换模型 bundle 的方式）、`pin`、`status`、`log`、`watch`。
  仅实现了 `update` 命名空间。
- **Dev 侧载路径。** 接受用独立 **dev 密钥**签名的 artifact（在受信任集中但由标志 / dev build 门控），让团队能刷本地构建而不碰生产签名（见 §5.4）。
  **已构建。** `robotctl update apply --from <dir>` 为一次调用覆盖源，所以 release 从板上目录来而 apply 的其他一切不变——preflight、签名、哈希、兼容性、健康门控、自动回滚。`scripts/dev-push.sh` 是整个路径：交叉编译、打包、签名、复制、apply。它*不*做两件事，都刻意：它不是 `updaterd install --from`，那强制 `on_apply` 和门控关因此不能用在活动 release 上；且它不放松验证，这是为什么它需要板上的 dev 密钥而非跳过检查的标志。降级守卫为 `--from` 让路与它为 `--version` 让路同理——点名目录的操作员不是倒退了的镜像，且本地构建是排在板子正在跑的任何东西之下的预发布，所以守卫它会拒绝每次推送。
  目录上的一个约束，是 systemd 的而非引擎的：`updaterd.service` 设 `PrivateTmp=yes`，给 unit 自己的 `/tmp` **和**自己的 `/var/tmp`，所以从 shell 复制到任一个的 release 不是 daemon 读的那个。这是为什么侧载目录住在板子用户家里以及为什么有 `Check::SideloadDir`——没有它失败是目录里 `ls` 列得出但 manifest 缺失，从外部不可证伪。
- **BLE 配置安全。** 相邻但重要：设置期间 wifi 凭证通过 BLE 传。那个特征必须配对 + 加密，否则是凭证泄露。更新 artifact 被签名所以伪造的*触发*低风险，但*配置*写入必须认证。确认 `btd` 已强制此。

## 16. Release 测试与信心

**动机。** 在上一个机器人上，release 由手动验证——手动回退到已知版本、手动应用新的、手动检查。人为错误空间太大。这里的设计目标：**让更新路径可脚本化且自动测试，使发货给客户是*决定*，不是手动流程。**

关键框架：那个人为错误表面（回退 → apply → 验证）的大部分住在更新*机制*里，那是纯软件且可在 CI 中用**无硬件**测试的。只有真正硬件相关的行为需要真实板子。

### 16.1 为可测试性设计

- **针对测试 artifact 的真实代码路径。** `updaterd` 从配置读它的源/manifest，所以测试把它指向本地目录或 `staging` 通道并驱动*精确的生产代码*——没有与现实漂移的 mock updater。
- **可脚本化、幂等的原语**（通过 `robotctl`，§15）：`apply --version X`、`rollback`、`pin <version>` / `unpin`、`model select`、**`reset-to-golden`**。之前的痛苦是缺少干净的"到恰好这个版本"命令——这些就是它。全部**非交互**：无提示，没什么让操作员答错。
- **`--dry-run`：** 跑 fetch / verify / compat / 空间检查并在交换前停止。
- **故障注入钩子：** 强制失败健康探测、损坏 artifact、非零钩子或交换中途 kill 的标志——所以回滚是*被测试*的，不是假设的。

### 16.2 两个测试层级

**Tier 1 —— 机制测试（CI，无硬件，快）。** 每个 PR 跑；仅此就移除大部分手动回退风险。自动覆盖：

- 签名有效 / 无效 / 错密钥 → 接受 / 拒绝
- 哈希不匹配 → 拒绝
- `min_hw_rev` / `model_api` 不兼容 → 用清晰状态拒绝
- 作为"最新"提供的旧但有效签名的 manifest → 作为降级拒绝（§8.4.1），而显式 `--version` 降级仍允许
- `rollback` 从不落到刚失败的 release，也不落到日志记录为已回滚的 release
- 启动计数器 trial 是逐组件的：一个组件的转换不能消耗另一个的预算
- **机器人特定状态在更新*和*回滚中保留**：校准、生成资产、学习状态、用户偏好都完好（§5.7）
- 健康更新 → 提交 + 修剪
- 失败健康探测 → 自动回滚；机器人在*旧*版本上健康
- 钩子非零退出 → 回滚
- **交换中途 kill -9 / 模拟断电** → 重启时状态一致，从无半活动 release
- 磁盘满 → 任何变更前中止
- 并发触发 → single-flight；第二个报告 busy
- **`robotd` 不存在 / 崩溃循环** → `btd` 和 `updaterd` 仍服务；更新、回滚和 reset-to-golden 都完成（§4.1）。包括一个"从故意坏的 release 恢复"测试——它演练最重要的路径。
- `robotd` IPC 挂起（socket 打开，无回复）→ 超时，不是停滞
- 版本矩阵：升级 A→B、跳过 A→C、回滚 B→A、跨 `schema_version` 升级迁移

Tier 1 用进程内实现 `RobotClient` 的 `FakeRobot` 驱动引擎。那是测试引擎*决策*的正确方式——fake 可以按需不健康、挂起或不存在，这些都不容易为真实 staging。但它从不序列化任何东西，所以它根本看不到线。

**Tier 1b —— 针对真实 `robotd` 进程**（`robotd/tests/updater_gate.rs`，仍 CI，仍无硬件）关上它。它派生真实二进制并让真实 `SocketRobotClient` 通过真实 unix socket 与它对话：引擎调用的每个方法必须以它解析的形态回来，更新必须门控并提交，且 `robotd --unhealthy` 必须回退 `current` 背后的*内容*，不仅是符号链接。

没别的覆盖这个。协议 crate 自己的往返测试不能检测偏差，因为两边共享结构体；`robotd` 中 `dispatch` 级的单元测试能廉价抓到改变的回复形态，但只有真实进程演练 socket 本身。

**Tier 2 —— 设备上验收（canary 机器人）。** 硬件相关行为（步态、电机、媒体）需要真实板子。保持几个跟踪 `staging`、对每个候选自动更新、跑脚本化验收测试（电机扫、步态冒烟测试、媒体初始化、健康自检）并报告通过/失败 + 更新日志的**实验室/canary 机器人**。可重复因为 reset-to-golden 和 `apply --version` 可脚本化。

### 16.3 通道与晋升

**已实现**为三个 GitHub Actions 工作流加一个 `cargo xtask` 发布器（`.github/workflows/`、`xtask/`）：

| | |
|---|---|
| `ci.yml` | fmt, clippy, tests, plus `board-test.sh` — the only job that proves the binaries run on aarch64 Linux |
| `release.yml` | on a `daemon-staging-v*` tag: cross-build, package, sign, **verify with the robot's own code path**, publish a prerelease |
| `promote.yml` | manual: re-sign a *stable* manifest, copy the validated artifact onto the stable release, retire staging |

发布器是 Rust `xtask` 而非 shell 脚本，原因只有一个：它复用 updater 验证用的完全相同的 `minisign`、`tar`、`zstd` 和 `sha2` crate。shell 版本会依赖单独安装的二进制，其行为可能与机器人接受的漂移——差异最不该能藏的地方。它也链接*完整* `minisign` crate（能签名）而 daemon 只链接 `minisign-verify`（不能）。

三个值得陈述的性质，因为每个是被断言而非假设的：

- **晋升从不重新构建。** stable manifest 携带 staging `sha256`，晋升在检查该摘要后把 staging artifact 复制到 stable release，所以客户端收到的字节是 canary 验证过的字节；测试断言摘要不变。

  manifest 曾指回 *staging* release 而非复制，理由是一组字节不会发散而两组会。那忽略的是机器人在安装前验证 `sha256`，所以发散的复制永远不能静默安装——而 artifact 住在像脚手架命名的 tag 下的 stable 通道是离一次清理就坏的。它坏了：删除 `daemon-staging-v0.1.x` release 留下 `daemon-v0.1.0`、`v0.1.1` 和 `v0.1.4` 正确签名且指向 404。Stable release 现在自包含，staging 由 `promote.yml` 自己退役。
- **Artifact 可复现。** tar 中固定 mtime 意味着相同输入产生相同归档，所以重建可与已发货的比较；测试断言相同输入的两个包哈希相同。
- **`release.yml` 发布前验证。** 它通过真实引擎安装 release（`updaterd install --from`，通过 `LocalDir` 源）并断言二进制落地为可执行。如果 updater 不能接受 release，没人能下载它。

两个守卫存在因为两个错误都容易：`package` 拒绝不匹配 `Cargo.toml` 的 `--version`（打 tag 不 bump），`promote` 拒绝不匹配 staging manifest 的版本。

- 通道：`staging` → `stable`。CI 发布候选到 `staging`。
- Canary 机器人用**每个命令一个标志**拿候选：

  ```
  sudo robotctl update apply daemon --staging
  ```

  早期草稿说 canary"自动拉 staging"，那按写的不可实现：`newest_version` 跳过 GitHub 标为预发布的任何东西*和*携带 semver 预发布组件的任何东西。那个过滤器正是让客户机器人远离候选构建的，所以它保留，`--staging` 是它唯一的选择加入——在 `staging_tag_prefix` 下的第二次扫描，允许 *GitHub* 标志同时仍排除 semver 预发布，所以分支构建永远不能被误认为候选。`latest_manifest` 不动，所以 `auto_apply` 和定期检查继续解析 stable：没有人和 root 没有东西漂移到候选上。

  第一次尝试是通过编辑 `tag_prefix` 把板子指向 staging，它对就坐在那的候选报告 `no releases in … with tag prefix "daemon-staging-v"`——前缀说去哪找，预发布过滤器拒绝看。

  **`--staging` 在通道落后于板子时拒绝。** 直接晋升到 stable 的 release 不发布候选——支持的路径，也是 `release.yml` 标为"NOT canaried"的那个——所以 staging 扫描继续用最后发布候选的版本应答。在已超过那个版本的板子上，`--staging` 曾安装它：验证、交换，然后在旧 release 不含的 daemon 启动失败时回退。回滚是对的且没说原因，所以解析出的候选现在与已安装的比较，拒绝点名两个版本。`--staging --version X` 不加门控且是绕过方式——点名候选的操作员在陈述意图，那也是板子刚从其回滚的那个被重新安装的方式。
- 绿灯时，**晋升**：把 `stable` 重新指向已验证的*相同字节*——重新签名 `stable` manifest 引用相同 tarball + 哈希。无重建、无重刷、无手工复制文件。晋升是一个命令 / 一次点击，发货恰好测试过的东西。
- 坏的 `stable` → 反向同样操作：把 `stable` 重新指向前一个已知良好的 manifest；`min_supported`（§8.1）然后在修复落地后把机器人向前拉。

### 16.4 溯源

`version.toml` 和更新日志记录每个 artifact 的 git SHA + 哈希，所以"客户端正在运行的确切版本"在实验室中总是可复现。

这关上上次伤人的循环：**人决定*是否*晋升；机器每次完全相同地做回退 / apply / 验证。**

### 16.5 引导状态 —— 健康门控结束了，payload 还没

`daemon` artifact 仍比最终少发货：`bin/updaterd`、`bin/robotctl`、`bin/robotd`、`version.toml`。还没有 `mediad` 或 `btd`。

那不循环：§4.1 确立了 `updaterd` 和 `btd` 发货在 daemon artifact *内部*，所以用 updater 更新 updater 是真实的最终流程——且最风险的那个。先演练它是 §9 构建顺序的要点。

`updater.example.toml` 中的两个设置在有东西可门控之前刻意无效。两者现在都生效了：

| | was | now | still pending |
|---|---|---|---|
| `on_apply` | `none` | `restart` with `["robotd", "configd"]` | — |
| `health` | `none` | `socket`, 30s | timeout is a guess until M4 measures a real boot |

`mediad` 在旧权威列表下需要一个条目现在不需要：重启集从 release 发货的 units 派生，跳过任何没有 `[Install]` 段的，而 `mediad.service` 有一个。所以 apply 重启它，从没跑过它的机器人在下次安装时启用它——规则是 unit 文件，不是任何人维护的名字。

**`health = none` 是设计最弱的时刻**：它在交换成功时立即提交，所以根本没有自动回滚，启动计数器是唯一剩余的恢复机制。离开那个状态是 M1 的目的。

把示例配置钉到无效值的单元测试现在断言相反：`on_apply` 必须重启 `robotd` 且必须**不**重启 `updaterd` 或 `btd`（§4.1），且 `health` 必须是 socket 探测。回归到 `probe = "none"` 会静默禁用自动回滚却看起来像一个词的 diff，那正是需要测试挡在前面的那种变更。

**仍未测试的：** `on_apply` 中的 `systemctl restart` 从未对真实 systemd 运行过——开发笔记本上没有，stub 它会测试 stub。健康门控本身*是*针对真实 `robotd` 进程通过真实 socket 测试的（`robotd/tests/updater_gate.rs`），所以仍未证明的具体是重启步骤和 30 秒超时。两者在 M4 在 Radxa 上落地。

## 17. 开放问题 / 未来

- **配置归属。** 配置目前跟模型 bundle 走。有没有配置该属于 daemon 通道，或成为自己的小通道？schema 迁移无论如何由钩子处理（§9）。
- **最小"成功了吗"电话回。** 机群管理推迟，但每次更新一次成功/失败 ping 会让我们早抓到坏 release 且是让 `min_supported`（§8.1）在实践中可操作的东西。值得吗？
- 分阶段推出 / 遥测 / 仪表盘：明确推迟；机群增长时重审。
- Delta 更新：推迟；当前节奏下 artifact 小。

本文档与实现之间已知的差距，刻意开放：

- **不支持一个模型槽内的竞争替代。** 每个模型是有一个版本线的组件（§5.5），覆盖"走、跳、站各自独立更新"。它*不*覆盖"两个不同走模型已安装，用户选一个"。那需要 `(name, version)` 存储键，波及约 14 个文件——`Store` 的键控方法、`known_bad`、`PendingUpdate`、`Pins`、`golden`、`pinned` 和线类型（所以 `API_VERSION` 升级）。它也把四件事从全局变成逐 bundle：修剪计数、回滚目标、"最新"和 golden。推迟到已知需要；猜那四个是被删的 `Layout::Library` 如何产生的。
- **无法发现可安装模型。** App 能列*已安装*的，但没有 `Source` 能回答"可安装什么"——今天加模型意味着编辑配置。想要 `Source::list_*` 操作。
- **无恢复模式。** §8.2 的链是 `current → previous → golden`，三者都实现了包括越过缺失或已知坏 previous 的升级。最后的"仍能重新获取的最小恢复模式"不存在。
- **一个 `RobotClient` 服务每个组件。** `HealthCheck::Socket { path }` 对*跑哪个探测*被遵守，但 socket 路径本身来自 `main` 构造的任何东西。两个组件都探测 `robotd` 时没问题；如果某个组件将来需要不同对端是陷阱。想要逐组件客户端。

仍开放：
- **手机交付 artifact 作为回退**——给没有可用互联网的机器人：从未配置、强制门户、CDN 被封、离线演示点。引擎侧几乎免费，因为 `LocalDir`（§15、§16.1）已经通过真实验证路径应用一个 manifest + artifact + 两个 `.minisign` 的目录，且签名让交付传输按构造不可信。代价全在链路上，必须是 wifi 或 USB（§3.1）。两种形态，解决不同问题：机器人托管 AP 让手机加入（不需要机器人凭据，但手机必须在加入*前*通过蜂窝获取 release，且两个移动 OS 都抗拒无互联网的网络），或——wifi 能用但 CDN 不能时——手机通过 LAN 推送，便宜得多但加了一个面向网络的摄取监听器。那个监听器的爆炸半径受签名验证限制（未签名上传不能安装；最坏是填满磁盘），但它无论如何想要 token 和刻意绑定（`architecture.md` §2.2）。备份计划，不是 v1。
- **Manifest 过时报告**（§8.4.2）——在 `status` 和 app 中提示"上次成功检查 N 天前"，把冻结攻击从静默转成可见。廉价；建议。签名 manifest 过期是真正的防御但带重签计划，推迟到发布常规化。
- **配置归属**——配置跟模型 bundle、daemon，还是成为自己的小通道？（钩子无论如何处理迁移，§9。）
- **最小成功/失败电话回**——每次更新一次 ping 会让我们早抓到坏 release，且是让 `min_supported`（§8.1）在实践中可操作的东西。值得，还是对 v1 太多？
- **行为/大脑层**——如果高层行为层（驱力、情绪、习惯）将来作为自己的 artifact 发货，它成为有自己兼容性约束的第三个通道。它的学习状态也是 §5.7 的材料。

明确**不**做：硬件变体矩阵（§5.6）、外设固件 OTA（§11.1）、分阶段推出 / 遥测、delta 更新。

自初稿以来已定：分离托管（GitHub + HF Hub）；schema 中的 `min_supported` 下限；模型 *bundle 带命名槽* + `model_api` 兼容性；多个受信任签名密钥；单一硬件目标；机器人特定状态保留（§5.7）；配置在文件而非 systemd unit 中。
