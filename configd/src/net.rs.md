# net.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/net.rs`
- **角色**：把「wifi」抽象为一个 `Net` trait，并提供两个平台无关实现：`UnavailableNet`（完全没有 wifi 栈时的降级后端）与 `FakeNet`（内存假实现）。生产实现是 Linux 下的 `nm::NetworkManager`。

## 2. 核心设计决策（模块文档）

- **NetworkManager 拥有凭据**（`architecture.md` §3）：`configd` 从不存 PSK，只把口令递给 NM；NM 以 root-only 持久化配置文件并自行重连。代码更少、安全更好、少一样要迁移的东西，且扛得住 `configd` 自身被重启、更新或回滚。
- trait 存在的理由与 `duck-control` 的 `RobotIo` 相同：整套测试要在没有硬件、没有网络、没有 D-Bus 的笔记本上跑；值得测的是围绕它的**分发与授权逻辑**，而不是 NM 本身。

## 3. 类型与 trait

### `pub type NetResult<T> = Result<T, String>`

错误用可付诸行动的字符串表达；`Err` 代表「机器坏了」，业务性拒绝用成功响应里的 `Failed` 表达（见 `main.rs` 的 `reply`）。

### `pub trait Net: Send + Sync`（`#[async_trait]`）

| 方法 | 说明 |
| --- | --- |
| `status() -> NetResult<NetStatusResult>` | 当前连接状态、SSID、信号、IP、MAC、接口名 |
| `scan() -> NetResult<NetScanResult>` | 扫描可见网络列表 |
| `connect(ssid, psk: Option<&str>) -> NetResult<ConnectResult>` | 加入网络并保存，使 NM 下次自行重连 |
| `forget(ssid) -> NetResult<ForgetResult>` | 删除已保存网络 |

## 4. `UnavailableNet`：每个调用都应答，且每个答案都说明原因

用于连不上系统总线时。**拒绝启动是显而易见的另一条路，但它是错的**：`configd` 还负责 `system.*`（`btd` 从中获取配对 PIN），若退出就会拖垮整个 BLE 配网，把「wifi 不可用」变成「机器人完全不可达」——而在这类板上手机是唯一入口。它同时是 `configd` 得以进入**启动恢复网**的条件：单元必须等待依赖而非退出，这样 `failed` 单元才意味着发布损坏而非板子损坏（`docs/design/boot-recovery-net.md`）。

字段 `reason`（总线错误）、`mac`、`iface`（后两者构造时为 `None`）。`reason` 被带进每次回复而不是只在启动时记一次日志，因为需要它的人正拿着手机、看不到 journal。

各方法行为：

- `status`：返回 `NetState::Unavailable`（**不是错误**）。「这里没有 wifi 栈」是可诊断的答案，协议中该状态正是为区分配置问题与网络问题而设；报错会让客户端无法分辨。
- `scan`：返回**空列表而非错误**，手机显示「未发现网络」（这是事实），与状态中给出的原因并列。
- `connect`：返回 `Err(reason)`。对没加入的网络谎报成功是谎言，默默不作为比说出问题更糟。
- `forget`：同样返回 `Err(reason)`。

## 5. `FakeNet`：仅存在于内存的 wifi 栈

供所有测试与 `--fake-net` 使用，使整个 `net.*` 表面（包括对错口令这类在真机上难以制造的失败）可在笔记本端到端演练。

### 内部状态 `FakeState`

- `visible: Vec<(proto::Network, Option<String>)>`：无线电能看到的网络，以及每个网络实际要求的口令（`None` 表示开放）。
- `saved: Vec<String>`：已保存的 SSID。
- `connected: Option<String>`：当前连接的 SSID。
- 整体由 `tokio::sync::Mutex` 保护。

### `new()`：范围内两个网络

| SSID | 信号 | 安全类型 | 正确口令 |
| --- | --- | --- | --- |
| `Pollen` | 82 | `WpaPsk` | `correct-key` |
| `Cafe` | 41 | `Open` | 无 |

另有 `with_visible(...)` 构造器与 `Default`（等于 `new`）。

### trait 实现行为

- `status`：已连接时返回 `Connected`，附带信号、固定 IP `192.168.50.63`、MAC `50:37:cd:16:1b:92`、接口 `wlan0`；未连接返回 `Disconnected`（MAC/接口仍给出）。
- `scan`：把 `visible` 映射为协议网络并按 `saved` 标记，**按信号降序排列**。
- `connect`：
  - SSID 不在 `visible` → `Failed { reason: NotFound }`；
  - 企业网（`Enterprise`）→ `Failed { Unsupported, "802.1X needs a certificate flow..." }`；
  - 需要口令但未给 → `Failed { Unsupported, "this network needs a passphrase" }`；
  - 口令不匹配 → `Failed { BadKey }`；
  - 成功：置 `connected`、加入 `saved`（不重复），返回 `Connected { ssid, ip4: 192.168.50.63 }`。
- `forget`：从 `saved` 移除，若正连着则断开；`removed` 反映长度是否变化。

## 6. 单元测试说明

| 测试 | 验证内容 |
| --- | --- |
| `an_unavailable_stack_answers_everything_and_joins_nothing` | 降级栈 `status` 为 `Unavailable`、`scan` 为空、`connect`/`forget` 报错且错误中含 `D-Bus`（理由必须到达无 journal 的客户端） |
| `a_wrong_key_is_reported_as_a_wrong_key` | 错口令明确报 `BadKey` |
| `a_missing_key_is_not_a_wrong_key` | 加密网络未给口令报 `Unsupported`（客户端应提示输入密码，而非「密码错误」） |
| `an_unknown_ssid_is_not_found` | 看不见的 SSID 报 `NotFound` |
| `a_corrected_passphrase_replaces_the_bad_attempt` | **钉住重配网契约**：先错后对，失败尝试不留存配置；改正后只有一份配置——一次 `forget` 即 removed，第二次 `removed=false`。对应 NM `AddAndActivateConnection` 总会添加、且允许同名配置的真实坑：错误配置若残留，会在以后启动时被自动连接 |
| `connecting_stores_the_network_and_forgetting_removes_it` | 完整配网弧线：扫描初始均未保存 → 连接成功、有地址、标记 saved → forget 后变 `Disconnected`，再次 forget 不是错误 |
| `an_open_network_joins_without_a_key` | 开放网络无需口令即可加入 |
| `scan_results_are_sorted_by_signal` | 扫描结果信号非递增排序 |

## 7. 要点小结

- 「wifi」是 trait，生产后端 NM、降级后端 `UnavailableNet`、测试后端 `FakeNet` 可互换。
- 后端不可用时**降级而非退出**：只读调用照实回答、加入类调用带理由报错，保住 BLE 配网与启动恢复网资格。
- 假栈内置 `Pollen`(WPA2/`correct-key`/82) 与 `Cafe`(开放/41)，固定 IP/MAC/接口，并精确区分 `NotFound`/`Unsupported`/`BadKey`。
- 重配网必须**替换而非累积**坏配置，这一契约由测试钉死。
