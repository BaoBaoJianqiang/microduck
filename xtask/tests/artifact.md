# tests/artifact.rs 文件解析（xtask）

## 文件位置

`d:\microduck\xtask\tests\artifact.rs`

## 定位

按 CI 的方式打包一个 release，然后**打开 tarball 检查内容**。这是比 `main.rs` 内字符串匹配测试更强的形式——后者只断言两个源文件一致，在描述本身错误时仍通过。`docs/project/install-path-gap.md` 选项 A 即此文件。

## 背景：两个 bug 同日到达板子

1. release 携带了 artifact 不含的 unit
2. （加了防第一个的测试两个 commit 后）unit 要 exec 的 binary 没打包——`btd.service` 报 `203/EXEC`，板上读起来像 daemon 坏了而非 release 不完整

## 此测试补上的盲区

- **无 unit exec 的 binary**（`robotctl`——install.sh 软链到 PATH，现有检查从 ExecStart 推导结构上看不到）
- **hook mode**（`hooks/postinstall` 缺执行位在更新门内静默失败）
- **生成的 hook**（`hooks/preinstall` 由 package 渲染，`.in` 模板被跳过）
- **package 本身运行**（`src=dest` 拆分、mode 赋值、版本漂移守卫、version.toml 的 binaries 列表——此前只能靠切 release 触发）

## 三个打包站点（PACKAGING_SITES）

1. `.github/workflows/dev.yml`（每次 push 运行，开发时上板的 artifact——必须排第一）
2. `.github/workflows/_build-release.yml`（staging+stable 共用的可复用 workflow；`release.yml` 现在只是入口）
3. `scripts/dev-push.sh`（笔记本→板路径，用相同 `cp … staged/` 和 `--include` 字面形式以便被本文件覆盖）

三份手维护拷贝是漂移风险——漂移的那一份就是那周没人切 release 的那一份。

## 核心：`packaged_release(site)`

- 从 workflow YAML 解析 staged binaries（`cp … staged/` 的 basename）和 `--include src=dest` 对
- 用 **stub binary**（`#!/bin/false`）而非真实 aarch64 二进制——名字才是承重部分，交叉编译太慢
- 调用真实 `xtask package`（用 workspace 真实版本，使版本漂移守卫被实际执行）
- zstd level 1（artifact 读一次就扔；shipping 默认 19 会让测试占满套件）
- 打开 tarball 返回 `BTreeMap<path, (mode, bytes)>`

## 测试项

- `the_artifact_carries_what_install_sh_reads` — updaterd.service/robotd.service 必在；bin/robotctl 必在（否则软链悬空）；scripts/robot-rescue 必在且可执行
- `every_unit_in_the_artifact_can_exec_what_it_names` — 从 artifact 内 unit 的 ExecStart 推导 binary 必须存在（`current/bin/<name>` 或 `/usr/local/sbin/<name>` 对应 `scripts/<name>`）
- `the_boot_recovery_net_is_packaged_whole` — timer+oneshot+两个脚本齐全；oneshot 无 `[Install]`（否则安装时 `enable --now` 会在更新中触发回滚检查）；check 脚本可执行
- `hooks_are_packaged_executable` — preinstall（生成的）和 postinstall 必在，所有 hooks 可执行位
- `every_include_lands_where_the_workflow_says` — 每个 `--include` 的 dest 路径在 artifact 中

## 关键摘要

artifact.rs 是打包的端到端验证：从三个打包站点的 YAML 解析真实打包配方，用 stub binary 跑 xtask package，打开 tarball 断言 install.sh 会读的一切都在且 mode 正确。补盲：无 unit exec 的 binary、hook mode、生成的 preinstall、package 本身运行。
