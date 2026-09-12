# 解析：`test-support/examples/fake-release.rs`

## 这是什么

一个 **Rust example 可执行程序**（169 行），作用是**手工铸造（mint）带签名的发布版**，让人能直接驱动真实的 `updaterd` 与 `robotctl`。它是 `test-support` crate 的 example（`cargo run -p test-support --example fake-release`），签名/打包能力来自库中的 `test_support::Publisher`。

文件头注释交代了定位：真实的两个二进制已经能完成机器人对发布版所做的一切（首装、apply 下一个、回滚坏版本、交换中途崩溃后恢复），但仓库里没有任何东西给你"一份**带签名的**发布版去喂它们"——`xtask package` 要求真实构建产物且版本要与 `Cargo.toml` 匹配，测试用的夹具又活在 `#[test]` 函数里。本程序就是这个缺失的零件，且只做这件事：**只发布（publish），从不安装**。

两个调用方：[`scripts/board-test.sh`](../../../scripts/board-test.sh)（主机上铸造、挂进 ARM64 容器）与 README 里的走查。

## 产出的目录树

```text
<root>/keys/prod.pub      信任公钥（机器人出厂自带的那种）
<root>/r/<version>/       每份带签名的发布版，已铸造但未提供
<root>/published/         "远端"：把发布版复制进去即等于提供它
<root>/opt/daemon/        安装树，`current` 符号链接将指向这里
<root>/var/               引擎状态：journal、锁、boot 计数器
<root>/updater.toml       配置
```

`r/` 与 `published/` 分开，是为了让调用者决定 `latest` 解析到什么：`local_dir` 源提供所指目录中**最新版本**，所以版本前进要逐个复制进去；而一份**应被拒绝**的发布可以先 stage 在 `r/` 里，从未具备可安装状态。

## CLI 参数（第 42-60 行）

| 参数 | 说明 |
|---|---|
| `root`（位置参数） | 要创建的目录；**已存在且非空则拒绝** |
| `--prefix` | 当树的*使用地*与铸造地不同时，写进 `updater.toml` 的路径（板测试主机铸造、容器挂载使用，路径全不同） |
| `releases`（必填，可多个） | 规格 `<version>[:tamper\|hook\|bad-hook]`，如 `1.0.0`、`1.2.0:tamper` |

三种变体：`tamper` 在签名后篡改 artifact（必须被拒绝）；`hook` 嵌入一个成功的 postinstall hook；`bad-hook` 嵌入一个失败的 hook（必须回滚）。

## main() 流程（第 62-168 行）

### 1. 空目录守卫（第 68-77 行）

非空则带完整说明退出。原因（注释）：每次运行铸造一把**全新密钥对**，向已有树追加会让旧发布无法用新盘上的密钥验证——"一个看起来与本夹具要检测的 bug 完全相同的签名失败"。

### 2. 建目录 + Publisher（第 79-83 行）

创建 `keys`、`published`、`opt/daemon`、`var`；`Publisher::new(keys_dir, published_dir)` 生成密钥对并负责后续签名发布。

### 3. 逐版本铸造（第 85-118 行）

- 按 `:` 拆分版本与变体；发布目录 `r/<version>`；
- 普通版与 `tamper` 版先发布为无 hook 的普通包；
- `hook`：嵌入 `echo "hook ran: $UPDATE_OLD_VERSION -> $UPDATE_NEW_VERSION"`（演示 hook 能拿到新旧版本环境变量）；
- `bad-hook`：嵌入 `echo 'migration failed' >&2; exit 1`；
- 未知变体直接报错——**失败关闭（fails closed）**：注释指出，敲错变体时静默发出一份普通包，会让调用方"必须被拒绝"的断言因错误的理由通过；
- `release.write()` 落盘后，`tamper` 再调 `publisher.tamper_in(dir, "daemon", version)` **在签名之后**破坏 artifact（顺序关键：先签后改，签名必然失配）。

### 4. 生成 updater.toml（第 120-148 行）

`prefix` 缺省取 root 的规范化绝对路径。配置逐项含义：

| 配置 | 值 | 为什么 |
|---|---|---|
| `trusted_keys_dir` / `state_dir` | `<prefix>/keys`、`/var` | 与树对齐 |
| `allow_fault_injection` | `true` | 否则 `--inject-fault` 被拒；客户机器人上永不设置 |
| `source.type` | `local_dir` 指向 `published/` | 用复制文件模拟"远端提供新版本" |
| `on_apply` | `{ action = "none" }` | **这里没有 systemd unit**。真实机器人上此值重启 robotd/mediad，永不重启 updaterd/btd（updater-design.md §4） |
| `health.probe` | `none` | 没有 robotd 应答；socket 探针无对象可问会一直轮询到超时、判失败并回滚每个发布。要演练回滚应改用 `--inject-fault fail_health` 故意打挂健康门 |
| `keep_previous` | `2` | 保留两个历史版本供回滚 |

### 5. 结尾提示（第 150-166 行）

无 `--prefix` 时打印可直接复制的下一步命令（把第一个版本复制进 `published/`，再跑 `updaterd install --from ...`）；有 `--prefix` 时只说明配置中的路径属于另一环境，不打印会骗人的命令。

## 与 systemd-fixture 的分工

本程序刻意铸造**无 unit、`on_apply = none`、健康探针 none** 的发布，用于驱动更新引擎自身的逻辑（下载、验证、拒绝、交换、回滚、崩溃恢复）——这是几乎所有测试的正确夹具；"重启机制是否真的工作"这一个问题由姊妹程序 [`systemd-fixture.rs`](systemd-fixture.rs解析.md) 回答。
