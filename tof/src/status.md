# status.rs 文件解析

## 文件位置

`d:\microduck\tof\src\status.rs`

## 核心设计决策

`Status` 回答一个 viewer 不该猜的问题：是没有传感器，还是它还没产出第一帧？由传感器线程写、每个连接读——值很小，Mutex 即是全部机制。

- **始终接受订阅。** 无论传感器是否存在，`accepted: true`，因为订阅有效且传感器出现时帧就会到达。拒绝会让早一秒订阅的客户端永久放弃。
- **初始状态是"bringing the sensor up"。** 固件上传需数秒，在此窗口打开的 viewer 应看到原因而非空无。
- **中毒锁不致命。** `lock()` 用 `unwrap_or_else(|e| e.into_inner())` 恢复中毒的 Mutex，因为此处持锁时不可能 panic（无 `panic` 路径），把中毒当作致命会因一个状态字段拖垮守护进程。

## 类型

```rust
pub struct Status {
    hz: u8,
    inner: Mutex<Inner>,
}

struct Inner {
    sensor: Option<&'static str>,    // 已应答的代际名
    unavailable: Option<String>,      // 无传感器的原因
}
```

## 函数

- `new(hz)` — 初始 `sensor=None`，`unavailable=Some("bringing the sensor up")`
- `up(sensor: &'static str)` — `sensor=Some`，`unavailable=None`
- `down(why: &str)` — `sensor=None`，`unavailable=Some(why)`
- `result() -> proto::TofStreamResult` — 始终 `accepted: true`，附带 `sensor`/`unavailable`/`rows`/`cols`/`hz`

## 单元测试

- `the_status_names_which_of_the_three_it_is` — 验证三态（启动中、测距中、已消失）的字段组合正确，且消失后订阅仍 `accepted`。

## 关键摘要

status.rs 是 tofd 给订阅者的传感器状态快照：三态（启动中/测距中/无传感器）+ 始终接受订阅，让早到的客户端不会因传感器尚未就绪而被拒。Mutex 中毒被降级为可恢复，避免状态字段拖垮守护进程。
