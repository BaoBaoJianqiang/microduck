# `advwatch.rs` 解读

## 概述

`advwatch` 是 `duckctl` 的一个示例程序（`cargo run -p duckctl --example advwatch -- <robot-name>`），用于**持续监控 BLE 广告的实际到达率**，而非 `duckctl` 默认的 8 秒一次性扫描。

核心问题：`duckctl` 扫描 8 秒后要么找到机器人要么找不到，这使得**广告间隔慢的机器人看起来像坏了**。`advwatch` 通过持续观察 2 分钟并打印到达模式，将"第二次尝试才找到"转化为可量化的数字。

### 三种故障的区分

这个工具存在的唯一理由是区分三种表面上相同的故障：

1. **机器人没有在广告** — 完全收不到
2. **机器人广告太稀疏，扫描窗口抓不到** — 能收到但间隔很长
3. **客户端有问题** — 同一房间其他设备正常但机器人异常

**关键设计**：同时测量范围内的**所有其他设备**，这使得结论是确定性的——如果一个机器人比信号弱 55 dB 的信标还少听到 10 倍，那就不是距离或干扰问题。

---

## 关键常量

```rust
const WATCH: Duration = Duration::from_secs(120);
```

- **2 分钟观察窗口**：因为要测量的是*静默期*，窗口必须足够长以包含若干个最坏情况的静默，才能对其发生频率做出有意义的统计。

---

## 核心数据结构

### `Device`

```rust
struct Device {
    arrivals: usize,   // 广告到达次数（去重后）
    last: f64,         // 上次到达时间（秒，相对于 start）
    name: Option<String>,  // 设备本地名称
    rssi: Option<i16>,     // 信号强度
}
```

- `arrivals` 不是事件计数，而是**去重后的广告到达次数**（见下方 0.2 秒去重逻辑）
- `last` 初始为 `-1.0`，确保第一次到达一定被计入

---

## 主流程详解

### 1. 初始化与扫描启动

```rust
let wanted = std::env::args().nth(1).unwrap_or("radxa-zero3".to_owned());
```

- 命令行第一个参数指定目标机器人名称，默认为 `radxa-zero3`

```rust
let manager = Manager::new().await?;
let adapter = manager.adapters().await?.into_iter().next().ok_or("no Bluetooth adapter")?;
let mut events = adapter.events().await?;
adapter.start_scan(ScanFilter::default()).await?;
```

- 使用 `btleplug` 库获取第一个可用的蓝牙适配器
- `ScanFilter::default()` — **不过滤任何设备**，这是故意的：需要同时测量所有设备作为对照
- 订阅适配器事件流

### 2. 事件循环（核心）

```rust
loop {
    let left = WATCH.saturating_sub(start.elapsed());
    if left.is_zero() { break; }
    let Ok(Some(event)) = tokio::time::timeout(left, events.next()).await else {
        break;
    };
    total += 1;
    // ...
}
```

- 每次迭代计算剩余时间，用 `tokio::time::timeout` 等待下一个事件
- 超时或事件流结束即退出循环
- `total` 统计所有设备的原始事件数（含重复）

#### 事件类型提取

```rust
let id = match &event {
    CentralEvent::DeviceDiscovered(id)
    | CentralEvent::DeviceUpdated(id)
    | CentralEvent::DeviceConnected(id)
    | CentralEvent::DeviceDisconnected(id) => id.clone(),
    CentralEvent::ServicesAdvertisement { id, .. }
    | CentralEvent::ManufacturerDataAdvertisement { id, .. }
    | CentralEvent::ServiceDataAdvertisement { id, .. } => id.clone(),
    _ => continue,
};
```

- 从所有包含设备 ID 的事件类型中提取 ID
- 忽略不携带设备 ID 的事件（如 `Manager` 事件等）

#### 获取设备属性

```rust
let Ok(peripheral) = adapter.peripheral(&id).await else { continue; };
let Some(properties) = peripheral.properties().await? else { continue; };
```

- 通过 ID 获取外围设备句柄和属性
- 属性在 btleplug 中是**累积的**：一次看到的名称或 UUID 会持续标识该外围设备，所以首次发现后 ID 就足够了

### 3. 广告到达去重（关键设计）

```rust
let now = start.elapsed().as_secs_f64();
let entry = per_device.entry(id.to_string()).or_insert(Device {
    arrivals: 0,
    last: -1.0,
    name: properties.local_name.clone(),
    rssi: properties.rssi,
});
if now - entry.last > 0.2 {
    entry.arrivals += 1;
    entry.last = now;
}
```

**为什么需要去重？** 一次广告接收会触发多个 btleplug 事件（`DeviceUpdated`、`ServicesAdvertisement`、`ManufacturerDataAdvertisement` 等）在同一瞬间到达，如果直接计数事件数会高估 3-4 倍。

**0.2 秒阈值**：同一设备在 200ms 内的多个事件视为同一次广告到达。这是基于 BLE 广告间隔通常在 100ms-1s 之间的合理选择。

```rust
if entry.name.is_none() {
    entry.name = properties.local_name.clone();
}
if properties.rssi.is_some() {
    entry.rssi = properties.rssi;
}
```

- 名称和 RSSI 是累积更新的：首次没有名称的设备后续可能获得，RSSI 取最新值

### 4. 机器人识别

```rust
let is_robot = properties.local_name.as_deref() == Some(wanted.as_str())
    || properties.services.contains(&SERVICE_UUID)
    || robot.as_deref() == Some(id.to_string().as_str());
```

三种方式识别目标机器人（满足任一即可）：
1. **本地名称匹配** — `local_name == wanted`
2. **服务 UUID 匹配** — 广告中包含 `btd::gatt::SERVICE_UUID`（机器人的 GATT 服务 UUID）
3. **已识别的机器人 ID** — 一旦某个 ID 被确认为机器人，后续该 ID 的事件直接匹配

```rust
if robot.is_none() {
    robot = Some(id.to_string());
    eprintln!("  first sighting at {:.1}s: id={} name={:?} services={}", ...);
}
hits.push(start.elapsed().as_secs_f64());
```

- 首次发现机器人时打印发现时间、ID、名称和服务数量到 stderr
- `hits` 向量记录机器人每次广告到达的精确时间戳

---

## 输出报告详解

观察结束后（`stop_scan`），程序输出多维度的诊断报告。

### 1. 总览

```rust
println!("\n{total} events from all devices, {} from the robot", hits.len());
```

- 所有设备的原始事件总数 vs 机器人的广告到达次数

### 2. 同房间设备排名（对照组）

```rust
let mut ranked: Vec<(&String, &Device)> = per_device.iter().collect();
ranked.sort_by_key(|device| std::cmp::Reverse(device.1.arrivals));
println!("heard most often in the same room, over {WATCH:?}:");
for (id, device) in ranked.iter().take(8) {
    // 打印到达次数、RSSI、名称，机器人标注 <-- the robot
}
```

- 按到达次数降序排列，取前 8 名
- 这是**对照组**：机器人的到达率与同房间其他设备直接比较
- 如果机器人排名靠后但 RSSI 很强，说明是广告间隔问题而非信号问题

### 3. 机器人专项统计

```rust
println!("  the robot: {} arrivals, {} — one every {:.1}s",
    device.arrivals, rssi, WATCH.as_secs_f64() / device.arrivals.max(1) as f64);
```

- 平均到达间隔 = 总观察时间 / 到达次数
- `arrivals.max(1)` 防止除零

```rust
let rank = ranked.iter().position(|(id, _)| *id == robot);
println!("the robot ranks {} of {} devices heard", rank.map(|r| r + 1).unwrap_or(0), ranked.len());
```

- 机器人在所有听到设备中的排名

### 4. 完全未发现的情况

```rust
if hits.is_empty() {
    println!("the robot was never heard from in {WATCH:?}");
    return Ok(());
}
```

- 2 分钟内完全没听到机器人，直接退出（后续时间线和间隙分析无意义）

### 5. 每秒时间线

```rust
let seconds = WATCH.as_secs() as usize;
let mut timeline = vec![b'.'; seconds];
for hit in &hits {
    let slot = (*hit as usize).min(seconds - 1);
    timeline[slot] = b'#';
}
println!("{}", String::from_utf8(timeline).unwrap());
```

- **每秒一个字符**：`#` 表示该秒至少收到一次广告，`.` 表示静默
- 120 个字符的时间线直观展示广告到达的分布模式
- 连续 8 个 `.` 就是一次会失败的 `duckctl` 扫描

### 6. 间隙分析

```rust
let mut gaps: Vec<f64> = Vec::new();
let mut previous = 0.0;
for hit in &hits {
    gaps.push(hit - previous);
    previous = *hit;
}
gaps.push(WATCH.as_secs_f64() - previous);
```

- 计算所有相邻到达之间的间隙，包括从 0 到第一次到达、最后一次到观察结束
- 降序排列后输出最坏的 6 个间隙

```rust
gaps.sort_by(|a, b| b.partial_cmp(a).unwrap());
println!("gaps between reports: worst {:.1}s, then {:?}", gaps[0], ...);
```

### 7. 最小间隙（广告间隔推断）

```rust
let mut closest = gaps.clone();
closest.sort_by(|a, b| a.partial_cmp(b).unwrap());
println!("closest reports: {:?}", closest.iter().take(8).map(|g| format!("{:.0}ms", g * 1000.0)).collect());
```

- **最小间隙就是广告间隔**：两次连续到达不可能比机器人的广告间隔更近
- 从到达数据中推断广告间隔不需要任何权限，而询问控制器需要板子上的 root 权限
- 这是一个零成本的旁路径断

### 8. 间隙直方图

```rust
let buckets = [0.2, 0.5, 1.0, 2.0, 4.0, 8.0, f64::MAX];
let mut counts = vec![0usize; buckets.len()];
for gap in &gaps {
    let slot = buckets.iter().position(|b| gap < b).unwrap();
    counts[slot] += 1;
}
println!("gap histogram  <200ms:{} <500ms:{} <1s:{} <2s:{} <4s:{} <8s:{} 8s+:{}", ...);
```

- 7 个桶的间隙分布直方图
- 直观展示间隙的整体分布模式

### 9. 超过扫描窗口的间隙

```rust
let over_scan = gaps.iter().filter(|g| **g >= 8.0).count();
println!("{over_scan} gap(s) of 8s or more — each one is a `duckctl` run that would report no robot");
```

- **8 秒是 `duckctl` 的扫描窗口**
- 任何 ≥8s 的间隙意味着在该窗口内启动的 `duckctl` 扫描会报告"找不到机器人"
- 这是将测量结果直接映射到用户体验的关键指标

---

## 设计要点与踩坑点

### 1. 到达次数 ≠ 事件计数

一次 BLE 广告接收会触发 btleplug 的多个事件（`DeviceUpdated`、`ServicesAdvertisement`、`ManufacturerDataAdvertisement` 等），直接计数事件会高估 3-4 倍。0.2 秒去重窗口解决了这个问题。

### 2. 对照组是结论确定性的关键

只测量机器人本身无法区分"广告慢"和"环境差"。同时测量所有设备后，如果机器人比弱信号的信标还少听到很多倍，就能排除距离和干扰因素。

### 3. 属性累积特性

btleplug 的 `properties` 是累积的：一次看到的名称或 UUID 会持续标识该外围设备。这意味着首次发现后，仅凭 ID 就能持续识别设备，不需要每次都匹配名称。

### 4. 三种机器人识别方式的冗余

名称匹配、服务 UUID 匹配、已识别 ID 匹配——三种方式互为冗余。有些设备可能不广播名称但广播服务 UUID，反之亦然。一旦 ID 被确认，后续直接匹配 ID 避免了属性缺失导致的漏检。

### 5. 2 分钟窗口的选择

窗口必须足够长以包含若干个最坏情况的静默。如果窗口只有 30 秒，可能只包含 1-2 个长间隙，无法对其频率做出统计判断。2 分钟（120 秒）在可接受的等待时间和统计意义之间取得平衡。

### 6. 最小间隙推断广告间隔

这是一个巧妙的零成本旁路径断：两次连续到达的最小间隔就是广告间隔的下界，不需要 root 权限读取控制器配置。

---

## 与其他模块的关系

- **`btd::gatt::SERVICE_UUID`**：引用 btd crate 中定义的 GATT 服务 UUID，用于通过服务 UUID 识别机器人
- **`duckctl`**：作为 duckctl 的 example 编译运行，与 `duckctl scan` 命令形成互补（scan 是一次性 8 秒扫描，advwatch 是持续 2 分钟监控）
- **`btleplug`**：跨平台 BLE 库，提供适配器管理、扫描、事件流等功能

---

## 测试要点

本文件是一个 `main` 函数示例，没有 `#[cfg(test)]` 模块。其验证方式是**在真实 BLE 环境中运行**，通过观察输出的时间线、间隙分布和设备排名来诊断广告到达问题。

关键可验证输出：
- 时间线中 `#` 和 `.` 的分布模式
- 间隙直方图中各桶的计数
- ≥8s 间隙的数量（直接对应 `duckctl` 失败率）
- 机器人在同房间设备中的排名
#（注：内容由AI生成）
