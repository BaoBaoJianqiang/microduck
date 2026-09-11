# `perf_probe.rs` 解读

## 概述

这不是测试——是一个**探针（probe）**。运行方式：

```bash
cargo test -p kinematics --release --test perf_probe -- --ignored --nocapture
```

它计时两种工作负载：
- **里程计形状**（双脚，每个 tick）
- **视觉形状**（头部摄像头）

数字输出到终端，不进断言：CI 中的挂钟时间阈值是不稳定性，而非覆盖率。

---

## 设计哲学：为什么不用 criterion 等基准框架

> Numbers land in the terminal, not in an assertion: wall-clock thresholds in CI are flakiness, not coverage.

**核心决定：不用 criterion/bencher 等基准框架，不设 CI 阈值。**

理由：
- **CI 环境噪声大**：共享 CI runner 上的时间测量受邻居负载影响，设阈值必然 flaky
- **基准的目的是发现回归**，而非阻止合并——人看数字比自动阈值可靠
- **`#[ignore]`**：默认 `cargo test` 不运行，只在显式 `--ignored` 时运行
- **`--release`**：必须在 release 模式下运行，debug 模式的优化被关闭，数字无意义
- **`--nocapture`**：println 输出到终端而非被测试框架捕获

---

## 探针 1：`time_site_pose`

```rust
#[test]
#[ignore = "perf probe, run manually with --release --nocapture"]
fn time_site_pose() {
    let model = Model::alpha();
    let feet = [
        model.site("left_foot").expect("site"),
        model.site("right_foot").expect("site"),
    ];
    let camera = model.site("head_camera").expect("site");

    const ITERS: u32 = 1_000_000;
    let mut angles = vec![0.0f64; model.num_joints()];
```

### 两个工作负载

| 工作负载 | 查询的 site | 对应什么 |
|----------|------------|----------|
| `both feet` | left_foot + right_foot | 里程计：每个 tick 查双脚位置 |
| `head camera` | head_camera | 视觉：查摄像头位姿用于 ToF 重投影 |

### 100 万次迭代

```rust
for (label, sites) in [("both feet", &feet[..]), ("head camera", &[camera][..])] {
    let start = Instant::now();
    let mut acc = 0.0f64;
    for i in 0..ITERS {
        // Vary the input so nothing folds away.
        angles[0] = f64::from(i % 100) * 0.001;
        for &site in sites {
            acc += black_box(model.site_pose(site, black_box(&angles))).pos[2];
        }
    }
    let per = start.elapsed().as_nanos() as f64 / f64::from(ITERS);
    println!("{label}: {per:.0} ns/query-set (acc {acc:.3})");
}
```

#### 为什么改变输入

```rust
angles[0] = f64::from(i % 100) * 0.001;
```

**Vary the input so nothing folds away.** 如果输入恒定，编译器可能把 100 万次相同的计算优化成一次。改变第一个关节角度确保每次迭代的计算路径不同，编译器不能折叠。

#### `black_box` 的作用

```rust
black_box(model.site_pose(site, black_box(&angles))).pos[2]
```

`std::hint::black_box` 告诉编译器这个值"可能被外部观察到"，阻止它：
- 优化掉未使用的计算
- 把结果提前计算（常量折叠）
- 把循环展开成已知值

`black_box(&angles)` 确保编译器不知道输入内容，`black_box(...).pos[2]` 确保结果被"使用"。

#### 输出

```
both feet: XXX ns/query-set (acc YYY.YYY)
head camera: XXX ns/query-set (acc YYY.YYY)
```

- **ns/query-set**：每次查询集（双脚或单摄像头）的纳秒数
- **acc**：累加器，确保结果被使用（如果编译器发现 acc 从未被读取，可能优化掉整个循环）

---

## 探针 2：`time_tof_reprojection`

```rust
#[test]
#[ignore = "perf probe, run manually with --release --nocapture"]
fn time_tof_reprojection() {
    use kinematics::tof::Reprojector;

    let rp = Reprojector::alpha();
    // Worst case: all 64 zones returned something.
    let ranges = [Some(1.2f64); kinematics::tof::ROWS * kinematics::tof::COLS];
```

### ToF 重投影

- **`Reprojector::alpha()`**：ToF（飞行时间）传感器重投影器
- **64 个 zone**：最坏情况——所有 64 个区域都返回了距离
- **`ranges`**：所有 zone 都有值（`Some(1.2)`），模拟最坏情况

### 10 万次迭代

```rust
const ITERS: u32 = 100_000;
let mut acc = 0usize;
let start = Instant::now();
for i in 0..ITERS {
    let head = [0.0, f64::from(i % 100) * 0.001, 0.0, 0.0];
    let zones = rp.project(
        black_box(&ranges),
        black_box(&head),
        &kinematics::tof::Posture::default(),
    );
    acc += zones
        .iter()
        .filter(|z| matches!(z, kinematics::tof::Zone::Hit { .. }))
        .count();
}
let per = start.elapsed().as_nanos() as f64 / f64::from(ITERS);
println!("full-frame reprojection: {per:.0} ns/frame (acc {acc})");
```

#### 为什么 10 万次而非 100 万次

ToF 重投影比单次 site 查询重得多（64 个 zone 的完整重投影），10 万次已经足够稳定。100 万次会让运行时间过长。

#### 输出

```
full-frame reprojection: XXX ns/frame (acc YYY)
```

- **ns/frame**：每帧完整重投影的纳秒数
- **acc**：命中的 zone 总数（确保结果被使用）

---

## 设计要点

### 1. `#[ignore]` 标记

```rust
#[ignore = "perf probe, run manually with --release --nocapture"]
```

- 默认 `cargo test` 跳过这些测试
- 显式 `cargo test -- --ignored` 才运行
- 这确保日常开发的 `cargo test` 快速通过，不会因为性能探针拖慢

### 2. 不设 CI 阈值

> wall-clock thresholds in CI are flakiness, not coverage

CI 上的时间测量受太多因素影响（邻居负载、CPU 频率、缓存状态），设阈值必然 flaky。性能回归应该由人看数字发现，而非自动阻止合并。

### 3. `black_box` 防止优化

性能探针最大的陷阱是编译器把整个循环优化掉。`black_box` 确保：
- 输入不被提前计算
- 输出不被丢弃
- 循环不被折叠

### 4. 最坏情况测量

- ToF 重投影：64 个 zone 全部有值（最坏情况）
- site 查询：双脚同时查询（里程计的实际负载）

测量的是**最坏情况**而非平均情况，因为控制循环需要在最坏情况下也能在预算内完成。

### 5. 改变输入防止常量折叠

```rust
angles[0] = f64::from(i % 100) * 0.001;
```

如果输入恒定，编译器可能把 100 万次相同的计算优化成一次。改变第一个关节角度确保每次迭代的计算路径不同。

---

## 与其他文件的关系

- **`kinematics::Model`**：被测对象，提供 `alpha()`、`site()`、`site_pose()`、`num_joints()`
- **`kinematics::tof::Reprojector`**：ToF 重投影器，提供 `alpha()`、`project()`
- **`kinematics::tof::Posture`**：ToF 重投影的姿态参数
- **`kinematics::tof::Zone`**：ToF zone 结果（`Hit` 或 miss）
- **`fk_against_mujoco.rs`**：同目录的正确性测试文件

---

## 关键踩坑点总结

1. **不用 criterion 等基准框架**：CI 上的挂钟时间阈值是不稳定性而非覆盖率。人看数字比自动阈值可靠。

2. **`#[ignore]` 标记**：性能探针默认不运行，只在显式 `--ignored` 时运行。日常 `cargo test` 快速通过。

3. **必须 `--release`**：debug 模式优化被关闭，数字无意义。必须在 release 模式下运行。

4. **`black_box` 是必须的**：没有它，编译器可能把整个循环优化掉，测量到的是 0 ns。

5. **改变输入防止常量折叠**：恒定输入会让编译器把 100 万次计算折叠成一次。

6. **最坏情况测量**：ToF 重投影测 64 个 zone 全部有值的情况，因为控制循环需要在最坏情况下也能在预算内完成。

7. **输出到终端不进断言**：数字是给人看的，不是给 CI 自动判断的。`--nocapture` 确保 println 不被测试框架吞掉。
#（注：内容由AI生成）
