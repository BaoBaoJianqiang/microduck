# 在这里构建，在板子上安装

在改一行与看机器人运行它之间的循环，无推送、无 CI 运行、无标签。从这个仓库的一个克隆用一个命令为板子构建、签名结果并通过 ssh 安装 —— 增量构建约一分钟，对比一次推送加一次 CI 运行的数分钟。

`scripts/dev-push.sh` 就是那个命令。下面的一切都是它的一个标志。

## 第一次推送之前，一次

三样东西，然后永不再。

**板子必须是一块 dev 板。** 产物用团队 dev 密钥签名，因此一个客户机器人拒绝它，正如它拒绝 `--ref`。[`install-dev.md`](install-dev.md) 是一块板子如何成为 dev 板的方式。

**dev 签名密钥** 放在 `~/.duck-keys/team.dev.key` —— CI 签名分支构建所用密钥的秘密半，团队成员有。如果你的在别处，设 `DUCK_DEV_SECRET_KEY`。

**一个能为板子构建的工具链。** 要么安装交叉编译器：

```bash
cargo install cargo-zigbuild --locked
```

```bash
brew install zig
```

要么什么都不装，给下面每个 `dev-push.sh` 传 `--docker`，那只需要一个运行的 Docker 守护进程。那条路径启动更慢，且是在你完全没有板子之前该用的 —— 见[在容器中构建](#改为在容器中构建)。

## 循环

命名机器人，让推送找到它：

```bash
scripts/dev-push.sh --name duck-c51b
```

或每个 shell 一次：

```bash
export DUCK_ROBOT=duck-c51b
```

```bash
scripts/dev-push.sh
```

名字是 `duckctl scan` 列出且 `robotctl system set-name` 设置的那个 —— 与那个工具读取的同一个 `DUCK_ROBOT`（[`duckctl.md`](duckctl.md)）。它的地址通过蓝牙询问 —— 机器人自己的 `net.status` 用它应答 —— 然后缓存，因此只有一个无法到达缓存地址的推送才回到无线电。这正是让一个新的 DHCP 租约、一次重刷或一个不同网络无需代价就能跟随的原因。

ssh 用户是 `radxa`。如果你的不是：

```bash
export DUCK_BOARD_USER=pierre
```

一个地址仍工作，且完全跳过无线电：

```bash
scripts/dev-push.sh radxa@192.168.1.42
```

```bash
export DUCK_BOARD=radxa@192.168.1.42
```

它交叉编译工作区、打包与一个版本相同的产物、用 dev 密钥签名、复制到板子上的 `~/duck-sideload`，并通过 `robotctl update apply --from` 在那里应用。然后它等待守护进程报告新版本：

```
==> building 0.5.1-dev.local.1763400000.g7fc1444 for the board (zigbuild)
==> packaging
==> signing with /Users/you/.duck-keys/team.dev.key
==> copying to radxa@192.168.1.42:/home/radxa/duck-sideload
==> applying on radxa@192.168.1.42
==> 0.5.1-dev.local.1763400000.g7fc1444 is live on radxa@192.168.1.42
==> checking every daemon is running it
    current -> 0.5.1-dev.local.1763400000.g7fc1444
    [ok] robotd
    [ok] configd
    [ok] padd
    [ok] updaterd
    [ok] btd
    [ok] mediad
    [ok] tofd
==> every daemon on radxa@192.168.1.42 is running 0.5.1-dev.local.1763400000.g7fc1444
```

`updaterd` 与 `btd` 在 apply 应答后五秒重启，因此那两行需要一会才到达。一个 `[--] padd published nothing` 不是失败 —— 它也是一个停止或故意禁用的守护进程看起来的样子，以及 `mediad` 在一块没摄像头的板子上看起来的样子。

一个 `[--] tofd published nothing` 意味着一个早于 `tofd` 发布身份的版本 —— 再推一次修复它 —— 而非一个缺失的传感器：`tofd` 无论是否装了传感器都运行。

在板子上，那个版本正是 `robotctl version` 报告的：

```bash
robotctl version
```

同一脏树的两次推送永不碰撞 —— 版本携带推送的时间戳，不仅是 commit。这里预期树是脏的。

**这是一次普通更新。** 签名、产物哈希、兼容性、健康门与自动回滚都运行：一个不起来的构建被回滚，板子回到它正在运行的东西。故意回去也是普通命令：

```bash
sudo robotctl update rollback daemon
```

## 观察你刚推送的

从第二个终端，在推送之前，以便重启显示在里面：

```bash
ssh radxa@192.168.1.42 'journalctl -f -u robotd -u configd -u btd -u padd'
```

单独 `-u updaterd` 是更新本身 —— 每个阶段、健康门，以及它调度的重启：

```bash
ssh radxa@192.168.1.42 'journalctl -f -u updaterd'
```

守护进程中的一个恐慌会带着完整回溯落在那里：这些二进制没有任何东西被剥离，因此帧有名字。

```bash
ssh radxa@192.168.1.42 robotctl health
```

控制循环是否起来，以及每个守护进程在运行哪个版本。[`cheatsheet.md`](cheatsheet.md) 有该问一个机器人的其余东西。

### 不止 `info`

每个 unit 设 `RUST_LOG=info`。要看一个守护进程的 `debug!` 行，用一个 drop-in 覆盖它 —— 在板子上，这里以 `robotd` 为例：

```bash
sudo mkdir -p /etc/systemd/system/robotd.service.d
```

```bash
sudo tee /etc/systemd/system/robotd.service.d/log.conf > /dev/null <<'EOF'
[Service]
Environment=RUST_LOG=debug
EOF
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart robotd
```

该 drop-in 住在 `/etc`，因此它比之后每次推送都活得久。完成后把它拿掉：

```bash
sudo rm /etc/systemd/system/robotd.service.d/log.conf
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart robotd
```

## 不安装验证

```bash
scripts/dev-push.sh --dry-run radxa@192.168.1.42
```

构建、签名、复制，然后做真实 apply 做的一切，除了交换：签名、哈希、兼容性、解包，以及检查没有已安装的 unit 被留指向这个构建不包含的二进制。它在 `current` 移动前停止，因此板子保持运行它正在运行的东西，且没有守护进程重启。

## 改为在容器中构建

```bash
scripts/dev-push.sh --docker radxa@192.168.1.42
```

无 zig、无 `cargo-zigbuild`、且无需从板子复制 libudev。在 Apple Silicon Mac 上，容器的目标是主机，因此没有东西被交叉编译；在 x86 笔记本上它在模拟下运行且脚本会说明。同样的产物，且 `--dry-run` 与 `--bootstrap` 同样工作。

两种模式保持分开的 `target/` 目录，因此在它们之间切换代价一次完整重建。默认日常更快且是 CI 用的。

## 重新安装板子上已经有的东西

推送把产物留在板子上的 `~/duck-sideload`，因此它可以无需构建任何东西被再次安装：

```bash
sudo robotctl update apply daemon --from ~/duck-sideload
```

同一个命令接受任何持有一个版本的目录 —— 例如一个 U 盘。每次推送替换那个目录而非添加到它。

## 没有机器人时为板子编译

要检查一个变更为目标构建 —— C 链接的 crate、仅 Linux 的路径 —— 而无需一块板子在跟前：

```bash
cargo board --bins
```

`cargo board` 是带板子目标与 glibc 下限的 `cargo zigbuild`，定义在 `.cargo/config.toml`。它需要与默认推送相同的 zig 工具链，且 `padd` 需要第一次推送从板子获取的 libudev 副本。`--docker` 两者都不需要。

## 对低于 0.5.0 的板子的第一次推送

`apply --from` 需要 API 版本 7，它首次在 0.5.0 中交付。一块运行任何更早东西的板子有一个无法被要求使用它的 `updaterd`，且拒绝调用而非静默从其配置源安装。用无门控的方式交付那个版本一次：

```bash
scripts/dev-push.sh --bootstrap radxa@192.168.1.42
```

那停止 `robotd` 并为那一次安装放弃健康门。之后每次推送都是普通命令。

## 当它不工作

**`no dev signing key at ...`** —— 板子像验证任何版本一样验证这个产物，因此它必须被签名。从团队成员那里获取 `team.dev.key`，或把 `DUCK_DEV_SECRET_KEY` 指向它。

**`cargo-zigbuild is not installed`** —— 安装它与 zig，或用 `--docker`。

**`no libudev.so.1 on <board>`** —— 第一次推送从板子复制那个库来链接 `padd`。一块还没起来的板子无法提供它；`--docker` 不需要它。

**重刷板子后关于 libudev 的链接器错误** —— 副本被缓存了。删掉它，下一次推送获取一个新的：

```bash
rm -rf ~/.cache/duck-cross/aarch64
```

**`apply failed (exit 2)`，`robotctl` 与 `updaterd` 报告 API 不匹配** —— 板子的已安装版本早于 `apply --from`。用一次 `--bootstrap`。

**`preflight check failed: SideloadDir: ... is not there for updaterd`** —— 版本在 `/tmp` 或 `/var/tmp` 下，如果 `DUCK_SIDELOAD_DIR` 或一个手写的 `--from` 指向那里就会发生。`updaterd.service` 设 `PrivateTmp=yes`，因此守护进程有它自己的 `/tmp` 与 `/var/tmp`，且两者都不是你的 shell 复制进去的那个。任何其他路径都工作；默认 `~/duck-sideload` 是一个。在一块版本早于那个检查的板子上，同样的错误读为 `no manifest for version <version> in <dir>` —— 对一个 `ls` 恰好列出那个清单的目录。

**`verification failed: signature did not verify against any of N usable trusted key(s)`** —— 读起来像一个损坏的版本，且通常意味着板子不是 dev 板：dev 密钥从未落地，或 `allow_dev_keys` 关闭，两者任一都会让那个密钥不在可用集合中。在板子上：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`0` 意味着密钥缺失；[`install-dev.md`](install-dev.md) 有修复的两半。

**`could not reach <name> over Bluetooth`** —— 机器人必须在广播且在范围内，名称路径才能解析一个地址。

```bash
duckctl scan
```

什么都没列出是一个关机、超出范围或已经连接到一部手机的机器人。改为给地址，无线电就不参与：

```bash
scripts/dev-push.sh radxa@192.168.1.42
```

**`<name> answered over Bluetooth but has no wifi address`** —— 它起来了但不在一个网络上，因此没有东西可 ssh。通过同一个无线电加入一个：

```bash
duckctl --name duck-c51b wifi connect <ssid> --psk <passphrase>
```

**`still <address>, which ssh could not reach`** —— 地址从没变过，因此 ssh 不高兴的是别的什么。一块重刷的板子是常见的；见下面的主机密钥。

**重刷后 ssh 拒绝连接** —— 板子重新生成了它的主机密钥。

```bash
./scripts/provision-board.sh radxa@192.168.1.42 --forget-host-key
```

**`the release is live but not everything is running it`** —— 交换发生且健康门通过，但一个守护进程仍在旧二进制上，这读起来像一个没工作的修复。脚本命名那些。在板子上：

```bash
robotctl health
```

`units` 块命名每个守护进程在运行哪个版本。

```bash
journalctl -u updaterd -b | tail
```

寻找 `restart scheduled`，或没有的原因。

```bash
sudo systemctl restart updaterd
```

这不该是必要的，因此值得读日志看为什么是。[`cheatsheet-dev.md`](cheatsheet-dev.md) 有完整的重启陷阱，[`../design/restart-order.md`](../design/restart-order.md) 是一步步的序列。

## 设置

| | |
|---|---|
| `DUCK_ROBOT` | 机器人，按名称。其地址通过蓝牙找到并缓存。 |
| `DUCK_BOARD_USER` | 板子上的 ssh 用户，用于名称路径。默认 `radxa`。 |
| `DUCK_PIN` | 机器人的配对 PIN，如果它不是出厂的 `000000`。由 `duckctl` 读取。 |
| `DUCK_BOARD_CACHE` | 解析的地址缓存在哪。默认 `~/.cache/duck/boards`。 |
| `DUCK_BOARD` | 板子，按地址，而非参数。`radxa@192.168.1.42`。 |
| `DUCK_DEV_SECRET_KEY` | dev 签名密钥。默认 `~/.duck-keys/team.dev.key`。 |
| `DUCK_SIDELOAD_DIR` | 产物落在板子上的哪里。默认那里的 `~/duck-sideload`。永不在 `/tmp` 或 `/var/tmp` 下：`updaterd` 有两者的私有副本且会读那些。 |
| `DUCK_CROSS_SYSROOT` | 缓存的 libudev 副本。默认 `~/.cache/duck-cross/aarch64`。 |

## 这故意不做什么

一个版本为出处做的任何事。版本携带一个时间戳而非标签，产物用一把客户机器人拒绝的密钥签名，且什么都不发布 —— 因此没有其他人能安装你刚运行的东西。切一个版本仍是一个标签与 `release.yml`（[`../../README.md`](../../README.md)）。
