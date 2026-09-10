# NPU，以及运行在其上的鸭子检测器

RK3566 有一个小型 INT8 NPU——0.8 TOPS，单核。本文记录将一个训练好的鸭子检测器部署到它上面的过程：运行什么、预期什么，以及在行为可以使用它之前还缺什么。

模型在 [duck_detector](https://github.com/pollen-robotics/duck_detector) 中训练，以量化后的 `.rknn` 形式到达这里。第一个模型，供参考：`yolo11n`，320×320，单类，来自三个会话的 150 帧，在留出会话上 mAP50 为 0.976——INT8 量化后 3.9 MB，在桌面上与浮点模型对比，2 个检测中保留了 2 个，框重叠度 95%。

## 这里有什么

| | |
|---|---|
| `duck-detect` | 字母盒（letterbox）、运行时绑定和解码——外加 `duck-bench`。 |
| `scripts/setup-npu.sh` | 启用 NPU 节点，安装 `librknnrt.so`，并报告驱动状态。 |

在阅读任何一个之前，有两个决策值得了解：

**`dlopen`，而非链接。** `librknnrt.so` 是厂商二进制文件，不在任何 Debian 套件中，一个链接它的 crate 无法在 CI 中交叉编译。`robotd` 以同样的方式接入 ONNX Runtime。代价是 `duck-detect/src/rknn.rs`；好处是 `cargo board --bins` 在笔记本上仍然可用。

**运行时反量化。** 量化模型的输出张量是带 scale 和零点的 int8。如果要求，`rknn_outputs_get` 会转换为 float，并且确实要求了——替代方案是把 scale 带进解码器并在某个地方悄悄搞错一次。

## 运行基准测试

驱动是门控：它是厂商内核的一部分，主线内核没有，用户空间的任何东西都无法绕过它的缺失。

**一次普通的 `robotctl update` 就能做到。** `hooks/preinstall` 在 `setup-gstreamer.sh` 和 `setup-rkaiq.sh` 旁边运行 release 自带的副本，永远不会致命，其报告在更新日志中——因此一个在 NPU 存在之前就已配置的板可以通过更新修复，而不是靠某人记住一条命令。手动运行是重试：

```bash
sudo sh /opt/robot/daemon/current/scripts/setup-npu.sh
```

**预期携带此功能的第一次更新会要求重启。** Armbian 在每台 Radxa Zero 3 上将 `npu@fde40000`  shipped 为 `status = "disabled"`，因此一块原厂板有硬件、内核和驱动，却仍然没有 NPU。脚本写入修复它的 overlay 并说明这一点；节点在下一次启动时绑定。`--no-enable-node` 仅安装运行时，`dmesg | grep rknpu` 是事后确认的方式。

运行 release 副本而非 `/usr/local/sbin/robot-setup-npu`：overlay 源在脚本旁边，而留在 `/usr/local/sbin` 的副本在首次运行时旁边什么都没有。

然后，从你机器上的克隆：

```bash
cargo board --bins -p duck-detect
scp target/aarch64-unknown-linux-gnu/release/duck-bench microduck@<robot>:/var/tmp/
scp <the>.rknn microduck@<robot>:/var/tmp/duck.rknn
scp -r datasets/raw/<a-session> microduck@<robot>:/var/tmp/frames
```

`duck-bench` 不在 release 中：它是一个测量工具，打包它会为了两个人的利益把它放到每台机器人上。它通过 `scp` 传输，直到有一个行为需要检测器，届时随 release 发布的是 `mediad` 内部的检测器，而不是这个。

```bash
/var/tmp/duck-bench --model /var/tmp/duck.rknn --frames /var/tmp/frames
```

它按重要性顺序回答三个问题：

1. **它能运行吗？** 无法加载的运行时、为其他平台构建的模型，或比运行时更旧的驱动，都会在这里失败，而不是在 daemon 内部失败。
2. **它还能看到鸭子吗？** 它报告每帧的检测结果，因为一个能运行但什么都检测不到的模型看起来和一个正常工作的模型一模一样。
3. **代价是什么？** 延迟百分位数和*本进程*消耗的 CPU——使用 NPU 的理由是不干扰 `robotd` 的 50 Hz 循环，这是一个需要测量的主张。

`--threshold` 是首先要触及的标志。**量化模型的分数有自己的尺度**——浮点模型的 0.5 不是这个模型的 0.5——因此一个什么都检测不到的运行更可能是阈值问题，而不是转换失败。在相信最坏情况之前先试试 `0.2`。

## 数据

来自一台 Radxa Zero 3，`duck-bench` 在定速 2 Hz 下，3 次传递共 30 帧：

| | 测量值 | 备注 |
|---|---|---|
| 驱动 / 运行时 | 0.9.8 / 2.3.2 | `setup-npu.sh` 会打印两者 |
| 延迟 p50 / p95 | 25.7 ms / 58.4 ms | 推理加解码，不含 JPEG 解码 |
| 每帧 CPU | 20.7 ms | 见下文——这不全是推理 |
| 检测 | | 针对人已标注的帧 |
| SoC 温度 | 63 °C | 定速运行结束时 |

**CPU 数字不是 NPU 的代价，而它的报告方式容易让人读成是。** 延迟列计时 `infer` + `decode`；CPU 列是整个循环的进程 CPU 除以帧数，因此它还承载 `letterbox_rgb`——一个 1280×720 → 320×320 的重采样，在 CPU 上运行，完全不在延迟中。剩余部分是否意味着 `rknn_run` 忙等（把 NPU 等待计入 CPU）尚不清楚。在 2 Hz 下无论如何都是一个核的 4%；在任何人把这引用为感知的代价之前，应该把两者分开测量。

## 还缺什么

**机器人上没有任何东西能获取帧。** `mediad` 有一个原始 NV12 tee 分支，正是为此存在——`architecture.md` §5.3——但没有 IPC 暴露它，这也是为什么采集数据集必须停止 `mediad` 才能占用摄像头。两条前进路径，且不互斥：

- **`media.frame`**：一个回答一帧的调用。远不止感知有用（控制台中的快照、bug 报告的静帧），而且它让采集不再与 daemon 争抢。
- **`mediad` 内部的检测器**：订阅原始分支，以几 Hz 运行模型，并在状态流上发布检测结果。这是它的归宿——感知靠近传感器，派生特征而非传输像素——也是行为会消费的东西。

一旦检测结果作为状态存在，`docs/ideas/autonomous_behavior.md` 中当前以蓝牙为键的行为（"一只鸭子在*附近*"）就可以以视觉为键（"一只鸭子在*那里*"）：接近、跟随、对视，以及一个鸭子们唱歌时互相看着对方的 chorale。
#（注：内容由AI生成）
