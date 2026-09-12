# ci-release-notes.sh

## 文件位置

`d:\microduck\scripts\ci-release-notes.sh`

## 核心设计决策

该脚本组装发布说明：先说明本次发布存在的原因（PROVENANCE），再列出变更内容（changelog）。

- **为什么存在**：被 `_build-release.yml` 和 `_promote-release.yml` 共同调用，确保两者对发布的描述不会漂移。此前晋升发布只写"Promoted from daemon-staging-v0.4.0"，而变更日志在 staging 发布上，晋升时被删除，导致稳定发布没有内容说明。
- **changelog 来源**：使用 GitHub 自带的生成器而非仓库内的 CHANGELOG.md。手动维护的 CHANGELOG.md 是第二个容易遗忘的地方，而生成器已知道合并的 PR，这正是本项目历史的构成。
- **上一个稳定发布作为左边界**：显式查找上一个非预发布、非草稿的 `daemon-v` 标签，而非让生成器自选。因为 `dev.yml` 每次推送都会发布预发布，"自上一个发布以来"通常意味着"20 分钟前"，没有意义。
- **容错设计**：changelog 生成失败不致命。有 changelog 的说明优于没有，没有 changelog 又优于发布失败——此脚本在制品签名和验证之后运行。

## 常量/类型/函数分析

### 环境变量（输入）

| 变量 | 是否必填 | 说明 |
|---|---|---|
| `TAG` | 必填 | 正在发布的标签 |
| `PROVENANCE` | 必填 | 一两句话说明这些字节来自哪里 |
| `GH_TOKEN` | 隐含必填 | `gh` 命令所需 |

### 核心逻辑

1. **查找上一个稳定发布**：
   ```sh
   gh release list --limit 100 --json tagName,isPrerelease,isDraft \
     --jq '[.[] | select(.isPrerelease == false and .isDraft == false) | .tagName]
           | map(select(startswith("daemon-v")))
           | first // empty'
   ```
   过滤掉预发布和草稿，只保留 `daemon-v` 前缀，取第一个。

2. **生成 changelog**（两种情况）：
   - 有上一个稳定发布且不等于当前 TAG：调用 `gh api repos/.../releases/generate-notes`，传入 `tag_name` 和 `previous_tag_name`。
   - 无上一个稳定发布或当前就是它：调用同一 API，只传 `tag_name`，由生成器自行处理。

3. **输出**：先打印 `PROVENANCE`，若有 changelog 则打印分隔线 `---` 和 changelog 正文，否则打印 `_No changelog could be generated for this release._`。

## 关键要点总结

1. 脚本使用 `set -eu`，严格模式。
2. 所有 `gh` 调用都带 `2>/dev/null || true`，失败不中断发布流程。
3. 输出到 stdout，由 CI workflow 捕获。
4. 核心问题解决：晋升发布不会丢失变更日志，因为每次发布时都用 GitHub 生成器重新生成。
5. 左边界选择稳定发布而非最新发布，确保 changelog 反映"自人们实际运行的上一版本以来的变化"。
