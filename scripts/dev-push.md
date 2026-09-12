# dev-push.sh

## 文件位置

`d:\microduck\scripts\dev-push.sh`

## 核心设计决策

该脚本在笔记本上构建守护进程并安装到板端，无需 CI。

- **两种构建方式，同一制品**：默认用 `cargo zigbuild` 交叉编译（最快，CI 也用）；`--docker` 在板端自己的用户态中构建（无需交叉，libudev 只需 apt-get）。
- **是普通更新**：板端通过 `robotctl update apply` 应用，预检、签名、制品哈希、兼容性、健康门控和自动回滚都与发布时完全一致——本地构建若起不来会被回滚，板端回到原来运行的版本。
- **版本格式 `<crate>-dev.local.<epoch>.g<sha7>`**：预发布版本，排序低于它前面的发布，永远不会看起来像车队的升级；每次 push 唯一（树可能脏，同一脏树两次 push 不能碰撞为"已是当前"）。
- **故意不做发布的来源认证**：版本带时间戳而非标签；用 dev 密钥签名（客户机器人拒绝）；不发布，别人不能安装你刚运行的东西。

## 常量/参数分析

### 命令行参数

| 参数 | 说明 |
|---|---|
| `--docker` | 在容器中构建而非 zigbuild |
| `--dry-run` | 只构建不应用 |
| `--bootstrap` | 强制无门控安装（适用于 `apply --from` 之前的旧 updaterd） |
| `--name` | 通过 BLE 名称找板端 |
| `user@host` | 直接指定板端地址 |

### 关键变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `KEY` | `$HOME/.duck-keys/team.dev.key` | team.dev 私钥，签名制品 |
| `REMOTE_DIR` | `$HOME/duck-sideload`（板端） | 制品落地目录 |
| `BOARD_USER` | `radxa` | 板端 ssh 用户 |
| `CACHE_DIR` | `$HOME/.cache/duck/boards` | BLE 地址解析缓存 |

## 核心逻辑

### 板端解析（名称优于地址）

地址会变（重刷、路由器重启、不同网络），mDNS 不可靠。机器人名称不变：`duckctl` 通过 BLE 按名称查找，`net.status` 返回当前地址。

缓存机制：每机器人一个缓存文件，先 ssh 探测缓存地址（4s 超时，BatchMode=yes），失败才 BLE 重新解析（10-20 秒）。

### 制品落地目录（非 /var/tmp）

`updaterd.service` 设 `PrivateTmp=yes`，unit 有自己的 `/tmp` 和 `/var/tmp`。制品放在 shell 看到的目录，updaterd 读命名空间给它的目录，`apply --from` 会找不到 manifest。改用 `$HOME/duck-sideload`（可写、在命名空间外，updaterd 无 `ProtectHome=`）。

### 构建

- **zigbuild 路径**：需要 `cargo-zigbuild` + `zig`，用 `cross-sysroot.sh` 构建 aarch64 sysroot（GStreamer 的 7 个 pkg-config 模块，版本与板端一致）。
- **docker 路径**：用 `scripts/dev-build.Dockerfile`，`--platform linux/arm64`，独立 target 目录 `target/docker`（避免 cargo 指纹混淆），命名卷缓存 registry。
- 只用 `DUCK_REVISION` 不用 `DUCK_BUILD_TIME`：每次 push 不变的树不重建任何 crate（节省 ~30s），版本字符串带 push 的 epoch。

### 打包与签名

- 拷贝 11 个二进制：updaterd、robotctl、robotd、configd、btd、padd、mediad、sounds、pet-detect、pet-features、tofd。
- `cargo run -p xtask -- package` 打包，`--zstd-level 1`（制品只读一次，压缩速度优先）。
- `--include` 列表与 `dev.yml`/`release.yml` 相同（xtask 测试断言一致）。
- `cargo run -p xtask -- sign` 用 dev 密钥签名。

### 应用

- `scp` 制品到 `$REMOTE_DIR`（替换而非追加）。
- 普通：`sudo robotctl update apply daemon --from '$REMOTE_DIR' --version '$VERSION'`。
- `--bootstrap`：`updaterd install --from --force`（强制关闭健康门控），然后重启 updaterd。

### 守护进程验证

apply 成功只意味着交换发生且健康门控通过，不意味着 7 个 daemon 都在运行新版本。轮询每个 daemon 的 `/run/<svc>/identity.json`，检查是否包含 `releases/<version>/bin/<svc>`。updaterd 和 btd 在回复后 5 秒重启，等 30 秒；其余在回复前已重启，不匹配即故障。

## 关键要点总结

1. 用 BLE 名称找板端而非地址，缓存减少 BLE 扫描开销。
2. 制品放 `$HOME/duck-sideload` 而非 `/var/tmp`，因 `PrivateTmp=yes` 使 updaterd 看不到。
3. 版本每次 push 唯一（epoch），避免脏树碰撞。
4. 构建只用 DUCK_REVISION，不变的树不重建。
5. apply 后验证所有 daemon 的 identity.json，确保真正运行新版本。
6. `--bootstrap` 用于旧版 updaterd 不支持 `apply --from` 的情况，代价是关闭健康门控一次。
