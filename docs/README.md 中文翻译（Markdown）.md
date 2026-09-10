# 文档

[README](../README.md) 是正门——什么是 microduck，以及该去哪里。如果你面前有一台，想要操控它，从[速查表](robot/cheatsheet.md)开始。

## `robot/` — 你有一台机器人

| | |
|---|---|
| [`cheatsheet.md`](robot/cheatsheet.md) | 所有 `robotctl` 命令。 |
| [`pair-a-gamepad.md`](robot/pair-a-gamepad.md) | 每个手柄一次：配对模式、`pad pair`，以及无法绑定（bond）时该怎么办。 |
| [`cheatsheet-dev.md`](robot/cheatsheet-dev.md) | 需要开发板的命令：分支构建、候选版本、开发推送。 |
| [`dev-push.md`](robot/dev-push.md) | 在你的机器上构建，通过 ssh 安装到板上，不经过 CI 运行。 |
| [`duckctl.md`](robot/duckctl.md) | 所有 `duckctl` 命令——从笔记本通过蓝牙操控机器人。 |
| [`install-dev.md`](robot/install-dev.md) | 从零开始为开发配置一块板。 |
| [`install-by-hand.md`](robot/install-by-hand.md) | 同样的安装拆成独立命令，用于一次只测试一个步骤。 |

## `design/` — 你在修改 daemon

它如何工作以及为什么。这些很少变动；当行为与设计文档不一致时，文档是 bug。

**一个机制只归一个页面所有，其他页面链接到它。** 下表就是这个归属分配：如果一个事实属于此处列出的某个页面，其他所有页面只说一句话并指向它，而不是再解释一遍。一个事实写在六个地方就会朝六个方向漂移，每个方向在局部看起来都合理——这就是为什么有六份文档都承诺 `updaterd` 和 `btd` 会保留旧二进制直到下次重启，而此时它们已经停止这么做两个版本了，包括有人在诊断这个问题时恰好会读的那两个页面。所以当两份文档不一致时，不拥有该机制的那一份是 bug。

| | |
|---|---|
| [`architecture.md`](design/architecture.md) | 服务拆分、IPC 契约、状态归属、安全与权限。 |
| [`robotd-design.md`](design/robotd-design.md) | 控制循环：Dynamixel 总线及谁拥有端口、模型、感知、观测、策略、安全——以及 tick 上还挂了什么。 |
| [`updater-design.md`](design/updater-design.md) | 更新引擎：验证、原子交换、健康门控、回滚、发布格式。 |
| [`restart-order.md`](design/restart-order.md) | 在每条移动 `current` 的路径上——以及启动时——哪个单元在哪个步骤重启。 |
| [`app-path-design.md`](design/app-path-design.md) | `btd` 和 `configd`——手机如何通过 BLE 配置机器人。 |
| [`remote-webrtc.md`](design/remote-webrtc.md) | WebRTC 会话、信令和控制通道——对端如何操控和观测机器人。 |
| [`webrtc-console.md`](design/webrtc-console.md) | WebRTC 客户端：从机器人提供服务、发现机器人，以及页面应该是什么样。 |
| [`boot-recovery-net.md`](design/boot-recovery-net.md) | 当启动的 release 无法启动其 daemon 时回退到 golden 镜像。 |

## `project/` — 你在运营项目

带日期的记录，而非参考文档。它们描述一个时刻，并故意会过时。

| | |
|---|---|
| [`roadmap.md`](project/roadmap.md) | 里程碑，以及今天能工作的 vs 已设计的。 |
| [`ci-setup.md`](project/ci-setup.md) | 发布流水线的一次性设置：密钥、机密、轮换。 |
| [`install-path-gap.md`](project/install-path-gap.md) | 为什么四个 install-path bug 流到了板上，以及什么修复了它。已关闭——它教出的规则在 [`updater-design.md`](design/updater-design.md) §9.1。 |
| [`slice-2-bringup.md`](project/slice-2-bringup.md) | 真实 Radxa Zero 3W 上 slice 2 的表现。 |
| [`update-over-ble.md`](project/update-over-ble.md) | 从手机驱动更新路径：发现了什么，以及通过无线电回滚是基于什么决定的。 |
| [`media-bringup.md`](project/media-bringup.md) | Radxa Zero 3W 的视频处理：VPU、MPP 需要什么，以及必须构建的两个插件。 |
| [`pad-minimal-pairing.md`](project/pad-minimal-pairing.md) | 游戏手柄能绑定的最小板载配置，通过逐个移除组件找到。 |

## `ideas/` — 尚未设计

暂存区。那些将来需要设计文档的东西，在有设计文档之前先写下来，这样思考不会丢失，也不会被误认为是决策。

| | |
|---|---|
| [`autonomous_behavior.md`](ideas/autonomous_behavior.md) | 行为栈：runtime 的大脑必须放弃什么，以及 chorale 和 theremin 工作留下的想法。 |

## 其他位置

| | |
|---|---|
| [`../CONTRIBUTING.md`](../CONTRIBUTING.md) | 构建、测试、仓库布局、约定、发布。 |
| [`project/npu-bringup.md`](project/npu-bringup.md) | RK3566 NPU 上的鸭子检测器：什么能运行、如何基准测试，以及仍然缺失的帧路径。 |
| [`../deploy/README.md`](../deploy/README.md) | 机器人镜像配置了什么，以及 provisioning 实际做了什么。 |
#（注：内容由AI生成）
