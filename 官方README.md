<p align="center">
  <img src="https://github.com/user-attachments/assets/c2f7c245-8217-46a1-8d1e-e0ba967cd969" alt="microduck" width="820">
</p>

<h1 align="center">Microduck</h1>

<p align="center">
  <em>一只用强化学习策略运动的小型双足机器人。</em>
</p>

<p align="center">
  <a href="https://pollen-robotics.com/microduck"><b>在这里购买</b></a> ·
  <a href="docs/robot/cheatsheet.md">速查表</a> ·
  <a href="https://github.com/pollen-robotics/microduck_rl">训练策略</a> ·
  <a href="docs/design/architecture.md">工作原理</a> ·
  <a href="CONTRIBUTING.md">参与贡献</a>
</p>

<p align="center">
  <a href="https://github.com/pollen-robotics/microduck/actions/workflows/ci.yml"><img src="https://github.com/pollen-robotics/microduck/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
</p>

---

**本仓库就是这只鸭子的大脑。** 约 25 cm、800 g 的机器人，由跑在 Rockchip RK3566 上的少量守护进程驱动：一个以神经网络策略带动十五个舵机的 50 Hz 控制环、无线电与摄像头，以及在不把机器人变砖的前提下把新软件装上机器人的更新机制。

运行一只 Microduck 所需的一切都在这里。**如果你想要一只，
[在这里购买](https://pollen-robotics.com/microduck)。**

它运行的策略在隔壁训练，位于
**[microduck_rl](https://github.com/pollen-robotics/microduck_rl)** —— MuJoCo 与 PPO、sim2real
配方，以及本仓库加载的 ONNX 导出。

## 它会做事

<table>
<tr>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/356a6011-8e0d-4b28-bda9-da78646583a3" controls width="100%"></video>
</td>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/abfbf250-1b1c-42cb-8430-00267e2b148a" controls width="100%"></video>

</td>
</tr>
<tr>
<td><b>它走路。</b>拿起手柄，驾驶。</td>
<td><b>它滚动。</b>装上轮子，按住十字键的上键，它就会加载另一个大脑。</td>
</tr>
<tr>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/7e70c1da-e120-428f-ae0b-f4de62f25984" controls width="100%"></video>
</td>
<td width="50%">
  <video src="https://github.com/user-attachments/assets/3eef63a5-6f84-47cf-90de-e717e6d7f8f0" controls width="100%"></video>
</td>
</tr>
<tr>
<td><b>它会捡东西。</b>喙到地面，一个按钮。</td>
<td><b>它会爬起来。</b>把它推倒，它自己站起来。</td>
</tr>
</table>

它还会坐下、踢球、按指令向前滚动，并用只属于它自己的声音嘎嘎叫。

## 到哪里找东西

### 你有一只鸭子

| | |
|---|---|
| [速查表](docs/robot/cheatsheet.md) | 每一条 `robotctl` 命令：驾驶、配置、声音、合唱、特雷门琴、wifi、更新、日志。从这里开始。 |
| [手柄](docs/robot/cheatsheet.md#gamepad-configd) | 完整的按键映射，以及配对一个手柄——[每个手柄一次](docs/robot/pair-a-gamepad.md)，外加绑不上时该怎么办。 |
| [`duckctl`](docs/robot/duckctl.md) | 从笔记本经蓝牙操控机器人，不需要网络也不需要 ssh。 |
| [更新](docs/robot/cheatsheet.md#updates-updaterd) | 安装、回滚、固定。每次更新都经验证、经健康门控且可回退。 |

### 你在它上面做开发

| | |
|---|---|
| [microduck_rl](https://github.com/pollen-robotics/microduck_rl) | 策略的来源：MuJoCo、PPO、域随机化，以及本仓库加载的 ONNX 导出。 |
| [工作原理](docs/design/architecture.md) | 整个系统在一页之内——守护进程、总线、一次更新如何到达机器人——然后每个部分一页。 |
| [配置一块 dev 板](docs/robot/install-dev.md) | 从一块空白板子到一台接受分支构建的机器人。 |
| [Dev 速查表](docs/robot/cheatsheet-dev.md) | 分支构建、发布候选、从笔记本驾驶，以及更新之后的重启陷阱。 |
| [推送你的分支](docs/robot/dev-push.md) | 在你的机器上构建、经 ssh 安装，约一分钟。 |
| [CONTRIBUTING.md](CONTRIBUTING.md) | 构建、测试、布局、约定、发布。 |
| [文档索引](docs/README.md) | 所有内容，包括设计页与未解决的问题。 |

## 底层一览

Rust，无框架，一个 workspace。`robotd` 拥有控制环与电机总线；`updaterd` 安装经签名的发布版，并在机器人起来后不健康时回滚；`configd` 拥有 wifi 与身份；`btd` 是手机使用的蓝牙路径；`padd` 读取手柄；`mediad` 经 WebRTC 推流摄像头；`tofd` 服务深度传感器。它们通过 Unix 套接字上的同一份 JSON-RPC 契约对话，而每个客户端——app、控制台、手柄、你的脚本——发送的调用完全相同。

有趣的决策都写了下来：[`docs/design/`](docs/design/) 是事情为什么是现在这样，而 [`docs/project/`](docs/project/) 是出过什么问题、以及什么能把它们关掉。

## 关于鸭子的说明

没有鸭子在制造这台机器人的过程中受到伤害。有若干只接受过咨询。
