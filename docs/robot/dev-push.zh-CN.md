# 在这里构建，在板上安装

在更改一行和观看机器人运行它之间的循环，没有推送，没有 CI 运行，也没有标签。从这个仓库的克隆发出的一个命令为板构建，签名结果，并通过 ssh 安装它——在增量构建上大约一分钟，而推送加 CI 运行需要几分钟。

`scripts/dev-push.sh` 就是那个命令。下面的一切都是它上面的一个标志。

## 一次，在第一次推送之前

三件事，然后永远不再需要。

**板必须是开发板。** Artifact 用团队开发密钥签名，因此客户机器人拒绝它，就像它拒绝 `--ref` 一样。[`install-dev.md`](install-dev.md) 是板如何成为开发板的方式。

**开发签名密钥**放在 `~/.duck-keys/team.dev.key`——CI 签名分支构建所用密钥的秘密一半，团队成员拥有。如果你的住在别处，设置 `DUCK_DEV_SECRET_KEY`。

**一个可以为板构建的工具链。** 要么安装交叉编译器：

```bash
cargo install cargo-zigbuild --locked
```

```bash
brew install zig
```

要么什么都不安装，给下面的每个 `dev-push.sh` 传递 `--docker`，这只需要一个运行中的 Docker 守护进程。那条路径启动较慢，是在你根本没有板之前要找的路径——参见[在容器中构建](#build-in-a-container-instead)。

## 循环

命名机器人，让推送找到它：

```bash
scripts/dev-push.sh --name duck-c51b
```

或者每个 shell 一次：

```bash
export DUCK_ROBOT=duck-c51b
```

```bash
scripts/dev-push.sh
```

名字是 `duckctl scan` 列出且 `robotctl system set-name` 设置的那个——与该工具读取的 `DUCK_ROBOT` 相同（[`duckctl.md`](duckctl.md)）。它的地址通过蓝牙询问——机器人自己的 `net.status` 用它回答——然后缓存，因此只有无法到达缓存地址的推送才会回到无线电。这就是使新的 DHCP 租约、重新刷新或不同网络无需代价即可跟随的原因。

ssh 用户是 `radxa`。如果你的不是：

```bash
export DUCK_BOARD_USER=pierre
```

地址仍然有效，并完全跳过无线电：

```bash
scripts/dev-push.sh radxa@192.168.1.42
```

```bash
export DUCK_BOARD=radxa@192.168.1.42
```

它交叉编译工作区，打包与 release 相同的 artifact，用开发密钥签名它，将其复制到板上的 `~/duck-sideload`，并在那里通过 `robotctl update apply --from` 应用它。然后它等待守护进程报告新 release：

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

`updaterd` 和 `btd` 在 apply 回复后五秒重启，因此那两行需要一点时间才能到达。`[--] padd published nothing` 不是失败——它也是停止或故意禁用的守护进程的样子，以及 `mediad` 在没有摄像头的板上的样子。

`[--] tofd published nothing` 意味着来自 `tofd` 根本没有发布身份的 release——再推送一次修复它——而不是传感器缺失：`tofd` 无论是否安装了一个都运行。

在板上，那个版本就是 `robotctl version` 报告的：

```bash
robotctl version
```

同一脏树的两次推送永远不会碰撞——版本携带推送的时间戳，而不仅仅是提交。这里预期树是脏的。

**这是一个普通的更新。** 签名、artifact 哈希、兼容性、健康门和自动回滚都运行：一个不起来的构建会被还原，板回到它正在运行的东西。故意回去也是普通命令：

```bash
sudo robotctl update rollback daemon
```

## 观看你刚刚推送的东西

从第二个终端，在推送之前，这样重启会显示在其中：

```bash
ssh radxa@192.168.1.42 'journalctl -f -u robotd -u configd -u btd -u padd'
```

单独的 `-u updaterd` 是更新本身——每个阶段、健康门，以及它安排的重启：

```bash
ssh radxa@192.168.1.42 'journalctl -f -u updaterd'
```

守护进程中的 panic 会带着完整的回溯落在那里：这些二进制文件没有被剥离任何东西，因此帧有名字。

```bash
ssh radxa@192.168.1.42 robotctl health
```

控制循环是否启动，以及每个守护进程正在运行哪个 release。[`cheatsheet.md`](cheatsheet.md) 有要询问机器人的其余内容。

### 比 `info` 更多

每个单元设置 `RUST_LOG=info`。要查看守护进程的 `debug!` 行，用 drop-in 覆盖它——在板上，这里是 `robotd`：

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

Drop-in 住在 `/etc`，因此它在以后的每次推送中都幸存。完成后把它取下来：

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

构建、签名、复制，然后做真实 apply 做的一切，除了 swap：签名、哈希、兼容性、解压，以及检查没有已安装的单元留下指向这个构建不包含的二进制文件。它在 `current` 移动之前停止，因此板继续运行它正在运行的东西，没有守护进程重启。

## 改为在容器中构建

```bash
scripts/dev-push.sh --docker radxa@192.168.1.42
```

没有 zig，没有 `cargo-zigbuild`，也没有板可从中复制 libudev。在 Apple Silicon Mac 上，容器的目标是主机，因此没有交叉编译；在 x86 笔记本电脑上它在仿真下运行，脚本会说明。相同的 artifact，`--dry-run` 和 `--bootstrap` 工作方式相同。

两种模式保持单独的 `target/` 目录，因此在它们之间切换花费一次完整的重建。默认值日常更快，也是 CI 使用的。

## 重新安装已经在板上的东西

推送将 artifact 留在板上的 `~/duck-sideload` 中，因此可以再次安装它而无需构建任何东西：

```bash
sudo robotctl update apply daemon --from ~/duck-sideload
```

相同的命令接受任何持有 release 的目录——例如 U 盘。每次推送替换该目录而不是添加到它。

## 没有机器人为板编译

要检查更改是否为目标构建——C 链接的 crate、仅 Linux 的路径——而无需伸手可及的板：

```bash
cargo board --bins
```

`cargo board` 是带有板的目标和 glibc 地板的 `cargo zigbuild`，定义在 `.cargo/config.toml` 中。它需要与默认推送相同的 zig 工具链，而 `padd` 需要第一次推送从板获取的 libudev 副本。`--docker` 两者都不需要。

## 对低于 0.5.0 的板的第一次推送

`apply --from` 需要 API 版本 7，它首先在 0.5.0 中发布。运行任何更早版本的板有一个无法被要求使用它的 `updaterd`，并拒绝调用，而不是安静地从其配置的源安装。以无门控的方式交付那个 release 一次：

```bash
scripts/dev-push.sh --bootstrap radxa@192.168.1.42
```

那停止 `robotd` 并为那一次安装放弃健康门。之后的每次推送都是普通命令。

## 当它不工作时

**`no dev signing key at ...`** — 板像验证任何 release 一样验证这个 artifact，因此它必须被签名。从团队成员那里获取 `team.dev.key`，或者将 `DUCK_DEV_SECRET_KEY` 指向它。

**`cargo-zigbuild is not installed`** — 安装它和 zig，或者使用 `--docker`。

**`no libudev.so.1 on <board>`** — 第一次推送从板复制那个库来链接 `padd`。还没起来的板无法提供它；`--docker` 不需要它。

**重新刷新板后关于 libudev 的链接器错误** — 副本被缓存。删掉它，下一次推送获取一个新的：

```bash
rm -rf ~/.cache/duck-cross/aarch64
```

**`apply failed (exit 2)`，`robotctl` 和 `updaterd` 报告 API 不匹配** — 板的已安装 release 早于 `apply --from`。使用 `--bootstrap` 一次。

**`preflight check failed: SideloadDir: ... is not there for updaterd`** — release 在 `/tmp` 或 `/var/tmp` 下，如果 `DUCK_SIDELOAD_DIR` 或手写的 `--from` 指向那里就会发生。`updaterd.service` 设置 `PrivateTmp=yes`，因此守护进程有它自己的 `/tmp` 和 `/var/tmp`，两者都不是你的 shell 复制进去的那个。任何其他路径都有效；默认值 `~/duck-sideload` 是一个。在 release 早于该检查的板上，同样的错误读起来像 `no manifest for version <version> in <dir>`——对于一个 `ls` 恰好列出那个 manifest 的目录。

**`verification failed: signature did not verify against any of N usable trusted key(s)`** — 读起来像损坏的 release，通常意味着板不是开发板：开发密钥从未落地，或者 `allow_dev_keys` 关闭，两者都会使该密钥不在可用集合中。在板上：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`0` 意味着密钥缺失；[`install-dev.md`](install-dev.md) 有修复的两半。

**`could not reach <name> over Bluetooth`** — 机器人必须在广告且在范围内，名称路径才能解析地址。

```bash
duckctl scan
```

什么都没列出是一个关机、超出范围或已经连接到手机的机器人。改为给出地址，无线电就不参与：

```bash
scripts/dev-push.sh radxa@192.168.1.42
```

**`<name> 通过蓝牙回答但没有 wifi 地址`** — 它起来了但不在网络上，因此没有什么可 ssh 到。通过同一个无线电加入一个：

```bash
duckctl --name duck-c51b wifi connect <ssid> --psk <passphrase>
```

**`still <address>, which ssh could not reach`** — 地址从未移动，因此 ssh 不满意的是别的东西。重新刷新的板是常见的一个；参见下面的主机密钥。

**重新刷新后 ssh 拒绝连接** — 板重新生成了它的主机密钥。

```bash
./scripts/provision-board.sh radxa@192.168.1.42 --forget-host-key
```

**`the release is live but not everything is running it`** — swap 发生了且健康门通过了，但一个守护进程仍在旧二进制文件上，这读起来像一个不工作的修复。脚本命名哪些。在板上：

```bash
robotctl health
```

`units` 块命名每个守护进程正在运行的 release。

```bash
journalctl -u updaterd -b | tail
```

寻找 `restart scheduled`，或者没有的原因。

```bash
sudo systemctl restart updaterd
```

这不应该是必要的，因此值得阅读日志了解为什么。[`cheatsheet-dev.md`](cheatsheet-dev.md) 有完整的重启陷阱，[`../design/restart-order.md`](../design/restart-order.md) 是逐步的序列。

## 设置

| | |
|---|---|
| `DUCK_ROBOT` | 机器人，按名称。其地址通过蓝牙找到并缓存。 |
| `DUCK_BOARD_USER` | 板上的 ssh 用户，用于名称路径。默认 `radxa`。 |
| `DUCK_PIN` | 机器人的配对 PIN，如果它不是出厂的 `000000`。由 `duckctl` 读取。 |
| `DUCK_BOARD_CACHE` | 解析的缓存在哪里。默认 `~/.cache/duck/boards`。 |
| `DUCK_BOARD` | 板，按地址，而不是参数。`radxa@192.168.1.42`。 |
| `DUCK_DEV_SECRET_KEY` | 开发签名密钥。默认 `~/.duck-keys/team.dev.key`。 |
| `DUCK_SIDELOAD_DIR` | Artifact 在板上落在哪里。默认那里的 `~/duck-sideload`。永远不要在 `/tmp` 或 `/var/tmp` 下：`updaterd` 有两者的私有副本，会读取那些。 |
| `DUCK_CROSS_SYSROOT` | 缓存的 libudev 副本。默认 `~/.cache/duck-cross/aarch64`。 |

## 这故意不做的事情

没有 release 为来源所做的任何事情。版本携带时间戳而不是标签，artifact 用客户机器人拒绝的密钥签名，并且没有任何东西被发布——因此没有其他人可以安装你刚刚运行的东西。切割 release 仍然是标签和 `release.yml`（[`../../README.md`](../../README.md)）。
#（注：内容由AI生成）
