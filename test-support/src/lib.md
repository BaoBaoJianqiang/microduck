# lib.rs 文件解析（test-support）

## 文件位置

`d:\microduck\test-support\src\lib.rs`

## 定位

供测试用的已签名 release 发布器。支撑 `local_dir` 源——让测试无需网络即可端到端驱动真实引擎。

## 由来

四个测试文件各自复制了"生成密钥对 → 构建 `.tar.zst` → 签名 → 写 manifest → 再签名"。改 manifest 格式要改四处，且已漂移（如 tar mtime 是否固定）。

## 与引擎用相同 crate

刻意用 `minisign`/`tar`/`zstd`/`sha2`，使夹具不可能产出真实代码会因夹具自创理由而拒绝的 artifact。

## `Publisher`

目录：
- `keys_dir` — 引擎应指向的可信公钥目录
- `releases` — `local_dir` 源路径

方法：
- `new(keys_dir, releases)` — 生成**未加密**密钥对（测试要输密码就没人跑了），公钥写入 `keys_dir/prod.pub`
- `key_file()` — 公钥文件路径（测试替换信任锚时不用知道文件名）
- `public_key()` — 公钥行（最后一行，minisign box 格式）
- `sign(data)` — detached minisign 签名
- `publish(version)` — 发布 daemon 通道 release
- `release(version)` — 返回 `Release` builder
- `tamper(channel, version)` — 签名后破坏 artifact（追加 `0xff`）——模拟篡改的镜像或截断传输。定义在一处使"篡改"含义全局一致，防止调用方自创引擎恰好没拦的损坏
- `unpublish(channel, version)` — 删除 release（manifest+sig+artifact+sig），使 `latest` 回退到旧版——模拟回退的镜像
- `point_ref_at(git_ref, version)` — 把一个名字指向已发布版本（如 CI 的移动 `daemon-dev-<branch>` tag）。**复制** manifest 及其签名，不重签——ref 是指针，永不重新签名

## `Release` builder

直到 `write()` 才落盘。builder 模式因为调用点想要不同的单项偏离（一个 hook、不同 channel、额外 manifest 字段），一个全参数函数会有五个参数四个是 `None`。

方法：
- `dir(dir)` — 发布到不同目录（同密钥）。model release 需各自 remote——共享目录会让 daemon 的 `latest` 解析到 model manifest（`local_dir` 无视 channel 取最新版本）
- `channel(channel)` — 非 daemon 通道（model 组件或错通道测试）
- `file(path, contents, mode)` — 加文件，mode 对 `hooks/postinstall` 重要（必须可执行否则引擎 hook runner 无物可跑）
- `hook(script)` — 嵌入 `hooks/postinstall` mode `0o755`
- `manifest(edit)` — 签名前修改 manifest（兼容地板、`min_supported`），最后应用可覆盖一切
- `write()` — 写 artifact + `.minisig` + `<version>.manifest.json` + `.minisig`

artifact 始终含 `version.toml`（`version=<v>`）——引擎读它报告已装版本，多个测试通过 `current` 软链读它证明**内容**交换了而非仅链接。

## `append`

固定 mtime=0 使同输入同归档——xtask 依赖的可复现性，也是曾经拷贝夹具分歧的点。

## `live_version` / `live_marker`

- `live_version(install_dir)` — `current` 指向的版本（软链目标文件名）
- `live_marker(install_dir)` — 读 `current/version.toml`——**穿过软链读内容**，证明内容切换了而非仅链接

## 关键摘要

lib.rs 提供测试用签名 release 发布器 Publisher+Release builder：未加密密钥、固定 mtime、同引擎 crate 防自创拒绝；tamper/unpublish/point_ref_at 模拟篡改/回退/移动 ref；live_marker 穿软链读内容证明真交换。
