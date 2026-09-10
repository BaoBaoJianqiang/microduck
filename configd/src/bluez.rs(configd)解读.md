# `bluez.rs`（configd）解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 角色 | BlueZ 适配器管理——蓝牙设备发现、配对、信任、连接 |
| 平台 | Linux only（通过 D-Bus 操作 BlueZ） |
| 实现的 trait | `pad::Pads`（游戏手柄配对/忘记/状态） |

## 二、核心定位

- configd 的 BlueZ 模块负责**蓝牙配置操作**：
  - 发现设备。
  - 配对（pair）。
  - 信任（trust）。
  - 连接（connect）。
  - 删除绑定（remove device）。
- 与 btd 的 BlueZ 模块不同：
  - btd 的 BlueZ 模块负责**GATT 服务**（广告、characteristic、通知）。
  - configd 的 BlueZ 模块负责**设备管理**（配对、信任、连接）。
  - 两者通过 D-Bus 与同一个 `bluetoothd` 通信，但操作不同的接口。

## 三、为什么配对顺序是 connect → pair

### 历史踩坑

- 之前配对手柄意味着知道其 MAC 地址并按非显而易见的顺序运行三个 `bluetoothctl` 命令。
- **先 `pair` 返回 `AuthenticationCanceled`**。
- 正确顺序是 `connect` 在 `pair` 之前。
- 该顺序现在在一个地方——此模块——原因在旁边，而非在配置脚本的注释和某人的 shell 历史中。

### 为什么 connect 先于 pair

- 某些设备（特别是 LE 设备）在未连接时不响应配对请求。
- `connect` 建立物理链接，然后 `pair` 执行认证。
- 先 `pair` 会导致 BlueZ 尝试对未连接的设备配对，设备无响应，超时后返回 `AuthenticationCanceled`。

## 四、`BlueZ` 结构体

```rust
pub struct BlueZ {
    adapter: bluer::Adapter,
    // 内部状态
}
```

### `BlueZ::new() -> Result<Self, String>`

- 获取 BlueZ session。
- 获取默认适配器。
- 确保适配器通电。
- 返回 `BlueZ` 实例。

## 五、`Pads` trait 实现

### `status() -> PadResult<Vec<proto::Pad>>`

- 返回此机器人绑定的每个手柄，已连接的在前。
- 流程：
  1. 获取适配器的设备列表。
  2. 过滤出 `looks_like_a_gamepad` 的设备。
  3. 读取每个设备的 `Paired`、`Trusted`、`Connected` 属性。
  4. 排序：已连接的在前。

### `pair(mac: Option<&str>, timeout: Duration) -> PadResult<proto::PadPairResult>`

- 配对处于配对模式的任何手柄，或 `mac` 指定的那个。

#### 流程

1. **启动发现**：`adapter.set_discovery_filter(...)`，`adapter.start_discovery()`。
2. **等待设备**：在 `timeout` 内等待符合条件的设备出现。
3. **选择设备**：
   - 如果 `mac` 指定，选择该 MAC 的设备。
   - 否则选择未绑定的手柄（`looks_like_a_gamepad` 且 `!paired`）。
   - 多个未绑定 → 拒绝（`Ambiguous`）。
   - 无未绑定但有已绑定 → 返回已绑定的（幂等，修复丢失的 Trusted）。
   - 无设备 → 拒绝（`NotFound`）。
4. **连接**：`device.connect()`。
5. **配对**：`device.pair()`。
6. **信任**：`device.set_trusted(true)`。
   - **信任是承重的**：配对但未信任的手柄看起来正确但重启后不重连，这是此契约存在要防止的失败。
7. **停止发现**：`adapter.stop_discovery()`。
8. 返回 `PadPairResult::Paired { pad }`。

#### 拒绝 vs 错误

- **拒绝**（未找到、两个候选、BlueZ 说不）是携带 `Failed` 的 `Ok`。
- `Err` 保留给机制损坏（D-Bus 连接失败等）。

### `forget(mac: &str) -> PadResult<proto::PadForgetResult>`

- 删除绑定，使此手柄停止重连。
- 流程：
  1. 获取设备对象路径。
  2. `adapter.remove_device(device_path)`。
    - `RemoveDevice` 删除对象，不只是键，因此手柄根本不再可见，直到有东西重新发现它。
  3. 返回 `PadForgetResult { removed: true }`。
- 设备不存在 → `removed: false`（不是错误）。

## 六、设备过滤：`looks_like_a_gamepad`

- 与 `pad.rs` 中的函数相同（四信号启发式）：
  1. `icon == "input-gaming"`
  2. `class` 为外设 + 游戏手柄次要类别
  3. `appearance == 0x03C4`（Gamepad）
  4. 名称包含 `controller`/`gamepad`/`joystick`/`dualsense`/`dualshock`
- 在此模块中用于从 BlueZ 报告的所有设备中过滤出游戏手柄。

## 七、发现过滤器

### 为什么设置发现过滤器

- 不设置过滤器时，BlueZ 报告所有设备（耳机、手机、电视等）。
- 设置 `Transport: le` 和 `DuplicateData: false` 减少噪音。
- 但不能只过滤游戏手柄（BlueZ 不支持按设备类别过滤），因此仍需 `looks_like_a_gamepad` 后过滤。

### `DuplicateData: false`

- 抑制重复广告数据。
- 对于配对，这是正确的：只需要看到设备一次。
- 与 btd 的合唱模块不同（合唱需要 `DuplicateData: true`，因为有效载荷是变化的信号）。

## 八、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `pad.rs` | 定义 `Pads` trait，此模块实现它 |
| `main.rs` | 根据 `--fake-pads` 选择 `BlueZ` 或 `FakePads` |
| `btd` | 通过 `pad.pair`/`pad.forget` 转发手机应用的手柄配对请求 |
| `padd` | 非特权，读取已配对的手柄并发送意图；不配置 BlueZ |
| `bluetoothd` | 实际执行蓝牙操作的系统守护进程 |
| `BlueZ`（btd） | btd 的 GATT 服务模块，与此模块共享同一个 `bluetoothd` |

## 九、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **配对顺序固化** | connect → pair → trust，原因在代码旁，不在 shell 历史中 |
| **信任是承重的** | 配对后必须 set_trusted(true)，否则重启后不重连 |
| **未绑定优先** | 选择规则偏好未绑定的手柄，使已绑定的手柄不阻止配对新的 |
| **拒绝而非猜测** | 多个候选时拒绝，命名一个解决它 |
| **幂等** | 无新设备时返回已绑定的，修复丢失的 Trusted |
| **trait 抽象** | 实现 `Pads` trait，可被 `FakePads` 替换用于测试 |
#（注：内容由AI生成）
