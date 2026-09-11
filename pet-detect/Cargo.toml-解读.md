# Cargo.toml（pet-detect crate）解读与架构梳理

> 分析对象：`Cargo.toml`（32 行），pet-detect crate 的构建配置。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：这是 `pet-detect` crate 的 Cargo.toml——板载麦克风抚摸检测。一个 ~20KB 的 CNN，跑在 log-mel 窗口上。从 `apirrone/microduck_pet_detect` 移植——相同特征、相同模型、相同阈值——加上 arecord worker 和环境声哨兵（原来在原型的 `pet_worker.rs` 里），所以所有听麦克风的东西都在一个 crate 里。

**crate 结构**：
- **lib**：`pet_detect`（库）——MelExtractor、PettingDetector、特征提取、ONNX 推理
- **bin**：`pet-detect`（src/bin/detect.rs）——实时检测 CLI
- **bin**：`pet-features`（src/bin/features.rs）——训练特征提取 CLI

---

## 二、证据矩阵

| # | 事实 | 定位 | 状态 |
|---|------|------|------|
| F1 | crate 名 pet-detect，lib 名 pet_detect | L7, L14 | confirmed |
| F2 | 两个 bin: pet-detect + pet-features | L16-22 | confirmed |
| F3 | 从 apirrone/microduck_pet_detect 移植 | L3 | confirmed |
| F4 | ~20KB CNN over log-mel windows | L1 | confirmed |
| F5 | arecord worker + ambient sentry 合并到此 crate | L4-5 | confirmed |
| F6 | ort 版本 =2.0.0-rc.11（与 duck-control 同 pin） | L31 | confirmed |
| F7 | ort load-dynamic（动态加载 ONNX Runtime） | L31 | confirmed |
| F8 | rustfft 6.2（FFT 计算 log-mel） | L28 | confirmed |
| F9 | hound（WAV 读写） | L27 | confirmed |
| F10 | workspace 共享 edition/rust-version/license | L8-11 | confirmed |

---

## 三、依赖架构

```
pet-detect crate
├── lib: pet_detect
│   ├── MelExtractor (rustfft)
│   ├── PettingDetector (ort ONNX Runtime)
│   ├── load_wav_mono_16k (hound)
│   └── i16_to_f32 / 常量
├── bin: pet-detect
│   └── 实时检测 (stdin PCM → stdout 概率)
└── bin: pet-features
    └── 训练特征提取 (WAV → raw f32)

依赖:
├── anyhow       — 错误处理
├── clap (derive) — CLI 参数解析
├── hound        — WAV 文件读写
├── rustfft 6.2   — FFT（log-mel 特征）
├── tracing      — 日志
└── ort =2.0.0-rc.11 (load-dynamic) — ONNX Runtime
```

---

## 四、关键设计决策

### 4.1 为什么 ort 用 load-dynamic

`load-dynamic` 意味着 ONNX Runtime 不编译进二进制——运行时动态加载 `libonnxruntime.so`。板上只有一个 ONNX Runtime 实例，duck-control 和 pet-detect 共享同一个 .so。pin 到 `=2.0.0-rc.11` 保证两边版本一致。

### 4.2 为什么两个 bin 共享一个 lib

- `pet-detect`（实时推理）和 `pet-features`（训练特征提取）共享 `MelExtractor`——训练/推理 parity。
- 如果特征提取代码在两个 bin 里各写一份，必然漂移。
- lib 是唯一真相源。

### 4.3 为什么合并 arecord worker 和 ambient sentry

原来在原型的 `pet_worker.rs` 里。合并后：所有听麦克风的代码在一个 crate——音频管道、特征提取、模型推理、环境声监控都在一起，不用跨 crate 传音频流。

### 4.4 为什么 ~20KB CNN

板载 CPU 上跑——模型必须小。20KB 意味着推理快、内存低、不拖控制循环。

---

## 五、结论

### confirmed
- C1：pet-detect crate，lib + 2 bin。
- C2：~20KB CNN over log-mel，从 apirrone/microduck_pet_detect 移植。
- C3：ort =2.0.0-rc.11 load-dynamic，与 duck-control 共享 ONNX Runtime。
- C4：rustfft 做 FFT，hound 做 WAV。
- C5：arecord worker + ambient sentry 合并到此 crate。

### inferred
- I1：workspace 是 microduck 项目根，edition/rust-version/license 统一管理。
- I2：ort load-dynamic 需要板上预装 libonnxruntime.so。

---

*报告生成时间：2026-09-11 | SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
