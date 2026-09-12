# detect.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\detect.rs`

## 核心设计决策

在 tee 上已有的原始帧分支里寻找其他鸭子。`architecture.md` §2 要求感知贴近传感器（推导特征而非把像素送到 `robotd`），§5.3 要求按需取帧；原始分支自管线写成就为这两者存在，本模块是第一个读取它的东西。

两个关键决策：

1. **是线程，不是任务。** 推理每帧 60 ms 阻塞工作，而本 tokio 运行时还要服务 WebRTC 信令；占住其 worker 十分之一秒会让会话建立无故卡顿。
2. **限频，且频率是热数字。** Radxa Zero 3 上满速可达 95 °C、CPU 节流到 408 MHz——为看清而走不好的机器人。每秒两次对"那边有没有鸭子"绰绰有余，只耗约十分之一核。

后端按模型扩展名选择：`.rknn` 走 NPU，`.onnx` 走 CPU；不让用户同时声明两者。

## 类型

### `Sighting`
一次观察的结果及其代价：`width/height`（框所在的帧尺寸，供消费者缩放）、`found: Vec<Detection>`、`took_ms`（该帧的推理+解码耗时）。

### `Detector`
`broadcast::Sender<Arc<Sighting>>` + 两个 `AtomicU64` 计数器（`looks`、`seen`），供 `robot.health` 风格报告。

### `Backend`
`Npu(rknn::Model)` 或 `Cpu(onnx::Model)`。

## 函数

- **`spawn_first(models, frames, hz, threshold, turn)`**：逐个尝试模型，返回第一个成功加载的；全部失败给出诊断并提示 `robot-setup-npu`。
- **`spawn(model, …)`**：起一个名为 `duck-detect` 的线程，从 `frames.latest()` 取帧，`letterbox_from_uyvy` 一次把 4:2:2 转成模型方形（避免先转整帧再缩放的 345 ms 浪费），推理后 `decode`。
- 心跳每 20 次（10 秒 @2Hz）打一次 info，区分 `looks/seen`（自上次报告）与 `total_looks/total_seen`（累计）；`starved` 计数无帧可取的 tick——这是区分"线程死了/tee 静默/正常没鸭子"三种状态的唯一线索。
- **`video_notification` / `notification`**：把视频几何与检测框序列化为无 id 的 JSON-RPC 通知（与 `robot.state` 同一机制），手写 JSON 因本 crate 无 serde 依赖。

## 单元测试

| 测试 | 意图 |
|---|---|
| `a_sighting_serialises_as_a_notification` | 通知形状正确：无 id、框在帧自身像素内、附带帧尺寸 |
| `an_empty_sighting_is_still_sent` | 空检测也要发消息，否则画面上的上一只鸭子永远不会被清除 |

## 关键摘要

- 独立线程、2 Hz 限频、NPU/CPU 按扩展名自动回退。
- 检测结果走广播通道到对等端，慢消费者用 `Lagged` 丢弃旧帧只保留最新。
- 报告机制刻意区分"自上次"与"累计"，避免"看到一只鸭子二十分钟前"与"现在"都读成 `seen=1`。
