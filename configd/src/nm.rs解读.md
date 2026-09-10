# `nm.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 角色 | NetworkManager D-Bus 客户端——`Net` trait 的真实实现 |
| 平台 | Linux only（通过 D-Bus 操作 NetworkManager） |
| 实现的 trait | `net::Net` |

## 二、核心定位

- `nm.rs` 是 `net::Net` trait 的真实实现。
- 通过系统 D-Bus 与 NetworkManager 通信。
- 操作的接口：
  - `org.freedesktop.NetworkManager`（管理器）
  - `org.freedesktop.NetworkManager.Device.Wireless`（无线设备）
  - `org.freedesktop.NetworkManager.AccessPoint`（接入点）
  - `org.freedesktop.NetworkManager.Settings.Connection`（连接配置）
  - `org.freedesktop.NetworkManager.AgentManager`（密码代理）

## 三、为什么用 NetworkManager 而非直接操作 wpa_supplicant

### NetworkManager 的优势

- 管理无线设备的电源、扫描、连接、重连。
- 持久化连接配置（`/etc/NetworkManager/system-connections/`）。
- 处理多 SSID 优先级、自动重连。
- D-Bus API 稳定，比直接操作 wpa_supplicant 简单。

### 为什么需要 root

- 修改系统连接（`/etc/NetworkManager/system-connections/`）需要 root。
- NetworkManager 的 `AddConnection`、`ActivateConnection` 等方法对非 root 受限。
- 这是 configd 以 root 运行的原因之一。

## 四、`NetworkManager` 结构体

```rust
pub struct NetworkManager {
    bus: zbus::Connection,
    // 内部状态
}
```

### `NetworkManager::new() -> Result<Self, String>`

- 连接系统 D-Bus。
- 验证 NetworkManager 服务可用。
- 返回 `NetworkManager` 实例。

## 五、`Net` trait 实现

### `status() -> NetResult<proto::NetStatus>`

- 查询当前网络状态。

#### 流程

1. 获取无线设备列表（`GetDevices` + 过滤 `DeviceType == 2`（Wifi））。
2. 获取第一个无线设备的 `ActiveConnection`。
3. 如果有活动连接：
   - 读取连接的 `Id`（SSID）。
   - 读取设备的 `Ip4Config`，获取 IP 地址。
   - 读取接入点的 `Strength`（信号强度）。
4. 无活动连接 → `connected: false`。

#### IP 地址处理

- NetworkManager 的 `Ip4Config.Addresses` 是 `(u32, u32, u32)` 数组（地址、前缀、网关），网络字节序。
- 转换为 `Ipv4Addr`。
- 只返回第一个地址（通常只有一个）。

### `scan(timeout: Duration) -> NetResult<Vec<proto::WifiNetwork>>`

- 扫描可用 wifi 网络。

#### 流程

1. 获取无线设备。
2. 调用 `RequestScan()`（触发扫描）。
3. 等待 `AccessPointAdded` 信号或超时。
4. 获取设备的 `AccessPoints` 列表。
5. 对每个接入点：
   - 读取 `Ssid`（字节数组，转换为字符串）。
   - 读取 `Strength`（0-100，转换为 dBm：`strength / 2 - 100`）。
   - 读取 `WpaFlags`/`RsnFlags`（判断是否安全）。
   - 读取 `Frequency`（可选，用于区分 2.4G/5G）。
6. 按信号强度排序（强在前）。
7. 去重（同一 SSID 可能有多个接入点，保留最强的）。

#### 扫描的注意事项

- `RequestScan` 是异步的，需要等待信号或轮询。
- 扫描期间 wifi 连接可能短暂中断（扫描离开当前信道）。
- NetworkManager 有扫描节流（频繁扫描会被延迟）。

### `connect(ssid: &str, psk: Option<&str>) -> NetResult<proto::NetConnectResult>`

- 连接到指定 wifi。

#### 流程

1. 检查 SSID 是否在最近扫描结果中。
   - 不在 → 拒绝（`NotFound`）。
2. 查找是否已有该 SSID 的保存连接。
   - 有 → 使用现有连接（`ActivateConnection`）。
   - 无 → 创建新连接（`AddConnection`）。
3. 创建连接配置：
   - `connection.type = "802-11-wireless"`
   - `connection.id = ssid`
   - `802-11-wireless.ssid = ssid`（字节）
   - `802-11-wireless.mode = "infrastructure"`
   - 如果有密码：
     - `802-11-wireless-security.key-mgmt = "wpa-psk"`
     - `802-11-wireless-security.psk = psk`
4. `ActivateConnection` 激活连接。
5. 等待连接成功或超时（`CONNECT_TIMEOUT`）。
   - 监听 `StateChanged` 信号。
   - 状态变为 `Activated` → 成功。
   - 状态变为 `Failed` 或超时 → 失败。
6. 成功 → 读取 IP 地址，返回 `Connected { ssid, ip }`。
7. 失败 → 根据原因返回 `AuthFailed` 或 `Timeout`。

#### 密码处理

- PSK 必须是 8-63 个 ASCII 字符或 64 个十六进制字符。
- 太短/太长 → 拒绝（`AuthFailed`，detail 说明）。
- 密码以明文存储在 NetworkManager 的系统连接中（`/etc/NetworkManager/system-connections/`，权限 0600）。

### `forget(ssid: &str) -> NetResult<proto::NetForgetResult>`

- 忘记指定 wifi 连接。

#### 流程

1. 查找该 SSID 的保存连接（`ListConnections` + 过滤 `id == ssid`）。
2. 找到 → `Delete()` 删除连接。
   - 如果当前活动连接是该 SSID，NetworkManager 会自动断开。
3. 返回 `NetForgetResult { removed: true }`。
4. 未找到 → `removed: false`（不是错误）。

## 六、D-Bus 调用模式

### 通用模式

```rust
bus.call_method(
    Some("org.freedesktop.NetworkManager"),
    "/org/freedesktop/NetworkManager",
    Some("org.freedesktop.NetworkManager"),
    "MethodName",
    &(args),
).await?
```

### 属性读取

```rust
bus.call_method(
    Some("org.freedesktop.NetworkManager"),
    &path,
    Some("org.freedesktop.DBus.Properties"),
    "Get",
    &("interface.name", "PropertyName"),
).await?
```

### 信号监听

- 使用 `zbus` 的 `receive_signal` 或 `MatchRule` 监听 D-Bus 信号。
- 用于等待扫描完成、连接状态变化。

## 七、错误处理

### NetworkManager 错误映射

| NetworkManager 错误 | 映射到 |
|---|---|
| 接入点不存在 | `NotFound` |
| 认证失败 | `AuthFailed` |
| 连接超时 | `Timeout` |
| D-Bus 调用失败 | `Err(String)`（机制损坏） |

### 超时

- 所有 D-Bus 调用都有隐式超时（zbus 默认）。
- 连接等待有显式 `CONNECT_TIMEOUT`。
- 扫描等待有显式 `timeout` 参数。

## 八、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `net.rs` | 定义 `Net` trait，此模块实现它 |
| `main.rs` | 根据 `--fake-net` 选择 `NetworkManager` 或 `FakeNet` |
| `btd` | 通过 `net.*` 转发手机应用的网络配置请求 |
| `updaterd` | 更新后可能需要重新连接 wifi |
| `store.rs` | 不直接保存 wifi 密码（由 NetworkManager 管理） |
| `NetworkManager` | 实际执行 wifi 操作的系统守护进程 |
| `wpa_supplicant` | NetworkManager 后端，不直接操作 |

## 九、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **trait 实现** | 实现 `Net` trait，可被 `FakeNet` 替换用于测试 |
| **D-Bus 通信** | 通过系统 D-Bus 与 NetworkManager 通信，不直接操作 wpa_supplicant |
| **持久化连接** | 利用 NetworkManager 的连接持久化，不自己管理密码存储 |
| **超时边界** | 扫描和连接都有超时，防止挂起 |
| **错误映射** | NetworkManager 错误映射为调用者可理解的原因 |
| **信号监听** | 使用 D-Bus 信号等待异步操作（扫描、连接），而非轮询 |
| **root 需求** | 修改系统连接需要 root，这是 configd 以 root 运行的原因之一 |
#（注：内容由AI生成）
