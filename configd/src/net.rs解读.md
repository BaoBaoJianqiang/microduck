# `net.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 角色 | 网络配置——作为 trait + 假实现 |
| 平台 | 跨平台（trait 可测试，真实实现见 `nm.rs`） |

## 二、为什么 configd 拥有网络配置

### 网络配置是配置问题

- wifi 连接是关于机器人如何加入网络的*配置*问题。
- 它需要 root（修改 NetworkManager 的系统连接）。
- 当机器人本身不工作时它必须可回答（`architecture.md` §3.1）——从未见过网络的机器人无法通过该网络配置，因此 BLE 是唯一入口。

### 与 btd 的关系

- btd 通过 `net.*` 转发手机应用的网络配置请求到 configd。
- configd 实际执行 NetworkManager 操作。
- btd 不直接操作网络——它只是传输。

## 三、`Net` trait

```rust
#[async_trait]
pub trait Net: Send + Sync {
    async fn status(&self) -> NetResult<proto::NetStatus>;
    async fn scan(&self, timeout: Duration) -> NetResult<Vec<proto::WifiNetwork>>;
    async fn connect(&self, ssid: &str, psk: Option<&str>) -> NetResult<proto::NetConnectResult>;
    async fn forget(&self, ssid: &str) -> NetResult<proto::NetForgetResult>;
}
```

| 方法 | 说明 |
|---|---|
| `status()` | 当前网络状态：是否连接、SSID、IP 地址、信号强度 |
| `scan(timeout)` | 扫描可用 wifi 网络，返回 SSID 列表 + 信号强度 |
| `connect(ssid, psk)` | 连接到指定 wifi，可选密码 |
| `forget(ssid)` | 忘记指定 wifi 连接 |

### `NetResult<T> = Result<T, String>`

- 出了什么问题，用调用者可采取行动的术语。

### `connect()` 的拒绝 vs 错误

- **拒绝**（网络不存在、密码错误、连接超时）是携带失败信息的 `Ok`。
- `Err` 保留给机制损坏（NetworkManager 不响应等）。
- 与 `pad::Pads` 相同的拆分，调度器将其转为结果或 `INTERNAL_ERROR`。

## 四、常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_SCAN_TIMEOUT` | 10 秒 | 调用者未指定时的扫描超时 |
| `MAX_SCAN_TIMEOUT` | 30 秒 | 调用者可要求的最长扫描时间 |
| `CONNECT_TIMEOUT` | 30 秒 | 连接 wifi 的超时 |

### 扫描超时的设计

- 10 秒足够 NetworkManager 完成一次扫描并报告结果。
- 30 秒上限防止客户端要求过长的扫描（扫描期间 wifi 可能断开）。

## 五、`FakeNet`——内存中的假实现

### 用途

- 被测试和 `--fake-net` 使用。
- 这使得整个 `net.*` 表面——以及用真实硬件难以安排的失败场景——可从没有无线电的笔记本锻炼。

### `FakeState`

```rust
struct FakeState {
    known: Vec<proto::WifiNetwork>,  // 已知/已保存的网络
    available: Vec<proto::WifiNetwork>,  // 当前可扫描到的网络
    connected: Option<String>,  // 当前连接的 SSID
    ip: Option<Ipv4Addr>,  // 当前 IP 地址
}
```

### 构造函数

| 函数 | 说明 |
|---|---|
| `FakeNet::new()` | 默认状态：几个已知网络，无连接 |
| `FakeNet::with(state)` | 自定义状态 |

### `status()` 实现

- 返回当前连接状态、SSID、IP 地址。
- 未连接 → `NetStatus { connected: false, .. }`。

### `scan()` 实现

- 返回 `available` 列表。
- 可模拟扫描超时（如果状态中设置了标志）。

### `connect()` 实现

1. 检查 SSID 是否在 `available` 中。
   - 不在 → 拒绝（`NotFound`）。
2. 如果需要密码且未提供 → 拒绝（`AuthRequired`）。
3. 模拟连接延迟。
4. 设置 `connected = Some(ssid)`，分配假 IP。
5. 返回 `NetConnectResult::Connected { ssid, ip }`。

### `forget()` 实现

1. 从 `known` 中移除 SSID。
2. 如果当前连接的是该 SSID，断开。
3. 返回 `NetForgetResult { removed: true }`。
4. 不存在 → `removed: false`（不是错误）。

## 六、数据结构

### `proto::NetStatus`

```rust
pub struct NetStatus {
    pub connected: bool,
    pub ssid: Option<String>,
    pub ip4: Option<String>,  // IPv4 地址
    pub signal: Option<i32>,  // 信号强度 dBm
}
```

### `proto::WifiNetwork`

```rust
pub struct WifiNetwork {
    pub ssid: String,
    pub signal: i32,  // dBm
    pub secured: bool,  // 是否需要密码
}
```

### `proto::NetConnectResult`

```rust
pub enum NetConnectResult {
    Connected { ssid: String, ip: String },
    Failed { reason: NetConnectFailure, detail: Option<String> },
}

pub enum NetConnectFailure {
    NotFound,
    AuthRequired,
    AuthFailed,
    Timeout,
}
```

### `proto::NetForgetResult`

```rust
pub struct NetForgetResult {
    pub removed: bool,
}
```

## 七、测试

| 测试 | 验证内容 |
|---|---|
| `status_reports_unconnected_when_not_connected` | 未连接时 status 报告 connected: false |
| `scan_returns_available_networks` | 扫描返回可用网络列表 |
| `connect_to_available_network_succeeds` | 连接到可用网络成功，分配 IP |
| `connect_to_unavailable_network_fails` | 连接到不可用网络拒绝（NotFound） |
| `connect_to_secured_network_without_password_fails` | 无密码连接安全网络拒绝（AuthRequired） |
| `connect_with_wrong_password_fails` | 错误密码拒绝（AuthFailed） |
| `forget_removes_network` | 忘记网络移除它，断开当前连接 |
| `forget_nonexistent_network_is_not_error` | 忘记不存在的网络不是错误 |
| `status_after_connect_reports_ip` | 连接后 status 报告 IP 地址 |

## 八、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `nm.rs` | `Net` trait 的真实实现，通过 D-Bus 操作 NetworkManager |
| `main.rs` | 根据 `--fake-net` 选择 `NetworkManager` 或 `FakeNet` |
| `btd` | 通过 `net.*` 转发手机应用的网络配置请求 |
| `updaterd` | 更新后可能需要重新连接 wifi |
| `NetworkManager` | wifi 配置的实际执行者 |
| `store.rs` | 可能保存已知网络列表（虽然实际由 NetworkManager 管理） |

## 九、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **trait 抽象** | `Net` trait 使网络逻辑可在无无线电的笔记本上测试 |
| **假实现** | `FakeNet` 使整个 API 表面可测试，可安排困难场景 |
| **拒绝 vs 错误** | 网络操作失败是携带原因的 Ok，机制损坏才是 Err |
| **超时边界** | 扫描和连接都有超时，防止挂起 |
| **幂等** | 忘记不存在的网络不是错误 |
| **与 btd 分层** | btd 只传输，configd 实际操作网络 |
#（注：内容由AI生成）
