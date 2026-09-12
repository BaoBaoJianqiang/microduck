# perf_probe.rs 文件解析

## 文件位置

`d:\microduck\kinematics\tests\perf_probe.rs`

## 核心设计决策

不是测试——是探针。运行方式：

```text
cargo test -p kinematics --release --test perf_probe -- --ignored --nocapture
```

测量里程计形工作负载（双脚，每 tick）与视觉形工作负载（头相机）的耗时。数字输出到终端而非断言：CI 中的墙钟阈值是抖动，不是覆盖率。

## 探针函数

### `time_site_pose`

- 取 `left_foot`、`right_foot` 与 `head_camera` site
- 1,000,000 次迭代，每迭代变 `angles[0]` 防止被优化掉
- 分别测"双脚"与"头相机"的 ns/query-set

### `time_tof_reprojection`

- 最坏情况：全部 64 个 zone 都有回波
- 100,000 次迭代，变 `head_pitch`
- 测 full-frame reprojection 的 ns/frame

## 关键摘要

`perf_probe.rs` 是手动运行的性能探针（`#[ignore]`），验证编译后的 FK 链查询与 ToF 重投影是否满足 50Hz 控制循环的预算。不做断言——性能数字只供参考，不进 CI。
