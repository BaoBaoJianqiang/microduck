# 文档

[README](../README.md) 是入口——microduck 是什么，以及该去哪看。如果你面前有一台机器想驱动它，从[速查表](robot/cheatsheet.md)开始。

## `robot/` —— 你有一台机器人

| | |
|---|---|
| [`cheatsheet.md`](robot/cheatsheet.md) | 所有 `robotctl` 命令。 |
| [`pair-a-gamepad.md`](robot/pair-a-gamepad.md) | 每个手柄一次：配对模式、`pad pair`，以及无法绑定时怎么办。 |
| [`cheatsheet-dev.md`](robot/cheatsheet-dev.md) | 需要开发板的命令：分支构建、候选版本、开发推送。 |
| [`dev-push.md`](robot/dev-push.md) | 在你的机器上构建，通过 ssh 安装到板端，无需 CI 运行。 |
| [`duckctl.md`](robot/duckctl.md) | 所有 `duckctl` 命令——在笔记本上通过蓝牙操作机器人。 |
| [`install-dev.md`](robot/install-dev.md) | 从零开始把一块板子配置为开发板。 |
| [`install-by-hand.md`](robot/install-by-hand.md) | 同样的安装拆成单独命令，用于逐步测试。 |

## `design/` —— 你在修改 daemon

它如何工作以及为什么。这些很少改动；当行为与设计文档不一致时，文档是 bug。

**一个机制由一个页面负责，其他页面链接到它。** 下表就是这种分配：如果一个事实属于此处列出的某个页面，其他每个页面只说一句话并指向它，而不是再解释一遍。写在六个地方的事实会朝六个方向漂移，每个方向在局部都合理——这就是为什么六份文档曾承诺 `updaterd` 和 `btd` 会保留旧二进制直到下次重启，而在它们停止这么做的两个版本之后，包括有人在诊断恰好这个问题时读的那两个页面。所以当两份文档不一致时，不负责该机制的那份是 bug。

| | |
|---|---|
| [`architecture.md`](design/architecture.md) | 服务划分、IPC 契约、状态归属、安全与权限。 |
| [`robotd-design.md`](design/robotd-design.md) | 控制循环：Dynamixel 总线与谁拥有端口、模型、感知、观测、策略、安全——以及 tick 上还挂着什么。 |
| [`updater-design.md`](design/updater-design.md) | 更新引擎：验证、原子交换、健康门控、回滚、发布格式。 |
| [`restart-order.md`](design/restart-order.md) | 在每条移动 `current` 的路径上以及启动时，哪个 unit 重启、在哪一步重启。 |
| [`app-path-design.md`](design/app-path-design.md) | `btd` 和 `configd`——手机如何通过 BLE 配置机器人。 |
| [`remote-webrtc.md`](design/remote-webrtc.md) | WebRTC 会话、信令和控制通道——对端如何驱动和观察机器人。 |
| [`webrtc-console.md`](design/webrtc-console.md) | WebRTC 客户端：从机器人提供服务、发现机器人，以及页面应该是什么样。 |
| [`boot-recovery-net.md`](design/boot-recovery-net.md) | 当启动的 release 无法启动其 daemon 时回退到 golden。 |

## `project/` —— 你在运营项目

带日期的记录，而非参考。它们描述某个时刻，并会故意过时。

| | |
|---|---|
| [`roadmap.md`](project/roadmap.md) | 里程碑，以及今天能用什么 vs. 设计了什么。 |
| [`ci-setup.md`](project/ci-setup.md) | 发布流水线的一次性配置：密钥、机密、轮换。 |
| [`install-path-gap.md`](project/install-path-gap.md) | 为什么四个安装路径 bug 到达了板端，以及什么堵住了它。已关闭——它教给的规则在 [`updater-design.md`](design/updater-design.md) §9.1。 |
| [`slice-2-bringup.md`](project/slice-2-bringup.md) | 一块真实的 Radxa Zero 3W 对 slice 2 做了什么。 |
| [`update-over-ble.md`](project/update-over-ble.md) | 从手机驱动更新路径：发现了什么，以及决定通过无线电做回滚的依据。 |
| [`media-bringup.md`](project/media-bringup.md) | Radxa Zero 3W 如何处理视频：VPU、MPP 需要什么，以及必须构建的两个插件。 |
| [`pad-minimal-pairing.md`](project/pad-minimal-pairing.md) | 手柄能绑定的最小板端配置，通过逐一去掉配置项找到。 |

## `ideas/` —— 尚未设计

占位。某些东西将来需要设计文档，在它有之前先写下来，这样思路不会丢失，也不会被误认为决定。

| | |
|---|---|
| [`autonomous_behavior.md`](ideas/autonomous_behavior.md) | 行为栈：运行时的大脑必须放弃什么，以及合唱与特雷门琴工作留下的想法。 |

## 其他

| | |
|---|---|
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | 构建、测试、仓库布局、约定、发布。 |
| [`project/npu-bringup.md`](project/npu-bringup.md) | RK3566 NPU 上的鸭子检测器：运行什么、如何基准测试，以及仍然缺失的帧路径。 |
| [`../deploy/README.md`](../deploy/README.md) | 机器人镜像配置了什么，以及配置实际做了什么。 |
