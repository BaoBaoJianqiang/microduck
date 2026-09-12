# 手动安装，一步步

`scripts/provision-board.sh` 做的事，作为独立的命令。当某一步需要单独测试时用这个；当你只想要一块工作的板子时用 `provision-board.sh`。

## 把文件传上去

从你机器上的一个克隆。放进 `~`，**不是** `/tmp` —— 中间有一次重启，而 `/tmp` 不存活重启。

```bash
scp scripts/setup-board.sh scripts/migrate-network.sh pierre@192.168.1.42:~/
```

```bash
scp scripts/install.sh deploy/dev-key/team.dev.pub pierre@192.168.1.42:~/
```

## 重启之前

先建 `robot` 组，以便成员关系在重启后生效而非需要再重启一次：

```bash
sudo groupadd --system robot
```

```bash
sudo usermod -aG robot "$USER"
```

板子启动 —— 设备树 overlay、内核控制台离开电机 UART、getty 掩码、`Privacy = device`、onnxruntime：

```bash
sudo sh ~/setup-board.sh
```

网络 —— netplan 到 NetworkManager：

```bash
sudo sh ~/migrate-network.sh
```

```bash
sudo reboot
```

重启不是可选的：一个设备树 overlay 与一个网络栈不能在一个运行中的内核下面交换。

## 重启之后

两者都再跑一次。它们是幂等的，且第二次 `migrate-network.sh` 运行正是退役 wifi 回退机制的那次，否则它会在任何 wifi 慢的启动上把这块板子还原到 netplan：

```bash
sudo sh ~/setup-board.sh
```

```bash
sudo sh ~/migrate-network.sh
```

然后守护进程。`install.sh` 从环境读取它的设置，而 `sudo -E` 正是让它们通过的东西：

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

对一个只应该接受版本的板子去掉 `DUCK_DEV_KEY`。把 `DUCK_REF` 设为一个分支以安装该分支最后构建的东西。

上面 `setup-board.sh` 运行时的 `DUCK_WEIRD_BLE=1` 就是 `--weird-ble`：给一块蓝牙完全无法绑定手柄的板子。见 [`pair-a-gamepad.md`](pair-a-gamepad.md)。

`DUCK_NO_START=1` 安装版本、unit、用户与组，且**什么都不**启用 —— 甚至下次启动也不。它还停止并禁用前一次安装留下运行的任何守护进程，因此无论卡是否全新状态都一样。用于把一个板子级故障与守护进程分开：重启到一块我们的东西都没运行的板子，测试，然后一个个把它们带起来。

```bash
sudo -E DUCK_NO_START=1 sh ~/install.sh
```

```bash
sudo reboot
```

重启是它的一部分，非整洁。版本自己的 `hooks/postinstall` 在 `install.sh` 能停止它们之前启用并启动每个守护进程，因此它们在那个启动上已经运行过，无论旋钮怎么说 —— 且一个守护进程死亡时不会撤销它推给子系统的东西。`btd` 留下 `Pairable` 设置、一个广播实例，以及其默认配对代理给适配器的 IO 能力。

要让它再次成为一个工作的机器人：

```bash
sudo systemctl enable --now updaterd robotd configd btd padd
```

给它命名，如果你想要一个名字：

```bash
robotctl system set-name duck-01
```

## GStreamer，给一块会流式传输的板子

初始化默认做这个 —— `provision-board.sh` 上的 `--no-gstreamer`（这里 `DUCK_GSTREAMER=0`）跳过它。两种方式都不重启，因此它可以随时在一块跳过它的板子上手动运行。

```bash
scp scripts/setup-gstreamer.sh pierre@192.168.1.42:~/
```

```bash
sudo sh ~/setup-gstreamer.sh
```

它打印这块板子能编码什么 —— 哪些 `webrtc*` 元素被注册、哪个 H.264 编码器可达、以及 VPU 是否被运行中的内核暴露。在 Zero 3W 上它报告 `/dev/mpp_service` 且无 `v4l2h264enc`，这是 Rockchip BSP 内核的预期形态：VPU 通过 MPP 而非 V4L2 到达，且报告命名两个证明它能编码的 Radxa deb。在任何内核变更后重新运行它；那是改变答案的事件：

```bash
sudo /usr/local/sbin/robot-setup-gstreamer
```

加 `--dev` 也安装头文件，用于在板子上构建 `gst-plugin-webrtc` 或一个 aarch64 sysroot 来交叉构建。

## 检查

```bash
robotctl health
```

```bash
robotctl version
```

`robotd` 在一块舵机未通电的工作台板子上报告不健康是诚实的答案，非失败的安装。

然后手柄 —— [`pair-a-gamepad.md`](pair-a-gamepad.md)：

```bash
sudo robotctl pad pair
```
