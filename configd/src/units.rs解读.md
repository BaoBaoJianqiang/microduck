# `units.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 172 行 |
| 角色 | 查询哪些守护进程在运行，以及每个运行的是哪个版本 |
| 平台 | Linux 运行（查询 systemd）；非 Linux 回退为 Unknown |

## 二、起源：从"padd 在运行吗"到全系统状态

### 最初的问题

- 开始于"padd 在运行吗？"，与手柄状态一起报告。
- 原因：连接的手柄 + 死亡的 padd 是看起来像硬件正常的故障：
  - 控制器的灯亮着。
  - 机器人忽略它。
  - 两个地方都没有说明原因。

### 扩展到所有守护进程

- 这个论点从不特定于 padd：
  - 死亡的 btd = 手机看不到的机器人，同样的沉默。
- 因此它为**一个版本管理的每个单元**回答。

## 三、两个信息源，各回答不同的问题

### 1. systemd：单元是否在运行

- 这些单元由 systemd 启动、停止和重启，因此 systemd 是唯一知道的。
- `configd` 故意不持有意见：
  - 不启动它们。
  - 不重启它们。
  - 报告是其全部参与。

### 2. 身份文件：运行的是哪个版本

- **不向 systemd 询问，也不从 `/proc` 推断**。
- 每个守护进程在启动时**发布自己的身份**到 `/run/<service>/identity.json`，这里读取它。
- 见 `duck_ipc_proto::Identity` 了解为什么这优于从外部检查进程——简而言之：
  - 进程知道自己的版本、git 修订和自己的 exe。
  - 不需要特权就能说。

### 为什么两者一起读

- 它们分开意味着不同的东西：
  - **已发布身份 + 已停止单元**：不可能发生——systemd 随单元删除运行时目录。
  - **已停止单元 + 无身份**：单元未运行。
  - **运行中守护进程太旧未发布身份**：也报告无身份。
  - 单元状态区分后两者。

## 四、常量

### `PADD = "padd.service"`

- 将手柄转换为意图的单元。
- 单独定义因为 `pad.status` 需要它。

### `MANAGED = [7 个单元]`

```rust
pub const MANAGED: [&str; 7] = [
    "updaterd.service",
    "robotd.service",
    "configd.service",
    "btd.service",
    "padd.service",
    "mediad.service",
    "tofd.service",
];
```

- 一个守护进程版本管理的每个单元，按读者想要的顺序：
  1. 更新引擎（updaterd）
  2. 机器人（robotd）
  3. 依赖两者的（configd, btd, padd, mediad, tofd）

#### 硬编码而非发现——真实限制

- 硬编码而非发现，这是一个值得命名的真实限制：
  - 添加到版本但未添加到此列表的单元在这里不可见。
- 替代方案——向 systemd 询问所有内容并过滤——会报告此项目不拥有的单元，对状态行更糟。
- `scripts/install.sh` 确切知道这些。

#### 已经付出过代价

- `mediad` 和 `tofd` 比它们被命名在这里早两个版本发布了单元。
- 因此一个人在更新后阅读的块——存在的目的是说明哪个守护进程仍在旧版本上——根本无法报告它们中的任何一个。

## 五、函数

### `state(unit: &str) -> proto::UnitState`

- systemd 对一个单元的说法。
- 窄问题，保留给 `pad.status`。

### `all() -> Vec<proto::ServiceUnit>`

- 返回所有 `MANAGED` 单元的状态和身份。
- 按 `MANAGED` 顺序。

### `describe(unit: &str) -> proto::ServiceUnit`

- 组合单元状态和身份。

#### Linux 版本

1. 查询 systemd 获取单元状态（失败则警告并设为 Unknown，不中断报告）。
2. 读取 `/run/<service>/identity.json` 获取身份。
3. 返回 `ServiceUnit { identity, unit, state }`。

#### 非 Linux 版本

- 没有 systemd 可询问，编造答案会让笔记本看起来像有损坏守护进程的机器人。
- 身份文件仍被读取：它是普通文件，手动在笔记本上运行的守护进程会发布一个。
- 状态设为 `Unknown`。

### `service_of(unit: &str) -> &str`

- `btd.service` 命名服务为 `btd`，这是它发布身份的名称。
- 去除 `.service` 后缀。

### `query(unit: &str) -> Result<proto::UnitState, String>`（Linux only）

#### 使用 `LoadUnit` 而非 `GetUnit`

- `GetUnit` 对 systemd 未加载的单元失败，这与单元不存在无法区分——而在这里是不同的答案。
- `LoadUnit` 如果文件存在就加载它，只在真正不存在时失败。

#### 单元不存在的处理

- `LoadUnit` 失败 → 记录 debug 日志，返回 `Absent`。
- 这是关于安装的事实（板在比添加该单元的版本更旧的版本上），不是报告失败。

#### ActiveState 映射

| systemd 状态 | 映射到 | 原因 |
|---|---|---|
| `active` | `Active` | 正常运行 |
| `activating` | `Active` | padd 在连接 robotd 的最初时刻处于此状态，报告为"未运行"会让启动中的机器人看起来损坏 |
| `reloading` | `Active` | 重载中仍在运行 |
| `inactive` | `Inactive` | 未运行 |
| `deactivating` | `Inactive` | 正在停止 |
| `failed` | `Inactive` | 失败是有原因的非活动，原因在日志中而非这里；合并它们保持这是状态行而非诊断 |
| 其他 | `Unknown` | 不熟悉的状态，警告并记录 |

### `property()`（Linux only）

- 通过 D-Bus `org.freedesktop.DBus.Properties.Get` 读取单元属性。
- 通用辅助函数，用于读取 `ActiveState` 等。

## 六、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `main.rs` | `system.services` 调用 `units::all()` |
| `pad.rs` | `pad.status` 调用 `units::state(PADD)` 判断 padd 是否运行 |
| `btd` | 通过 `system.services` 暴露给手机应用 |
| `updaterd` | 更新后用于检查哪些守护进程仍在旧版本 |
| `duck_ipc_proto::Identity` | 定义身份文件格式和 `read_identity()` |
| `systemd` | 实际管理单元的系统服务 |

## 七、设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **单一信息源** | 运行状态问 systemd，版本问守护进程自己发布的身份文件 |
| **不重复 systemd 的工作** | configd 不启动/重启单元，只报告 |
| **硬编码白名单** | `MANAGED` 列表确保只报告项目拥有的单元 |
| **优雅降级** | 查询失败不中断整个报告，单个单元设为 Unknown |
| **跨平台** | 非 Linux 回退为 Unknown，不编造答案 |
| **状态语义清晰** | `activating` 视为 Active，`failed` 视为 Inactive，避免误报 |
#（注：内容由AI生成）
