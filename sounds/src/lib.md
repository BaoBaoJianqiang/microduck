# lib.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\lib.rs`

## 核心设计决策

`sounds` 是机器人的声音合成器：一个可由单一种子确定性派生的鸭子叫声合成器。从 `apirrone/microduck_sounds`（Python/numpy）移植到 Rust，使发布版本自带语音生成器，不再需要 venv、numpy、ffmpeg。

一个整数种子确定性派生出 `Personality`（音高中心、谐波倾斜、鼻音、颤音、呱噪感、节奏）。两个种子听起来像两种不同的生物；同一种子永远一致。每个声音 **tag** 有多个 **variant**（同一声音内的小重抽），避免鸭子听起来像卡住的录音。

Tag 列表：`alarm`、`greet`、`inquire`、`peck`、`chirp`、`coo`、`wheee`。`wheee` 是分段的（start / loop / end），使 `robotd` 可以在滑行期间持续流式播放。

**移植带来的变化**：配方忠实移植，但随机流不是 numpy 的，且渲染原生 48 kHz 而非 22.05 kHz + 重采样——所以**每只鸭子的声音在音库重新生成时会重抽一次**。这是音库版本号提升（`BANK_VERSION=5`），与上游合成器重新调音是同一类事件，不是数据丢失：声音由 SoC 序列号派生，从此稳定，由 `rng.rs` 中的固定 RNG 测试守护。

**未移植**：`parrot` 模块（麦克风→学短语咯咯叫）——一个运行时未发货的实验。

## 常量分析

- `BANK_VERSION: u32 = 5` —— 当合成器变化到足以让现有音库在下次安装时重新渲染时递增。`.seed` 标记包含它，旧音库停止匹配并重新生成。v4 是最后一个 Python 音库，v5 是 Rust 移植（新 RNG、原生 48 kHz）。

## 类型分析

- `Recipe = fn(&Personality, u32) -> Vec<f32>` —— 配方类型签名：人格 + 变体输入，输出 `SR` 采样率的 f32 单声道。
- `TAGS: [(&str, Recipe); 7]` —— 所有 tag 及其配方，顺序即渲染顺序。
- `variant_count(tag)` —— 每个 tag 的变体数。`greet`/`chirp` 为 12（最常听到），`wheee` 为 6，其余为 10。

## 函数分析

- `render(tag, p, variant) -> Result<Vec<f32>>` —— 合成一个声音，返回 f32 mono。
- `to_wav(buffer, path)` —— 将缓冲区写为 16-bit 单声道 wav（采样率 `SR`）。
- `render_all(p, out_dir) -> Result<Vec<PathBuf>>` —— 将每个 (tag, variant) 渲染到 `<out_dir>/<tag>/<tag>_<letter>.wav`，即 `robotd` 播放的布局。分段 tag 写 `_start_`/`_loop_`/`_end_` 三元组。
- `hardware_seed() -> Result<u32>` —— 硬件派生的语音种子：SoC efuse 序列号 sha256 取 u32。序列号烧入芯片， survives 重刷。`/proc/device-tree/serial-number` 主路径，`/etc/machine-id` 回退。
- `seed_from_id(id) -> u32` —— `sha256(id)` 前 8 个十六进制字符作为 u32，与 Python 安装程序完全一致。

## 单元测试描述

- `the_seed_derivation_matches_the_installer` —— 固定种子派生与 shell 原版 (`sha256sum | cut -c1-8`) 一致。`"test-serial"` → `0xC96F_1146`。
- `render_all_writes_the_layout_the_robot_plays_from` —— `render_all` 写入机器人播放的文件布局，验证文件数与 `chirp/chirp_a.wav` 等存在。

## 关键摘要

lib.rs 定义了 sounds crate 的公共 API：7 个 tag 的配方表、音库版本控制、硬件种子派生（与 Python 安装程序兼容），以及离线批量渲染到 wav 的功能。声音是派生的而非存储的——音库在每次安装时从种子重新渲染，这也是 RNG 与采样率被显式固定和守护的原因。
