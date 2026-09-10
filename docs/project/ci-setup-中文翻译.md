# CI 设置

状态：草稿 · 日期：2026-07-28 · 负责人：pierre

发布流水线的一次性设置。密钥托管见 [`updater-design.md`](../design/updater-design.md) §5.4，staging → stable 模型见 §16.3。

## 决定：两个密钥、两个触发器，且此计划上没有门控

**2026-07-29 决定。** 分支推送用 `team.dev` 签名；打标签的发布和升级用 `release-1` 签名。两个密钥都在 CI 中。

| 触发器 | 工作流 | 密钥 | 到达客户机器人 |
|---|---|---|---|
| 推送到任何分支 | `dev.yml` | `team.dev`（仓库 secret） | **否**——那里 `allow_dev_keys = false`，且受信任的文件名必须以 `.dev.pub` 结尾 |
| 标签 `daemon-staging-v*` | `release.yml` | `release-1`（`release` 环境 secret） | 在升级之前不会——作为预发布发布 |
| 手动升级 | `promote.yml` | `release-1` | **是** |

这种分割以一种早期提议没有的方式干净：密钥在 staging → stable 路径内*从不交叉*，因此升级仍然在相同字节上重新签名清单（§16.3）。由 `team.dev` 签名的制品不能被升级，因为 `promote` 将 `sig_url` 指向 staging 制品的现有签名——这就是为什么开发构建保持为开发构建而不是变成发布候选。

### 原本打算做什么，以及为什么它不在那里

计划是将 `release-1` 门控在 `release` 环境的必需审阅者规则后面。它无法被创建：

```
HTTP 422: Failed to create the environment protection rule.
Please ensure the billing plan supports the required reviewers protection rule.
```

标签保护作为替代方案被检查，也不可用：

```
403: Upgrade to GitHub Pro or make this repository public to enable this feature.
```

必需审阅者、部署分支策略、分支保护和规则集在*私有*仓库上都是 Team/Pro 功能，而 `pollen-robotics` 在免费计划上。`release` 环境存在，零保护规则。

### 已接受的风险，直白陈述

**任何有推送权限的人都可以读取 `release-1`。** 将其范围限定到 `release` 环境阻止了未声明该环境的工作流看到它，但任何协作者都可以编写一个声明它的工作流。因此"仅用于发布"是已经互相信任的人之间的约定，而不是访问控制——工作流文件不是边界。

这是故意接受的：团队很小且互相信任，没有机器人离开过大楼，而替代方案（手动签名每个发布）在今天对一个尚不存在的威胁没有任何收益。

**当以下任一情况变为真时重新审视**，因为成本会急剧变化且失败是这个设计无法撤销的——泄露的密钥意味着向每台机器人发送 `release-2` 签名的更新，而任何错过它的机器人将永远信任被攻破的密钥：

- 一台机器人在某人家中，或
- 有推送权限的人不是你会直接把签名密钥交给他的人。

那时的修复是将组织升级到 GitHub Team，这保留了这种分割并添加了门控；或者将 `release-1` 签名移回笔记本。

## 分层（未变，且仍然是限制损害的东西）

无论上面决定了什么，限制被攻破成本的是哪个密钥可以从哪里到达：

| 密钥 | 在 CI 中 | 角色 |
|---|---|---|
| `release-1` | **目前不在**——见上文 | 签名每个发布和升级 |
| `release-2` | 否 | 如果 CI 或 `release-1` 被攻破，第一个轮换目标 |
| `release-3` | 否，理想情况下永远不在联网机器上 | 最后手段 |
| `team.dev` | 预期，仅开发工作流 | 分支构建；不能触及客户机器人，因为那里 `allow_dev_keys` 为 false |

所有**公钥**从一开始就进入每个机器人镜像——机器人只能对照烘焙进它的集合进行验证，因此这是使轮换无需物理重刷即可成为可能的唯一机会。

## Secrets 和变量

GitHub Secrets 是**只写的**：一旦设置，没有人——包括你——可以读回它们。它们是*部署副本*，永远不是存储。密码管理器仍然是记录系统；丢失它意味着密钥消失，每台信任它的机器人将永远无法再被签名。

**将它们范围限定到 `release` 环境，而不是仓库。** 仓库 secret 可以被仓库中的每个工作流作业读取；环境 secret 只能被声明该环境的作业读取。在此计划上，这种差异阻止了不相关的工作流看到密钥，仅此而已（见上文）——但它严格更好且不花成本：

```bash
gh secret set MINISIGN_SECRET_KEY --env release < ~/.duck-keys/release-1.key
```

```bash
gh secret set MINISIGN_PASSWORD --env release
```

第二个会提示，因此密码永远不会落在 shell 历史或记录中。

**Secrets**（加密，不可读回）。当前状态：

| 名称 | 范围 | 值 | 已设置 |
|---|---|---|---|
| `MINISIGN_SECRET_KEY` | `release` 环境 | `~/.duck-keys/release-1.key`，两行 | ✅ |
| `MINISIGN_PASSWORD` | `release` 环境 | `release-1` 的密码 | ✅ |
| `MINISIGN_DEV_SECRET_KEY` | **仓库** | `~/.duck-keys/team.dev.key` | ✅ |

`MINISIGN_DEV_SECRET_KEY` 故意是仓库范围的：每次分支推送都用它签名，因此将其门控在环境后面意味着开发工作流声明一个为 `release-1` 准备的环境。它不需要密码 secret——开发密钥是未加密的，因此 CI 可以非交互式签名，`xtask keycheck` 确认这一点并称对开发密钥正确、对发布密钥错误。

**变量**（明文，可读——公钥不是 secret）：

| 名称 | 值 |
|---|---|
| `MINISIGN_PUBLIC_KEY` | `~/.duck-keys/release-1.pub` 的密钥行 |

公钥被 `release.yml` 用来在发布之前通过机器人自己的代码路径验证发布。将其保持为*变量*而非 secret 是故意的：将公钥当作 secret 会引起关于哪一半是哪一半的混淆。

**不要**添加 `release-2` 或 `release-3`。它们的全部价值在于不在这里。

## `release` 环境

`release.yml` 和 `promote.yml` 都声明 `environment: release`。在 Settings → Environments 下创建它并添加**必需审阅者**。

没有它，任何能推送 `daemon-staging-v*` 标签的人都可以为整个机群签名。有了它，到达签名密钥需要第二个人的批准——这恢复了本地签名本可以给出的大部分东西，代价是每个发布一次点击。

Fork 的拉取请求永远不会收到 secrets，因此无论如何密钥都无法从贡献者 PR 到达。

## 密钥在哪里被处理

每个工作流恰好有一个步骤将密钥写入磁盘，并且立即被移除：

```
umask 077
printf '%s' "$MINISIGN_SECRET_KEY" > "$RUNNER_TEMP/secret.key"
cargo run -p xtask -- sign --dir dist --key "$RUNNER_TEMP/secret.key"
shred -u "$RUNNER_TEMP/secret.key" || rm -f "$RUNNER_TEMP/secret.key"
```

写入文件而不是作为参数传递，因为命令行上的密钥对运行器上的任何其他东西在进程列表中可见。

`release.yml` 的验证步骤故意**不需要**密钥：`xtask package` 发出第二个带有裸文件名 URL 的清单（用于 `LocalDir`），`xtask sign` 一次签名两者。重新签名来验证意味着在一个作业中处理签名密钥两次而没有任何收益。

## 切一个发布

**GitHub releases 页面是入口。** 你创建的东西决定了会发生什么，`release.yml` 读取标签来算出来：

| 你创建 | 模式 | CI 做什么 |
|---|---|---|
| 预发布，标签 `daemon-staging-v0.4.0` | `staging` | 为 aarch64 交叉构建、打包、用 `release-1` 签名、通过真实引擎验证、发布一个**预发布** |
| 发布，标签 `daemon-v0.4.0`，staging 0.4.0 存在 | `promote` | 在**相同的制品字节**上重新签名稳定清单，staging 已验证；不重新构建；退役 staging 发布 |
| 发布，标签 `daemon-v0.4.0`，没有 staging 0.4.0 | `stable` | 构建 0.4.0 并直接发布到 stable，说明它从未被金丝雀测试过 |

运行摘要命名了三个中的哪一个运行了，因为当发布看起来不对时，"是哪一个"是第一个问题。

从终端推送标签做同样的事——发布对象为你创建：

```
git tag daemon-staging-v0.4.0 && git push --tags
```

先提升 workspace 版本：`xtask package` 拒绝与 `Cargo.toml` 不一致的标签。

两个值得知道的属性，因为它们是分割的*目的*：

- 预发布被普通的 `update apply` 跳过，因此未升级的构建不能到达没有用 `--staging` 要求它的机器人。两个发布步骤都在已存在的发布上重新断言该标志——否则有人起草时没勾选框的 staging 发布将可以被整个机群安装，而标记为预发布的稳定发布将不会交付给任何人。
- 升级从不重新构建。稳定发布最终是自包含的（清单、签名、制品、引导二进制），这就是为什么 staging 发布之后可以被删除。在那之前的稳定发布——`daemon-v0.3.0`——仍然将它们的 `url` 指向其 staging 发布，因此**那些 staging 发布不能被删除。**

手动升级是相同的配方，只是不需要先创建发布，也是 `min_supported` 所在的地方（§8.1——它强制低于该版本的机器人更新而不等待客户端）：

```
gh workflow run promote --field version=0.4.0
```

### 工作流

```
release.yml            入口：决定 staging / promote / stable
_build-release.yml     构建 · 打包 · 签名 · 验证 · 发布        （被调用）
_promote-release.yml   复制字节 · 验证 sha · 重新签名 · 退役 staging （被调用）
promote.yml            workflow_dispatch → _promote-release.yml
dev.yml                每次推送：一个对客户未签名的开发构建，`team.dev` 密钥
```

配方存在于两个被调用的工作流中，因此 staging 和 stable 路径在它们交付的内容上不会漂移。`xtask` 的打包绊线从 `_build-release.yml` 和 `dev.yml` 读取 `--include` 列表，因此未被打包的单元、钩子或 sysusers 文件会失败测试而不是失败机器人。

两个发布步骤都只在发布不存在时创建它，然后用 `--clobber` 上传，因此中途失败的作业可以重新运行。以前不是这样：一个在创建发布后死掉的发布作业只能通过手动删除发布和标签来重试。

## 轮换密钥

如果 `release-1` 或 CI 被攻破：

1. 将 `MINISIGN_SECRET_KEY` / `MINISIGN_PASSWORD` 替换为 `release-2` 的。
2. 发布一个由 `release-2` 签名的发布。机器人已经信任它——这就是为什么两个公钥从第一个镜像起就交付。
3. 在后续发布中从 `trusted_keys_dir` 移除 `release-1.pub`，因此被攻破的密钥停止被接受。
4. 生成一个替代的第三个密钥以便仍然有备用：
   `cargo xtask keygen --kind release --name release-4 --out ~/.duck-keys`

步骤 3 故意滞后于步骤 2：在每台机器人都获得新签名的发布之前撤销旧密钥会使任何错过它的机器人陷入困境。
#（注：内容由AI生成）
