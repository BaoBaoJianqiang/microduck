# main.rs 文件解析（xtask）

## 文件位置

`d:\microduck\xtask\src\main.rs`

## 定位

发布方 CLI：打包/签名/密钥管理/提升。发布契约的**发布方**一侧，永不发货到机器人。

## 命令

- `Package` — 组装 `.tar.zst` artifact + 未签名 manifest
- `Sign` — 用 minisign 密钥签目录下所有 artifact 和 manifest
- `Keygen` — 生成密钥对（release 加密长寿命 / dev 不加密 CI 用）
- `Keycheck` — 验证密钥对可用且公私匹配
- `Promote` — 从已发布 staging artifact 发出 stable manifest（不重建）

## 关键常量

- `VERSION_FILE="version.toml"` — artifact 内机器人身份文件
- `SIG_SUFFIX=".minisig"` — 签名文件后缀

## Package 流程

1. **版本漂移守卫**：`--version` 须匹配 workspace 版本；预发布版本（`0.2.0-dev.<run>.<sha>`）发布三元组匹配即接受（否则每个分支构建都要 `--allow-version-drift`，使该标志失效）
2. 读 `bin_dir` 所有文件 → `bin/<name>` mode `0o755`
3. `--include src=dest`：`hooks/` 和 `scripts/` 起头的给 `0o755`，其余 `0o644`
4. **preinstall hook 从模板生成**（不接受 `--include=hooks/preinstall`）——板前置条件是每个 release 的属性，不能靠人记得加 flag
5. 写 `version.toml`（version/channel/revision/binaries 列表）
6. zstd level 19（发布一次性操作）；CI smoke 用 level 1（否则未 strip 的 debug 包 ~400s，占任务过半）
7. 固定 mtime=0 使 artifact 可复现（同输入同归档）
8. manifest：channel/version/url/sha256/sig_url/size/min_hw_rev/schema_version=1，可选 source_revision、min_supported
9. **两份 manifest**：`manifest.json`（带 base_url）+ `<version>.manifest.json`（url 裸文件名，供 `LocalDir` sideload）——一次签名覆盖两种

## Keygen 设计

- **拒绝在仓库内写密钥**（提交的签名密钥无法通过删除来"反泄露"）
- Release 密钥：必须加密（`MINISIGN_PASSWORD` 或 `--password`），长寿命，所有机器人信任
- Dev 密钥：不加密（CI 非交互签名），`.dev.pub`/`.dev.key` 命名——`verify::KeyRing` 识别 `.dev.pub` 后缀，仅 `allow_dev_keys` 时可用
- **必须立即生成第二把 release 密钥**：机器人验证的是烧入镜像的公钥*集合*；只有一把时丢失/泄露只能人工重刷，无法 OTA 轮换。从第一个镜像就烧两把公钥
- 私钥文件 mode `0o600`，写入前设置（共享机器上短暂全局可读即已泄露）

## Keycheck

做**真实 sign-and-verify 往返**而非检查文件：能解析的密钥不一定能用，`.pub` 不一定是它的 `.pub`。先尝试未加密（自动判断密钥类型，不靠文件名猜）。

## Sign

签目录下所有非 `.minisig` 文件。artifact 和 manifest 都签：manifest 让机器人信任其内容，artifact 让字节可独立于 manifest 验证。

## Promote

- 读 staging manifest，校验 version 匹配
- url 重指向 stable release（`https://github.com/ORG/REPO/releases/download/<stable_tag>/<artifact>`）
- **sha256 不变**（同一字节）
- `min_supported` 不继承：为补救坏 staging 构建设的强制升级地板不应静默变成全舰队强制升级

## 辅助

- `onnxruntime_versions()` — 从 `[workspace.metadata.onnxruntime]` 读 floor+target，唯一真相源（preinstall hook、`setup-board.sh`、`ort` 必须一致；曾漂移导致板子能装 release 但加载 policy 时 panic）
- `render_preinstall_hook()` — 替换 `@ONNX_FLOOR@`/`@ONNX_TARGET@`，拒绝残留占位符

## 内联测试（一致性守卫）

大量测试防止 `--include` 列表（三份拷贝：`dev.yml`、`_build-release.yml`、`dev-push.sh`）漂移：
- 每个 install.sh 安装的 unit 必须被打包
- hooks 运行的每个 script 必须被打包
- 每个 install.sh 步骤必须到达更新过的板子（§9.1）——`ALSO_ON_UPDATE` 或 `FIRST_INSTALL_ONLY` 必须写明理由
- 每个 policy `.onnx` 必须打包
- pet model 必须打包
- `promote.yml` 必须上传它指向的 artifact + sig
- 每个 sysusers.d 文件必须打包
- 每个 repo 内的 hook 必须打包
- 每个 unit ExecStart 的 binary 必须被 stage
- `setup-board.sh`/`setup-npu.sh`/`setup-gstreamer.sh` 的字面版本必须与 Cargo.toml 一致
- preinstall 模板必须完全渲染
- `board-test.sh` 的 CHECKS 字符串内不含单引号（整个脚本在单引号字符串内传给 `sh -c`）
- 打印给操作员的命令必须用绝对路径

## 关键摘要

main.rs 是发布方 CLI：package 组装 artifact+manifest（preinstall 从模板生成、版本漂移守卫、双 manifest）；promote 同字节重签名指向 stable release；keygen 双 release 密钥轮换、拒绝仓库内写密钥；大量内联一致性测试防三份 `--include` 拷贝漂移。
