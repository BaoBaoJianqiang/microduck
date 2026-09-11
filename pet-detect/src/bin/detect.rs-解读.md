# detect.rs（pet-detect 实时抚摸检测器）解读与架构梳理

> 分析对象：`detect.rs`（85 行），`pet-detect` 二进制——从板载麦克风实时检测头部抚摸。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `pet-detect`——实时抚摸检测器。从 stdin 读取 16-bit signed LE 单声道 PCM（16 kHz），在 stdout 上每次推理输出一行：`<ts_ms>\t<p>\t<state>`。

**用法**：
```
arecord -D plughw:aic3104,0 -f S16_LE -r 16000 -c 1 -t raw | pet-detect --model <onnx>
```

**设计要点**：
- **训练/推理一致性**：特征提取通过 `pet_detect` crate（MelExtractor），Python 训练脚本也通过 `pet-features` 二进制提取特征——模型永远在机器人实际计算的特征上训练。
- **滞回阈值**：enter=0.95（开始抚摸），exit=0.85（结束抚摸），防止抖动。
- **stride = WINDOW/4 ≈ 250ms**：推理窗口间隔。
- **事件走 stderr，结果走 stdout**：`println!` 输出推理结果到 stdout，事件（开始/结束）走 stderr——脚本可以管道化 stdout 而不被事件信息干扰。

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | stdin: S16_LE mono 16kHz PCM | L1-2 | confirmed |
| F2 | stdout: `<ts_ms>\t<p>\t<state>` | L2, L77 | confirmed |
| F3 | 默认模型路径 /opt/robot/daemon/current/models/pet_detect.onnx | L26 | confirmed |
| F4 | stride 默认 WINDOW_SAMPLES/4 ≈ 250ms | L31 | confirmed |
| F5 | enter_threshold=0.95, exit_threshold=0.85 | L34-37 | confirmed |
| F6 | 偶数字节校验（i16 流） | L63-65 | confirmed |
| F7 | 事件走 stderr | L80 | confirmed |
| F8 | 推理结果走 stdout | L77 | confirmed |
| F9 | 状态: "petting" / "normal" | L73-75 | confirmed |
| F10 | 依赖 pet_detect crate | L14 | confirmed |

---

## 三、数据流

```
arecord (板载麦克风)
  → 16kHz S16_LE mono raw audio
  → stdin
  → pet-detect
    ├─ 4096 字节 chunk
    ├─ i16 → f32 转换
    ├─ PettingDetector::push_samples()
    │   ├─ 提取 log-mel 特征
    │   ├─ ONNX 模型推理
    │   └─ 滞回状态机
    ├─ stdout: ts_ms \t probability \t state
    └─ stderr: event 事件（开始/结束抚摸）
```

---

## 四、关键设计决策

### 4.1 为什么事件走 stderr

stdout 是机器可读的推理流（`<ts>\t<p>\t<state>`），可以直接管道给其他脚本。事件（抚摸开始/结束）是调试/人看的——走 stderr 不污染 stdout。

### 4.2 为什么滞回阈值

0.95 进 / 0.85 出——概率在 0.90 附近抖动时不会反复切换状态。如果用单一阈值，抚摸判定会闪进闪出。

### 4.3 为什么 stride = WINDOW/4

每个推理窗口之间有 75% 重叠——时间分辨率高，不会漏过快的抚摸事件。

---

## 五、结论

### confirmed
- C1：实时抚摸检测 CLI，stdin PCM → stdout 概率+状态。
- C2：滞回阈值 0.95/0.85，stride ~250ms。
- C3：事件 stderr / 结果 stdout。
- C4：依赖 pet_detect crate（模型推理+特征提取）。

### inferred
- I1：PettingDetector 内部做 ONNX 推理和 log-mel 特征提取。
- I2：被 mediad 或 robotd 通过管道调用（音频管道）。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
