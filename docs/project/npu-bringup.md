# NPU 以及其上的鸭子检测器

RK3566 有一个小型 INT8 NPU —— 0.8 TOPS，单核。这是把一个训练好的鸭子检测器放上它的记录：运行什么、期望什么，以及在一个行为可以使用它之前还缺少什么。

模型在 [duck_detector](https://github.com/pollen-robotics/duck_detector) 中训练，以一个量化的 `.rknn` 来到这里。第一个模型，供参考：`yolo11n` 320×320，单类，来自三个会话的 150 帧，在一个留出会话上 mAP50 0.976 —— INT8 量化后 3.9 MB，在桌上对照浮点模型以 95% 框重叠保留了 2 个检测中的 2 个。

## 这里有什么

| | |
|---|---|
| `duck-detect` | 字母盒、运行时绑定、解码 —— 加上 `duck-bench`。 |
| `scripts/setup-npu.sh` | 启用 NPU 节点，安装 `librknnrt.so`，并报告驱动情况。 |

在阅读任一个之前值得知道的两个决定：

**`dlopen`，而非链接。** `librknnrt.so` 是一个不在任何 Debian 套件中的厂商 blob，而一个链接它的 crate 无法在 CI 中交叉编译。`robotd` 以同样方式到达 ONNX Runtime。代价是 `duck-detect/src/rknn.rs`；好处是 `cargo board --bins` 仍可在笔记本上工作。

**运行时反量化。** 一个量化模型的输出张量是带 scale 与 zero point 的 int8。如果被要求，`rknn_outputs_get` 会转换为 float，而它被要求了 —— 替代方案是把 scale 带进解码器并悄悄弄错一次。

## 运行基准

驱动是闸门：它是厂商内核的一部分，主线没有，且用户空间中没有任何东西能绕过它的缺席。

**一个普通的 `robotctl update` 做这件事。** `hooks/preinstall` 在 `setup-gstreamer.sh` 与 `setup-rkaiq.sh` 旁边运行版本自己的副本，从不会致命，其报告在更新日志中 —— 因此一个在 NPU 存在之前初始化的板子通过一次更新而非某人记得一个命令来修复。手动运行它是一次重试：

```bash
sudo sh /opt/robot/daemon/current/scripts/setup-npu.sh
```

**预期携带此内容的第一次更新会要求重启。** Armbian 在每个 Radxa Zero 3 上把 `npu@fde40000` 作为 `status = "disabled"` 交付，因此一块现货板子有硬件、内核与驱动，却仍没有 NPU。脚本写入修复它的 overlay 并说明；节点在下一次启动时绑定。`--no-enable-node` 只安装运行时，`dmesg | grep rknpu` 是你之后确认的方式。

运行版本副本而非 `/usr/local/sbin/robot-setup-npu`：overlay 源位于脚本旁边，而留在 `/usr/local/sbin` 的副本在第一次运行时旁边什么都没有。

然后，从你机器上的一个克隆：

```bash
cargo board --bins -p duck-detect
scp target/aarch64-unknown-linux-gnu/release/duck-bench microduck@<robot>:/var/tmp/
scp <the>.rknn microduck@<robot>:/var/tmp/duck.rknn
scp -r datasets/raw/<a-session> microduck@<robot>:/var/tmp/frames
```

`duck-bench` 不在一个版本中：它是一个测量工具，打包它会把它放到每个机器人上，只为两个人的利益。它经由 `scp` 走，直到有一个需要检测器的行为，届时交付的是 `mediad` 内部的检测器，而非这个。

```bash
/var/tmp/duck-bench --model /var/tmp/duck.rknn --frames /var/tmp/frames
```

它按重要性顺序回答三个问题：

1. **它能运行吗？** 一个无法加载的运行时、一个为另一平台构建的模型、或一个比运行时旧的驱动都在这里失败，而非在守护进程内部。
2. **它还能看见鸭子吗？** 它报告每帧检测，因为一个能运行且什么都检测不到的模型看起来与一个工作的模型完全一样。
3. **它代价多少？** 延迟百分位与*本进程*消耗的 CPU —— 使用 NPU 的理由是让 `robotd` 的 50 Hz 循环不受干扰，而那是一个要测量的断言。

`--threshold` 是第一个该用的标志。**量化模型的分数在它自己的尺度上** —— 浮点模型的 0.5 不是这个模型的 0.5 —— 因此一次什么都检测不到的运行更可能是阈值问题而非一个坏掉的转换。在相信最坏情况之前试试 `0.2`。

## 数字

来自一块 Radxa Zero 3，`duck-bench` 在调速后的 2 Hz，30 帧，3 次通过：

| | 测量值 | 备注 |
|---|---|---|
| 驱动 / 运行时 | 0.9.8 / 2.3.2 | `setup-npu.sh` 打印两者 |
| 延迟 p50 / p95 | 25.7 ms / 58.4 ms | 推理加解码，非 JPEG 解码 |
| 每帧 CPU | 20.7 ms | 见下 —— 这不全是推理 |
| 检测 | | 对照一个人已经标注过的帧 |
| SoC 温度 | 63 °C | 在一次调速运行结束时 |

**CPU 数字不是 NPU 的代价，而它报告的方式让人把它读成一个。** 延迟列计时 `infer` + `decode`；CPU 列是整个循环的进程 CPU 除以帧数，因此它也承载 `letterbox_rgb` —— 一个在 CPU 上运行的 1280×720 → 320×320 重采样，且完全不在延迟中。余数是否意味着 `rknn_run` 忙等（把 NPU 等待记到 CPU 上）尚不知道。在 2 Hz 下无论哪种方式都是一个核的 4%；在任何人把它引述为感知的代价之前，应该把两者分开测量。

## 还缺少什么

**机器人上没有任何东西能拿到一帧。** `mediad` 有一个原始 NV12 tee 分支，正是为此存在 —— `architecture.md` §5.3 —— 但没有 IPC 暴露它，这也是为什么采集数据集必须停止 `mediad` 才能拿摄像头。两条前进道路，且它们不互斥：

- **`media.frame`**：一个应答一帧的调用。对远不止感知有用（控制台中的快照、bug 报告的静帧），且它让采集不再与守护进程争抢。
- **`mediad` 内部的检测器**：订阅原始分支，以几 Hz 运行模型，并在状态流上发布检测。这是它最终的归宿 —— 挨着传感器的感知，派生特征而非交付像素 —— 且这是行为会消费的东西。

一旦检测作为状态存在，`docs/ideas/autonomous_behavior.md` 中当前以蓝牙为键的行为（"一只鸭子在*附近*"）可以以视觉为键（"一只鸭子在*那里*"）：接近、跟随、面向，以及一个鸭子们唱歌时互相看着对方的 chorale。
