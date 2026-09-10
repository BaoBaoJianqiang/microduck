# 手动安装，逐步进行

`scripts/provision-board.sh` 做的事情，作为单独的命令。当某个步骤需要单独测试时使用这个；当你只想要一个能工作的板时使用 `provision-board.sh`。

## 复制文件上去

从你机器上的一个克隆。复制到 `~`，**不是** `/tmp`——中间有一次重启，而 `/tmp` 不能在重启后幸存。

```bash
scp scripts/setup-board.sh scripts/migrate-network.sh pierre@192.168.1.42:~/
```

```bash
scp scripts/install.sh deploy/dev-key/team.dev.pub pierre@192.168.1.42:~/
```

## 重启之前

先创建 `robot` 组，这样成员资格在重启后就生效，而不需要再重启一次：

```bash
sudo groupadd --system robot
```

```bash
sudo usermod -aG robot "$USER"
```

板 bring-up——设备树 overlay、内核控制台离开电机 UART、getty 掩码、`Privacy = device`、onnxruntime：

```bash
sudo sh ~/setup-board.sh
```

网络——netplan 到 NetworkManager：

```bash
sudo sh ~/migrate-network.sh
```

```bash
sudo reboot
```

重启不是可选的：设备树 overlay 和网络栈不能在运行中的内核下交换。

## 重启之后

两个都再跑一次。它们是幂等的，而第二次 `migrate-network.sh` 运行是退役 wifi 后备的步骤，否则该后备会在任何 wifi 缓慢的启动上将此板还原为 netplan：

```bash
sudo sh ~/setup-board.sh
```

```bash
sudo sh ~/migrate-network.sh
```

然后是守护进程。`install.sh` 从环境读取其设置，而 `sudo -E` 是让它们通过的方式：

```bash
export DUCK_TOKEN=github_pat_replace_with_your_token
```

```bash
export DUCK_REF=main
```

```bash
export DUCK_DEV_KEY=$HOME/team.dev.pub
```

```bash
sudo -E sh ~/install.sh
```

对于应该只接受 release 的板，去掉 `DUCK_DEV_KEY`。将 `DUCK_REF` 设置为一个分支以安装该分支最近构建的内容。

在上面的 `setup-board.sh` 运行上设置 `DUCK_WEIRD_BLE=1` 就是 `--weird-ble`：用于蓝牙根本无法绑定游戏手柄的板。参见 [`pair-a-gamepad.md`](pair-a-gamepad.md)。

`DUCK_NO_START=1` 安装 release、单元、用户和组，并启用**什么都不启用**——甚至不为下一次启动启用。它还会停止并禁用先前安装留下运行的任何守护进程，因此无论卡是新的还是不是，状态都是相同的。用于将板级故障与守护进程分开：重启到一个没有我们任何东西运行的板，测试，然后逐个启动它们。

```bash
sudo -E DUCK_NO_START=1 sh ~/install.sh
```

```bash
sudo reboot
```

重启是其中的一部分，不是整理。release 自己的 `hooks/postinstall` 在 `install.sh` 能停止它们之前启用并启动每个守护进程，因此无论旋钮怎么说，它们都在那次启动上运行过了——而守护进程在死亡时不会撤销它推送到子系统的东西。`btd` 留下 `Pairable` 设置、一个广告实例，以及其默认配对代理给适配器的 IO 能力。

要使其再次成为一个能工作的机器人：

```bash
sudo systemctl enable --now updaterd robotd configd btd padd
```

给它起个名字，如果你想要一个名字：

```bash
robotctl system set-name duck-01
```

## GStreamer，用于将要流式传输的板

Provisioning 默认执行此操作——`provision-board.sh` 上的 `--no-gstreamer`（这里是 `DUCK_GSTREAMER=0`）跳过它。两种方式都不需要重启，因此可以在跳过它的板上随时手动运行。

```bash
scp scripts/setup-gstreamer.sh pierre@192.168.1.42:~/
```

```bash
sudo sh ~/setup-gstreamer.sh
```

它打印此板可以编码什么——注册了哪些 `webrtc*` 元素、可以到达哪个 H.264 编码器，以及运行中的内核是否暴露了 VPU。在 Zero 3W 上它报告 `/dev/mpp_service` 且没有 `v4l2h264enc`，这是 Rockchip BSP 内核的预期形态：VPU 通过 MPP 而非 V4L2 到达，报告命名了证明它编码的两个 Radxa deb。在任何内核更改后重新运行它；那是改变答案的事件：

```bash
sudo /usr/local/sbin/robot-setup-gstreamer
```

添加 `--dev` 也安装头文件，用于在板上构建 `gst-plugin-webrtc` 或用于交叉构建的 aarch64 sysroot。

## 检查它

```bash
robotctl health
```

```bash
robotctl version
```

`robotd` 在舵机未通电的台架板上报告不健康是诚实的答案，不是安装失败。

然后是游戏手柄——[`pair-a-gamepad.md`](pair-a-gamepad.md)：

```bash
sudo robotctl pad pair
```
#（注：内容由AI生成）
