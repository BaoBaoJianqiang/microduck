# units.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/units.rs`
- **角色**：回答「机器人的哪些守护进程在运行、各自跑的是哪个版本」。`system.services`（全量）与 `pad.status` 中 `padd` 驱动状态（单点）的数据来源。

## 2. 核心设计决策（模块文档）

- 起点是「`padd` 在运行吗？」：手柄灯亮着、机器人却毫无反应、两处都没有任何线索，正是「连着的手柄 + 死掉的 padd」这种看起来像硬件正常的故障。该论据从不限于 `padd`——死掉的 `btd` 是一台手机看不见的机器人，同样沉默——所以它对一个发布所管理的每个单元作答。
- **向 systemd 询问，而不是自行跟踪**：这些单元由 systemd 启动、停止、重启，只有 systemd 知道真相。`configd` 刻意不持立场：不启动、不重启它们，报告就是它参与的全部。
- **「跑的是哪个版本」不问 systemd、也不从 `/proc` 推断**：每个守护进程启动时把自己的身份发布到 `/run/<service>/identity.json`，本模块读取它（见 `duck_ipc_proto::Identity`）。进程最清楚自己的版本、git revision 与 exe，且无需特权即可说出。
- 两个问题一起读，因为分开看含义不同：systemd 停单元时会删掉运行时目录，所以「有身份但单元已停」不可能发生；但「单元停了、没有身份」与「守护进程太老、不会发布身份」都表现为空，**单元状态正是区分二者的依据**。

## 3. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `PADD` | `"padd.service"` | 把手柄变成 intent 的单元；`main.rs` 的 `PadStatus` 用它附带驱动状态 |
| `MANAGED` | 长度 7 的数组，见下 | 一个守护进程发布所管理的全部单元，按读者想要的顺序排列 |

`MANAGED` 的硬编码顺序：

1. `updaterd.service`（更新引擎）
2. `robotd.service`（机器人本体）
3. `configd.service`
4. `btd.service`
5. `padd.service`
6. `mediad.service`
7. `tofd.service`

排序理由：先更新引擎、再机器人、然后是依赖前两者的单元。**硬编码而非动态发现**是一个被明确点名的真实限制：加进发布却忘了加进此表的单元在此不可见；另一种做法（列出 systemd 全部单元再过滤）会报进本项目不拥有的单元，对状态行更糟。`scripts/install.sh` 知晓的恰好是这七个。这已经付出过一次代价：`mediad`、`tofd` 的单元在被列入此表两个发布之前就已上船，导致更新后那段「谁还停留在旧版本」的状态块当时完全报不出它们。

## 4. 函数逐项解析

| 函数 | 平台 | 职责 |
| --- | --- | --- |
| `pub async fn state(unit) -> proto::UnitState` | 全部 | 窄查询：只返回某单元的状态（`describe(unit).state`），供 `pad.status` 使用 |
| `pub async fn all() -> Vec<proto::ServiceUnit>` | 全部 | 按 `MANAGED` 顺序逐个 `describe`，带容量预分配 |
| `pub async fn describe(unit) -> proto::ServiceUnit` | Linux / 非 Linux 两个 `cfg` 版本 | 汇总一个单元的状态与身份 |
| `fn service_of(unit) -> &str` | 全部 | 去掉 `.service` 后缀：`btd.service` 发布身份时用的服务名是 `btd` |
| `async fn query(unit) -> Result<UnitState, String>` | Linux | 实际向 systemd 查询并映射状态 |
| `async fn property(bus, path, interface, name)` | Linux | 对指定单元对象调用 `org.freedesktop.DBus.Properties.Get`，返回 owned `Value` |

### `describe` 的两个版本

- **Linux**：先 `query` 取状态；查询失败只记 `warn` 并置 `UnitState::Unknown`——这只是状态报告中的一行，读不到一行不能毁掉整份报告。随后 `proto::ServiceUnit { identity: proto::read_identity(service_of(unit)), unit, state }`。
- **非 Linux**：板下没有 systemd 可问，编造状态会让笔记本看起来像一台守护进程损坏的机器人；状态固定 `Unknown`，但**身份文件仍照常读取**（它是普通文件，笔记本上手跑的守护进程也会发布）。

### `query`：systemd 状态映射

- 连系统总线后调用 systemd `Manager.LoadUnit`（而非 `GetUnit`）：`GetUnit` 对未加载单元报错，与「单元不存在」无法区分，而这两者在此是不同答案；`LoadUnit` 在文件存在时会加载它，仅当真的不存在时才失败——此时返回 `UnitState::Absent`（以 `debug` 记录，代表该板的发布比加入此单元的版本旧，是关于安装的事实而非故障）。
- 读取 `org.freedesktop.systemd1.Unit` 的 `ActiveState` 字符串属性并映射：

| systemd ActiveState | 映射 | 理由 |
| --- | --- | --- |
| `active` / `activating` / `reloading` | `Active` | `activating` 算活动：`padd` 启动头几刻正在连 `robotd`，报成「没在跑」会让启动中的机器人显得坏了 |
| `inactive` / `deactivating` / `failed` | `Inactive` | `failed` 是带原因的 inactive，原因在日志里；合并以保持这是状态行而非诊断 |
| 其他陌生值 | `Unknown` | 记 `warn`（含具体 state 与 unit） |

## 5. 要点小结

- 运行状态只信 systemd，版本身份只信各进程自发布的 `/run/<svc>/identity.json`，二者并列以区分「停了」与「太老」。
- 受管单元七强名单硬编码、固定顺序，漏登记即不可见（已有历史教训）。
- 用 `LoadUnit` 区分「未加载」与「不存在」；单行查询失败降级为 `Unknown` 不拖垮整份报告。
- 非 Linux 下状态一律 `Unknown`，但身份文件仍读取。
