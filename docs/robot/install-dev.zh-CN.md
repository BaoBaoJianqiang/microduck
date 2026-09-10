# 在开发板上安装

将板从一无所有变成你可以推送分支的机器人。

开发板信任团队开发密钥，因此它会安装团队中任何人构建的任何东西。客户机器人的设置不同，并故意拒绝那些构建——这里的一切都假设是开发板，永远不是客户机器人。

开发构建没有任何放松：相同的签名和哈希验证、相同的健康门、相同的自动回滚。唯一的区别是哪个密钥签名了它，而这就是使这些构建远离客户机器人的原因——它们双重拒绝开发密钥。`allow_dev_keys = false`，并且受信任密钥只有在其文件名以 `.dev.pub` 结尾时才算作开发密钥。下面设置的两半存在就是为了翻转 exactly 那个。

## 刷新板

使用 [Armbian imager](https://www.armbian.com/radxa-zero-3/)。选择 **Radxa Zero 3**，然后 **Armbian 26.2.1 Minimal**。

写入之前，填写 imager 的配置文件——wifi 网络和密码，以及你想要的用户名和密码。在那里做这件事省去了以后的串行控制台：板在第一次启动时加入你的网络，并立即可通过 ssh 到达。

然后添加你的 ssh 密钥，这样 provisioning 在重启板后可以重新连接：

```bash
ssh-copy-id radxa@192.168.1.42
```

## 你需要什么

- 板的 **IP 地址**。这个镜像上的 mDNS 不可靠，因此 `.local` 名称在它想的时候解析。`duckctl ip` 通过蓝牙询问机器人，这不需要你自己的网络也不需要 DHCP 租约来读取；如果板还没有在广告，你的路由器的租约表是后备。
- **ssh 密钥访问**，来自上面的步骤。Provisioning 重启板并自行重新连接，密码提示无法在那之后幸存。
- 一个 **GitHub token**，在这个仓库是私有的时候：没有它其 release 资产不可达。一旦公开，token 是可选的，只购买更高的 API 速率限制（`docs/design/updater-design.md` §6.1）。
- 一个 **这个仓库的克隆**。它需要的开发密钥提交在 `deploy/dev-key/team.dev.pub`，因此没有什么要向任何人要的。

## 安装

从你自己机器上的克隆，两个命令：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
./scripts/provision-board.sh --pause-btd-on-pair --name <MY_COOL_ROBOT_NAME> radxa@192.168.1.42
```

那发送你的开发密钥，开始 provisioning，等待重启，流式传输日志，并在 `robotctl health` 上结束。

### 为什么 `--pause-btd-on-pair` 在那个命令中

在 aic8800 无线电上，当 `btd` 正在广告时，手柄无法形成**新的**绑定。该标志留下一个标记，这样 `robotctl pad pair` 会为配对窗口停止 `btd` 并对适配器进行电源循环，然后再次启动它。现有的绑定不受影响——绑定的手柄在整个栈启动时连接并驱动——因此代价是一个守护进程在配对时长时间关闭。

它在这里是默认值，因为需要它但没有用它 provision 的板表现为一个不会配对的游戏手柄，而你首先追逐的每个可能原因都在别的地方。

### 三种配置，以及如何判断你有哪种

有两个独立的故障，因此有两个标志。**配对一个手柄并读取失败**，然后选择：

| 你看到的 | 板想要的 |
|---|---|
| 手柄绑定并驱动 | 什么都不需要——不带标志 provision |
| 手柄不会绑定；最后的 SMP 步骤永远不会完成 | `--pause-btd-on-pair` |
| 即使 `btd` 暂停手柄也不会绑定 | `--weird-ble`（暗示暂停，并添加 `Privacy = device`） |
| 手柄绑定，然后**抖动**——`PIN or Key Missing (0x06)`，没有输入设备 | `--weird-ble` 对这个板是错的：去掉它，保留暂停 |

最后一行是要注意的那个。在只需要暂停的板上设置 `Privacy = device` 会产生一个立即停止工作的绑定，这比一个明显不会配对的手柄更难诊断——在 `50:37:CD:16:1D:90` 上测量，其中 `off` 加暂停绑定并保持，而 `device` 在 45 秒内抖动 46 次。因此 `--weird-ble` **不再**是默认值。

要将板从 `--weird-ble` 移到仅暂停，保留标记：

```bash
sudo sed -i 's/^Privacy = device/Privacy = off/' /etc/bluetooth/main.conf && sudo reboot
```

要走另一条路，用 provisioning 留在板上的脚本副本：

```bash
sudo DUCK_WEIRD_BLE=1 /usr/local/sbin/robot-setup-board && sudo reboot
```

要检查板两者都不需要，去掉两者并配对一个手柄：

```bash
sudo rm /var/lib/robot/weird-ble
sudo sed -i '/^Privacy = /d' /etc/bluetooth/main.conf && sudo reboot
```

在对 `Privacy` 的任何更改后重新配对：它更改了存储密钥派生所针对的地址，因此现有的绑定停止匹配并以 `PIN or Key Missing` 抖动，直到它们被重新制作。

所有这些都是 aic8800 无线电的变通方法，不是设计的属性；它随无线电一起消失。[`pair-a-gamepad.md`](pair-a-gamepad.md) 有细节。

它是一个**查看器**，不是做工作的东西——provisioning 安装一个在启动时恢复的 systemd 单元，因此无论你是否还在观看，板都会完成。Ctrl-C 不花费你任何东西，你可以重新捡起日志：

```bash
ssh -t radxa@192.168.1.42 'sudo tail -f /var/lib/robot/provision.log'
```

`--ref BRANCH` 从分支 provision：其脚本运行 bring-up，其守护进程的构建安装在上面。`golden` 保持 stable release——它是启动恢复网的后备，而作为 golden 的分支构建会给坏分支一个坏后备——而 `current` 是分支。

如果该构建无法安装，或者它被安装然后被健康门回滚，Provisioning **失败**。当要求分支时，一个安静地运行 stable release 的开发板是最糟糕的调试失败：一切看起来都已安装，而被测的代码不在那里。在 provisioning 之前给 CI 一两分钟，如果它停止，用 `gh run list --branch BRANCH` 检查。

其他有用的标志：`--name Ducky` 命名机器人而不是让它保持从其自己的序列号派生的 `duck-7f3a`（`robotctl system set-name` 稍后更改它，因此这只节省一个命令），`--local` 发送这个克隆的 `provision.sh` 而不是获取它（这是在不先合并的情况下测试对 provisioning 脚本的更改的方式），以及 `--no-dev-key` 制作一个只接受 release 的板。

## 检查它工作了

```bash
robotctl health
```

板只有在密钥真的安装了才算作开发板，而这是你可以检查而不是必须记住的事情：

```bash
grep -c 'DEV BOARD' /var/lib/robot/provision.log
```

`1` 意味着是。`0` 意味着密钥没有落地，`--ref` 稍后会被拒绝，错误读起来像损坏的 release。这就是这个检查存在的原因，以便及早捕获。

然后是真正的测试——在它上面放一个分支：

```bash
sudo robotctl update apply --ref main daemon
```

## 当重新刷新后 ssh 拒绝连接时

重新刷新会重新生成板的主机密钥，因此你上次使用的地址现在呈现一个不同的密钥。`StrictHostKeyChecking=accept-new` 不覆盖它：主机不是新的，它的密钥是。对此的原始 ssh 错误是一堵关于可能攻击的文本墙。

```bash
./scripts/provision-board.sh radxa@192.168.1.42 --forget-host-key
```

这比听起来更重要，因为 DHCP 租约会被重用——上周是一个板的地址今天是另一个板，带有不同的密钥。

## 当板以不同的地址回来时

Provisioning 中间的 wifi 切换可能会让板留在与你给的不同的租约上。`provision-board.sh` 去寻找：在它等待 ssh 的同时，它也通过蓝牙询问机器人它最终得到了什么地址，并采用答案。

```
  bluetooth: the robot reports 192.168.1.57, and 192.168.1.42 was its old lease.
```

什么都不用做——运行的其余部分在那里寻址。

有三件事阻止它工作，它会说是哪个：

- 它需要 `cargo` 和这个克隆，因为 `duckctl` 是一个示例而不是已安装的二进制文件。
- 它只能在 `btd` 运行后询问，在第一次被 provision 的板上这是第 2 阶段的几分钟后。
- 机器人报告其 **wifi** 地址，因此你通过以太网到达的板不被覆盖。

`--no-ble` 关掉它。被赋予了自己配对 PIN 的机器人需要它在 `DUCK_PIN` 中。

## 使现有板接受开发构建

对于以其他方式 provision 的板，或者在你有密钥之前设置的板。两半都是需要的：单独任何一个都会留下一个仍然拒绝分支构建的板。

简单的方法是用密钥重新运行安装程序，它验证它并一步翻转标志：

```bash
sudo DUCK_TOKEN="$DUCK_TOKEN" DUCK_DEV_KEY=/tmp/team.dev.pub sh /tmp/install.sh
```

它将密钥安装为 `team.dev.pub`，无论源文件叫什么。那个名字很重要：`.dev.` 中缀是将密钥分类为开发密钥的东西，而以任何其他名字落地的密钥被信任为**release**密钥。

手动，如果你宁愿看到每个步骤：

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

当给定 `DUCK_TOKEN` 时，`scripts/install.sh` 为你写这个。这些步骤适用于以其他方式 provision 的板——并且它们只在这个仓库私有时需要，或者在获取足够频繁以想要 token 购买的更高速率限制的板上需要。

`updaterd` 从其自己的环境读取 `GITHUB_TOKEN`，因此在你的 shell 中导出它不会到达守护进程——它需要一个 systemd drop-in。

```bash
sudo mkdir -p /etc/systemd/system/updaterd.service.d
```

在下一个块中替换你自己的 token——这是这里唯一的占位符：

```bash
sudo tee /etc/systemd/system/updaterd.service.d/token.conf > /dev/null <<'EOF'
[Service]
Environment=GITHUB_TOKEN=ghp_replace_with_your_token
EOF
```

Drop-in 默认是世界可读的，而这个持有凭据：

```bash
sudo chmod 600 /etc/systemd/system/updaterd.service.d/token.conf
```

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl restart updaterd
```

在*开发者的*板上的 token 是可以的。在客户机器人上的 token 不是——镜像中的机群范围凭据无法在不重新刷新的情况下轮换——这就是为什么答案是公共仓库而不是随附 token（`docs/design/updater-design.md` §6.1）。没有 token 的板仍然可以从本地目录或开发推送安装。

## 没有网络安装

在已经有 release 的板上，以普通方式安装本地目录——通过守护进程，带有健康门和自动回滚：

```bash
sudo robotctl update apply daemon --from /media/usb/release
```

这也是 `scripts/dev-push.sh` 结束的方式；[`dev-push.md`](dev-push.md) 是笔记本电脑到板的路径。

本节的其余部分是**裸板**情况：工厂或离线安装，在有守护进程可询问之前。因为那个原因它是 `updaterd` 而不是 `robotctl`，而 `updaterd` 故意不在 `PATH` 上：

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release
```

该目录持有 release 是什么：`<version>.manifest.json`、其 `.minisig`、artifact 和 artifact 的 `.minisig`。签名、哈希和兼容性的检查与下载时完全相同——`--from` 更改字节来自哪里，而不是信任什么。

一旦 release 上线，该命令拒绝运行，因为它强制 `on_apply` 和健康门关闭，而对一个工作的机器人那样做会静默禁用自动回滚。有一种情况无论如何都需要它，而 `robotctl update apply` 无法帮助——一个板，其已安装的 `updaterd` 太旧，无法接受*修复*太旧的 release。它每次都将新 release 回滚，而运行该门的二进制文件正是被替换的那个。停止机器人并明确说明：

```bash
sudo systemctl stop robotd
```

```bash
sudo /opt/robot/daemon/current/bin/updaterd install --from /media/usb/release --force
```

当 `robotd` 仍在回答时，`--force` 本身被拒绝，因为反对意见是关于一个*工作的*机器人失去其安全网。它为那一次安装放弃自动回滚，其他什么都不放弃——签名、哈希和兼容性仍然被检查，如果 release 行为不端，`sudo robotctl update rollback daemon` 是恢复路径。

## 深入

[`deploy/README.md`](../../deploy/README.md) 是所有这一切实际做什么的参考：信任链、什么最终在哪里、其他进入方式（在没有克隆的板上、手动逐步）、日志去哪里以及什么在重启后幸存。
#（注：内容由AI生成）
