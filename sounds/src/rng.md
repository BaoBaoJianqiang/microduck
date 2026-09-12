# rng.rs 文件解析

## 文件位置

`d:\microduck\sounds\src\rng.rs`

## 核心设计决策

一个小型确定性 RNG，配方依赖它，刻意自有。Python 原版用 `np.random.default_rng`（PCG64 + numpy 分布）。机器人的声音是**派生**的，不是存储的——音库在每次安装时从种子重新渲染——所以生成器**就是**声音。依赖 `rand` 会把每只鸭子的声音绑定到该 crate 今年发布的算法；`StdRng` 明确保留变更权。40 行我们拥有的 xoshiro 不会漂移。

**实现**：xoshiro256++，通过 splitmix64 播种（均为公共领域，Blackman & Vigna）。均匀分布取高 53 位；正态分布用 Box–Muller。无需与 numpy 匹配——移植时所有声音重抽一次，音库版本提升使这成为重新生成而非损坏。

## 类型分析

- `Rng` 结构体：`s: [u64; 4]`（xoshiro 状态）、`spare_normal: Option<f32>`（Box–Muller 产生成对，备用值下次取用）。

## 函数分析

- `from_seed(seed: u32)` —— 像 Python 那样从 u32 播种（`seed & 0xFFFFFFFF`）。splitmix64 将其扩展为四个 xoshiro 字，使小种子也能良好混合。
- `next_u64()` —— xoshiro256++ 核心步进。
- `random() -> f64` —— [0,1) 均匀，取高 53 位（f64 尾数全精度）。
- `uniform(lo, hi)` —— [lo, hi) 均匀。
- `integers(lo, hi) -> i64` —— [lo, hi) 均匀整数（numpy `integers` 半开约定）。
- `choice(choices)` —— 均匀选一个元素。
- `standard_normal() -> f32` —— 标准正态，Box–Muller，缓存备用值。
- `standard_normal_vec(n)` —— n 个标准正态。
- `crc32(data) -> u32` —— IEEE CRC-32，如 `zlib.crc32`——变体 RNG 的 tag 哈希。多项式由已在现场的声音固定。

## 单元测试描述

- `the_stream_is_pinned` —— 同种子产生相同流，固定值防止无依赖重构悄悄改变流（那会重新配音整个机群）。
- `uniform_stays_in_range` —— 10000 次均匀分布落在范围内。
- `normals_have_sane_moments` —— 100000 个正态样本均值≈0、方差≈1。
- `crc32_matches_zlib` —— CRC-32 与 zlib 参考值一致：`b""`→0、`b"chirp"`→`0xEC28_BF8B`、`b"123456789"`→`0xCBF4_3926`。

## 关键摘要

rng.rs 是整个声音系统的基石：xoshiro256++ + splitmix64 + Box–Muller + CRC-32，全部手写无依赖。因为"生成器就是声音"，RNG 的输出流被测试钉死，任何重构若改变流会响亮失败。这是把声音身份锁死在代码中的方式。
