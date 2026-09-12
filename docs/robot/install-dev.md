# 安装在一块 dev 板上

把一块板子从无到有变成一个你可以推送分支的机器人。

一块 dev 板信任团队 dev 密钥，因此它会安装团队中任何人构建的任何东西。一个客户机器人以不同方式设置且故意拒绝那些构建 —— 这里的一切都假设是一块 dev 板，绝不是客户机器人。

dev 构建没有任何东西被放松：相同的签名与哈希验证、相同的健康门、相同的自动回滚。唯一区别是哪把密钥签了它，而那正是让这些构建远离客户机器人的东西 —— 它们双重拒绝一个 dev 密钥。`allow_dev_keys = false`，且一个受信任密钥仅当其文件名以 `.dev.pub` 结尾才算 dev 密钥。下面设置的两半存在正是为了翻转那个。

## 刷板子

使用 [Armbian imager](https://www.armbian.com/radxa-zero-3/)。选 **Radxa Zero 3**，然后 **Armbian 26.2.1 Minimal**。

写入之前，填写 imager 的配置 —— wifi 网络与密码，以及你想要的用户名与密码。在那里做省掉之后的一个串口控制台：板子在第一次启动加入你的网络且立即可通过 ssh 触达。

然后添加你的 ssh 密钥，以便初始化在重启板子后可以重连：

```bash
ssh-copy-id radxa@192.168.1.42
```

## 你需要什么

- 板子的 **IP 地址**。这个镜像上的 mDNS 不可靠，因此一个 `.local` 名称在它想解析时才解析。`duckctl ip` 通过蓝牙问机器人，这不需要你自己的网络也不需要 DHCP 租约来读；如果板子还没广播，你的路由器的租约表是回退。
- **ssh 密钥访问**，来自上面的步骤。初始化重启板子并自己重连，而一个密码提示无法存活那个。
- 一个 **GitHub token**，在这个仓库私有时：它的发布资产没有它不可达。一旦公开，token 是可选的且只买到更高的 API 速率限制（`docs/design/updater-design.md` §6.1）。
- 一个**本仓库的克隆**。它需要的 dev 密钥提交在 `deploy/dev-key/team.dev.pub`，因此没有东西需要问别人要。

## 安装

从你自己机器上的一个克隆，两个命令：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
./scripts/provision-board.sh --pause-btd-on-pair --name <MY_COOL_ROBOT_NAME> radxa@192.168.1.42
```

那发送你的 dev 密钥、开始初始化、等待重启、流式传输日志，并以 `robotctl health` 结束。

### 为什么那个命令里有 `--pause-btd-on-pair`

在 aic8800 无线电上，当 `btd` 广播时手柄无法形成**新**绑定。该标志留下一个标记，以便 `robotctl pad pair` 在配对窗口停止 `btd` 并对适配器下电再上电，然后再次启动它。已有绑定不受影响 —— 一个已绑定的手柄在整个栈起来时连接并驱动 —— 因此代价是一个守护进程在配对时长内停机。

它在这里是默认的，因为一块需要它却没带它初始化的板子表现为一个无法配对的手柄，且你首先会追查的每个看似合理的原因都在别处。

### 三种配置，以及如何分辨你有哪种

有两个独立的故障，因此有两个标志。**配对一个手柄并读取失败**，然后选择：

| 你看到的 | 板子想要的 |
|---|---|
| 手柄绑定并驱动 | 无 —— 无标志初始化 |
| 手柄不绑定；最后一个 SMP 步骤永不完成 | `--pause-btd-on-pair` |
| 即使 `btd` 暂停手柄也不绑定 | `--weird-ble`（隐含暂停，并添加 `Privacy = device`） |
| 手柄绑定，然后**抖动** —— `PIN or Key Missing (0x06)`，无输入设备 | `--weird-ble` 对这块板是错的：去掉它，保留暂停 |

最后一行是要留意的。在一块只需要暂停的板子上 `Privacy = device` 产生一个立即停止工作的绑定，这比一个明显无法配对的手柄更难诊断 —— 在 `50:37:CD:16:1D:90` 上测量，`off` 加暂停绑定并保持，而 `device` 在 45 秒内抖动 46 次。因此 `--weird-ble` **不再**是默认。

要把一块板子从 `--weird-ble` 移到仅暂停，保留标记：

```bash
sudo sed -i 's/^Privacy = device/Privacy = off/' /etc/bluetooth/main.conf && sudo reboot
```

要反过来，用初始化留在板子上的脚本副本：

```bash
sudo DUCK_WEIRD_BLE=1 /usr/local/sbin/robot-setup-board && sudo reboot
```

要检查一块板子两者都不需要，去掉两者并配对一个手柄：

```bash
sudo rm /var/lib/robot/weird-ble
sudo sed -i '/^Privacy = /d' /etc/bluetooth/main.conf && sudo reboot
```

在任何对 `Privacy` 的更改后重新配对：它更改了存储密钥派生所依据的地址，因此已有绑定停止匹配并以 `PIN or Key Missing` 抖动，直到被重新制作。

所有这些都是对 aic8800 无线电的变通方案，非设计属性；它随无线电一起消失。[`pair-a-gamepad.md`](pair-a-gamepad.md) 有细节。

它是一个**查看器**，非做工作的东西 —— 初始化安装一个在启动时恢复的 systemd unit，因此无论你是否还在看，板子都会完成。Ctrl-C 不损失任何东西，且你可以重新接上日志：

```bash
ssh -t radxa@192.168.1.42 'sudo tail -f /var/lib/robot/provision.log'
```

`--ref BRANCH` 从一个分支初始化：它的脚本运行启动，且它的守护进程构建被安装在上面。`golden` 保持为 stable 版本 —— 它是启动恢复网的回退，而一个分支构建作为 golden 会给一个坏分支一个坏回退 —— 而 `current` 是分支。

如果那个构建无法安装，或它被安装然后被健康门回滚，初始化**失败**。一块在要求了一个分支时悄悄运行 stable 版本的 dev 板是最难调试的失败：一切看起来都安装了，而被测的代码不在那里。给 CI 它的一两分钟再初始化，且如果它停了用 `gh run list --branch BRANCH` 检查。

其他有用的标志：`--name Ducky` 命名机器人而非留它从自己的序列号派生的 `duck-7f3a`（`robotctl system set-name` 稍后更改它，因此这只省一个命令），`--local` 发送这个克隆的 `provision.sh` 而非获取它（这是在合并前测试初始化脚本变更的方式），且 `--no-dev-key` 做一块只接受版本的板子。

## 检查它工作了

```bash
robotctl health
```

板子只有在密钥真的安装了才算 dev 板，且那是一个你可以检查而非必须记住的东西：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`1` 表示是。`0` 表示密钥没落地，且 `--ref` 稍后会以一个读起来像损坏版本的错误被拒绝。这正是这个检查存在以尽早捕获的失败。

然后真正的测试 —— 给它放一个分支：

```bash
sudo robotctl update apply --ref main daemon
```

## 当 ssh 在重刷后拒绝连接

重刷会重新生成板子的主机密钥，因此你上次用的地址现在呈现一个不同的。`StrictHostKeyChecking=accept-new` 不覆盖它：主机不是新的，它的密钥是。这个的原始 ssh 错误是一堵关于可能攻击的文字墙。

```bash
./scripts/provision-board.sh radxa@192.168.1.42 --forget-host-key
```

这比听起来重要，因为 DHCP 租约会被复用 —— 上周是一块板子的地址今天是另一块板子，带一个不同的密钥。

## 当板子以一个不同地址回来

初始化中间的 wifi 切换可能让板子落在一个与你给的不同的租约上。`provision-board.sh` 会去找：在它等 ssh 的同时，它也通过蓝牙问机器人它最终落在了哪个地址，并采用那个答案。

```
  bluetooth: the robot reports 192.168.1.57, and 192.168.1.42 was its old lease.
```

无需做任何事 —— 运行的其余部分都定址到那里。

三件事阻止它工作，且它说明是哪件：

- 它需要 `cargo` 与这个克隆，因为 `duckctl` 是一个示例而非已安装的二进制。
- 它只能在 `btd` 运行后才能问，这在一块第一次被初始化的板子上是阶段 2 的几分钟后。
- 机器人报告它的 **wifi** 地址，因此你用以太网到达的板子不被覆盖。

`--no-ble` 关掉它。一个被给了自己配对 PIN 的机器人需要把它放在 `DUCK_PIN` 中。

## 让一块已有板子接受 dev 构建

给一块以其他方式初始化的板子，或一块在你有密钥之前设置的板子。两半都需要：单独任一半都会留下一块仍拒绝分支构建的板子。

简单的方法是用密钥重新运行安装程序，它验证密钥并在一步中翻转标志：

```bash
sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/install.sh
```

无论源文件叫什么，它都把密钥安装为 `team.dev.pub`。那个名字重要：`.dev.` 中缀正是把一个密钥分类为 dev 密钥的东西，而一个以任何其他名字落地的密钥被当作**发布**密钥信任。

手动，如果你更想看每一步：

```bash
sudo cp team.dev.pub /etc/robot/trusted_keys/team.dev.pub
```

```bash
sudo sed -i 's/^allow_dev_keys.*/allow_dev_keys        = true/' /etc/robot/updater.toml
```

```bash
sudo systemctl restart updaterd
```

## token，手动

当给定 `DUCK_TOKEN` 时 `scripts/install.sh` 为你写这个。这些步骤给一块以其他方式初始化的板子 —— 且它们只在这个仓库私有时需要，或在一块取够频繁想要 token 买到的更高速率限制的板子上。

`updaterd` 从它自己的环境读取 `GITHUB_TOKEN`，因此在你的 shell 中导出它到不了守护进程 —— 它需要一个 systemd drop-in。

```bash
sudo mkdir -p /etc/systemd/system/updaterd.service.d
```

在下一个块中替换你自己的 token —— 这是这里唯一的占位符：

```bash
sudo tee /etc/systemd/system/updaterd.service.d/token.conf > /dev/null <<'EOF'
[Service]
Environment=GITHUB_TOKEN=ghp_replace_with_your_token
EOF
```

一个 drop-in 默认是世界可读的，而这个持有一个凭据：

```bash
sudo chmod 600 /etc/systemd/system/updaterd.service.d/token.conf
```

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl restart updaterd
```

一个在*开发者*板子上的 token 没问题。一个在客户机器人上的 token 不是 —— 一个镜像中机群范围的凭据无法在不重刷的情况下轮换 —— 这就是为什么答案是一个公共仓库而非一个交付的 token（`docs/design/updater-design.md` §6.1）。一块没有 token 的板子仍可以从一个本地目录或一次 dev 推送安装。

## 无网络安装

在一块已经有一个版本的板子上，以普通方式安装一个本地目录 —— 通过守护进程，带健康门与自动回滚：

```bash
sudo robotctl update apply daemon --from /media/usb/release
```

那也是 `scripts/dev-push.sh` 结束时做的；[`dev-push.md`](dev-push.md) 是笔记本到板子的路径。

本节的其余部分是**裸板**情况：一次工厂或离线安装，在有一个守护进程可问之前。因为那个原因它是 `updaterd` 而非 `robotctl`，且 `updaterd` 故意不在 `PATH` 上：

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release
```

该目录持有一个版本是什么：`<version>.manifest.json`、它的 `.minisig`、产物以及产物的 `.minisig`。签名、哈希与兼容性被检查，与下载时完全一样 —— `--from` 改变字节从哪来，而非信任什么。

那个命令在一个版本上线后拒绝运行，因为它强制 `on_apply` 与健康门关闭，且对一个工作的机器人那么做会静默禁用自动回滚。有一种情况无论如何需要它，且 `robotctl update apply` 帮不上 —— 一块其已安装 `updaterd` 太旧无法接受*修复*太旧的那个版本的板子。它每次都回滚新版本，而运行那个门的二进制正是被替换的那个。停止机器人并明确说明：

```bash
sudo systemctl stop robotd
```

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release --force
```

`--force` 在 `robotd` 仍应答时本身被拒绝，因为反对意见是关于一个*工作的*机器人失去其安全网。它为那一次安装放弃自动回滚，其他什么都不放弃 —— 签名、哈希与兼容性仍被检查，且 `sudo robotctl update rollback daemon` 是如果版本行为不端时的恢复路径。

## 深入

[`deploy/README.md`](../../deploy/README.md) 是所有这些实际做什么的参考：信任链、什么最终落在哪里、其他进入方式（在无克隆的板子上，手动一步步）、日志去哪以及什么在重启中存活。
