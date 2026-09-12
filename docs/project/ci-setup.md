# CI 设置

状态：草稿 · 日期：2026-07-28 · 负责人：pierre

发布流水线的一次性设置。密钥保管见 [`updater-design.md`](../design/updater-design.md) §5.4，staging → stable 模型见 §16.3。

## 决定：两把密钥、两个触发器，且本计划上无闸门

**2026-07-29 决定。** 分支推送用 `team.dev` 签名；打标签的版本与晋升用 `release-1` 签名。两把密钥都在 CI 中。

| 触发器 | 工作流 | 密钥 | 到达客户机器人 |
|---|---|---|---|
| 推送到任何分支 | `dev.yml` | `team.dev`（仓库密钥） | **否** —— 那里 `allow_dev_keys = false`，且受信任文件名必须以 `.dev.pub` 结尾 |
| 标签 `daemon-staging-v*` | `release.yml` | `release-1`（`release` 环境密钥） | 在晋升前不会 —— 作为预发布发布 |
| 手动晋升 | `promote.yml` | `release-1` | **是** |

这种拆分以一种早期提案没有的方式干净：密钥在 staging → stable 路径*内部*从不交叉，因此晋升仍在相同字节上重新签名清单（§16.3）。一个由 `team.dev` 签名的产物无法被晋升，因为 `promote` 把 `sig_url` 指向 staging 产物的现有签名 —— 这就是为什么 dev 构建保持为 dev 构建而非变成发布候选。

### 原本打算什么，以及为什么它不在那里

计划是把 `release-1` 门控在 `release` 环境的 required-reviewers 规则后面。它无法被创建：

```
HTTP 422: Failed to create the environment protection rule.
Please ensure the billing plan supports the required reviewers protection rule.
```

标签保护被作为替代品检查，也不可用：

```
403: Upgrade to GitHub Pro or make this repository public to enable this feature.
```

必需审查者、部署分支策略、分支保护与规则集在一个*私有*仓库上都是 Team/Pro 特性，而 `pollen-robotics` 在免费计划上。`release` 环境存在，零保护规则。

### 已接受的风险，直白陈述

**任何有推送权限的人都能读取 `release-1`。** 把它限定到 `release` 环境阻止了一个不声明该环境的工作流看到它，但任何协作者都能编写一个声明它的工作流。因此"仅用于发布"是已经互相信任的人之间的一个约定，而非访问控制 —— 工作流文件不是边界。

这是被故意接受的：团队小且互信，没有机器人离开过大楼，替代方案（手动签名每个发布）在今天对一个尚不存在的威胁买不到任何东西。

**当以下任一为真时重新审视**，因为代价急剧变化且失败是本设计无法撤销的那种 —— 一个泄露的密钥意味着向每个机器人交付一个 `release-2` 签名的更新，而任何错过它的机器人永远信任被攻破的密钥：

- 一个机器人在某人家中，或
- 某个有推送权限的人不是你会直接把签名密钥交给他的人。

那时的修复是把组织升级到 GitHub Team，这保留这种拆分并加上闸门；或把 `release-1` 签名移回一台笔记本。

## 分层（不变，且仍是限制损害的东西）

无论上面决定什么，限制一次攻破代价的是哪把密钥从哪里可达：

| 密钥 | 在 CI 中 | 角色 |
|---|---|---|
| `release-1` | **目前不** —— 见上 | 签名每个发布与晋升 |
| `release-2` | 否 | 如果 CI 或 `release-1` 被攻破时的第一个轮换目标 |
| `release-3` | 否，理想情况下永远不在联网机器上 | 最后手段 |
| `team.dev` | 预期，仅 dev 工作流 | 分支构建；不能触碰客户机器人，因为那里 `allow_dev_keys` 为 false |

所有**公钥**从一开始就进入每个机器人镜像 —— 一个机器人只能对照烘焙进它的集合验证，因此这是让轮换无需物理重刷即可进行的唯一机会。

## 密钥与变量

GitHub Secrets 是**只写**的：一旦设置，没人 —— 包括你 —— 能读回它们。它们是一个*部署副本*，绝不是存储。密码管理器仍是记录系统；丢失它意味着密钥没了，且每个信任它的机器人永远不能再被签名。

**把它们限定到 `release` 环境，而非仓库。** 一个仓库密钥对仓库中的每个工作流作业可读；一个环境密钥仅对声明该环境的作业可读。在本计划中，这种区别阻止了一个不相关的工作流看到密钥，仅此而已（见上）—— 但它严格更好且不花成本：

```bash
gh secret set MINISIGN_SECRET_KEY --env release < ~/.duck-keys/release-1.key
```

```bash
gh secret set MINISIGN_PASSWORD --env release
```

第二个会提示，因此密码短语永远不会落到 shell 历史或记录中。

**Secrets**（加密，不可读回）。当前状态：

| 名称 | 范围 | 值 | 已设置 |
|---|---|---|---|
| `MINISIGN_SECRET_KEY` | `release` 环境 | `~/.duck-keys/release-1.key`，两行 | ✅ |
| `MINISIGN_PASSWORD` | `release` 环境 | `release-1` 的密码短语 | ✅ |
| `MINISIGN_DEV_SECRET_KEY` | **仓库** | `~/.duck-keys/team.dev.key` | ✅ |

`MINISIGN_DEV_SECRET_KEY` 故意是仓库范围的：每个分支推送都用它签名，因此把它门控在一个环境后面意味着 dev 工作流声明一个为 `release-1` 准备的环境。它不需要密码短语密钥 —— 一个 dev 密钥是未加密的，以便 CI 可以非交互式签名，`xtask keycheck` 确认这一点并称对 dev 密钥正确、对发布密钥错误。

**Variables**（明文，可读 —— 公钥不是秘密）：

| 名称 | 值 |
|---|---|
| `MINISIGN_PUBLIC_KEY` | `~/.duck-keys/release-1.pub` 的密钥行 |

公钥被 `release.yml` 用来在发布前通过机器人自己的代码路径验证一个发布。把它作为一个*变量*而非 secret 是故意的：把公钥当作 secret 会引发关于哪一半是哪一半的混淆。

**不要**添加 `release-2` 或 `release-3`。它们的全部价值就在于缺席于此。

## `release` 环境

`release.yml` 与 `promote.yml` 都声明 `environment: release`。在 Settings → Environments 下创建它并添加**必需审查者**。

没有它，任何能推送 `daemon-staging-v*` 标签的人都能为整个机群签名。有了它，到达签名密钥需要第二个人的批准 —— 这以每次发布一次点击的代价，恢复了本地签名本会给予的大部分东西。

Fork PR 永远不接收 secrets，因此无论如何密钥从贡献者 PR 不可达。

## 密钥在哪里被处理

每个工作流恰好一个步骤把密钥写到磁盘，且立即移除：

```
umask 077
printf '%s' "$MINISIGN_SECRET_KEY" > "$RUNNER_TEMP/secret.key"
cargo run -p xtask -- sign --dir dist --key "$RUNNER_TEMP/secret.key"
shred -u "$RUNNER_TEMP/secret.key" || rm -f "$RUNNER_TEMP/secret.key"
```

写到文件而非作为参数传递，因为命令行上的密钥对运行器上的任何其他东西都在进程列表中可见。

`release.yml` 的验证步骤故意不需要**任何**密钥：`xtask package` 发出第二个带裸文件名 URL 的清单（用于 `LocalDir`），`xtask sign` 一次签名两者。重新签名来验证意味着在一个作业中处理签名密钥两次而无任何好处。

## 切一个版本

**GitHub 发布页面是入口。** 你创建的东西决定发生什么，`release.yml` 读取标签来算出：

| 你创建 | 模式 | CI 做什么 |
|---|---|---|
| 预发布，标签 `daemon-staging-v0.4.0` | `staging` | 为 aarch64 交叉构建、打包、用 `release-1` 签名、通过真实引擎验证、发布一个**预发布** |
| 发布，标签 `daemon-v0.4.0`，staging 0.4.0 存在 | `promote` | 在 staging 验证过的**相同产物字节**上重新签名一个稳定清单；不重建；退役 staging 发布 |
| 发布，标签 `daemon-v0.4.0`，无 staging 0.4.0 | `stable` | 构建 0.4.0 并直接发布到 stable，附注说明它从未被灰度过 |

运行摘要命名三者中哪个运行了，因为当一个发布看起来不对时"是哪一个"是第一个问题。

从终端推送标签做同样的事 —— 发布对象为你创建：

```
git tag daemon-staging-v0.4.0 && git push --tags
```

先提升工作区版本：`xtask package` 拒绝一个与 `Cargo.toml` 不一致的标签。

两个值得知道的属性，因为它们正是拆分的*目的*：

- 一个预发布被普通的 `update apply` 跳过，因此一个未晋升的构建无法到达一个没有用 `--staging` 要求它的机器人。两个发布步骤都在一个已存在的发布上重新断言该标志 —— 否则一个某人草拟时没勾选框的 staging 发布会被整个机群安装，而一个被标记为预发布的 stable 发布会交付给无人。
- 晋升从不重建。stable 发布最终是自包含的（清单、签名、产物、引导二进制），这就是为什么 staging 发布之后可以删除。在那为真之前的 stable 发布 —— `daemon-v0.3.0` —— 仍把它们的 `url` 指向其 staging 发布，因此**那些 staging 发布绝不能删除。**

手动晋升是同一个配方，只是没有先创建一个发布，且是 `min_supported` 所在的地方（§8.1 —— 它强制低于该版本的机器人不等客户端就更新）：

```
gh workflow run promote --field version=0.4.0
```

### 工作流

```
release.yml            入口：决定 staging / promote / stable
_build-release.yml     构建 · 打包 · 签名 · 验证 · 发布        （被调用）
_promote-release.yml   复制字节 · 验证 sha · 重新签名 · 退役 staging（被调用）
promote.yml            workflow_dispatch → _promote-release.yml
dev.yml                每次推送：一个不为客户签名的 dev 构建，`team.dev` 密钥
```

配方位于两个被调用的工作流中，因此 staging 与 stable 路径在交付什么上不会漂移。`xtask` 的打包绊索读取 `_build-release.yml` 与 `dev.yml` 的 `--include` 列表，因此一个未被打包的 unit、hook 或 sysusers 文件会让测试失败，而非机器人。

两个发布步骤都只在发布缺席时创建它，然后用 `--clobber` 上传，因此一个半途失败的作业可以重新运行。以前不是这样：一个在创建发布后死亡的发布作业只能通过手动删除发布与标签来重试。

## 轮换密钥

如果 `release-1` 或 CI 被攻破：

1. 用 `release-2` 的替换 `MINISIGN_SECRET_KEY` / `MINISIGN_PASSWORD`。
2. 发布一个由 `release-2` 签名的版本。机器人已经信任它 —— 这就是为什么两个公钥从第一个镜像就交付了。
3. 在随后的发布中从 `trusted_keys_dir` 移除 `release-1.pub`，以便被攻破的密钥不再被接受。
4. 生成一个替换的第三把密钥以便仍有一个备用：
   `cargo xtask keygen --kind release --name release-4 --out ~/.duck-keys`

步骤 3 故意滞后于步骤 2：在每个机器人都拿到新签名的发布之前撤销旧密钥会让任何错过它的机器人陷入困境。
