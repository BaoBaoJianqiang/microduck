# store.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/store.rs`
- **角色**：配置存储。核心命题是「**配置是一个文件，而不是一个服务**」（`architecture.md` §3.1）。持有身份与偏好（机器人名字、配对 PIN），**不持有任何凭据**——凭据归 NetworkManager。

## 2. 核心设计决策（模块文档）

- **刻意不做成独立守护进程**：普通文件 + `flock` 串行化写 + 写临时文件再 `rename(2)` 保证原子性，没有单点故障，任何服务挂掉时文件仍可读，更新器永不碰它——因此能同时扛过更新与回滚（`updater-design.md` §5.7）。
- **暂不引入 inotify**：只有出现第二个进程读此文件时才需要；今天 `configd` 是唯一读写者，监视一个自己独占写的文件纯属仪式。

## 3. 常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `MAX_NAME` | `24` | 可接受的最长名字。约束来自 BLE 而非品味：传统广播有效载荷总共 31 字节，本地名要与 flags 和 16 字节服务 UUID 共享；超长名字会被适配器静默截断或挤进手机可能不请求的 scan response。在这里显式截断并回传实际存值，是诚实的做法 |
| `DEFAULT_PIN` | `"000000"` | 出厂配对 PIN。刻意使用众所周知的默认值而非每板随机值：它证明的是物理在场，仅此而已；存在意义是让整套流程（agent、存储、六位契约）就位并保证开箱可配。每机 PIN 印在机器人底部是预置环节的事；结果中的 `is_default` 让所有客户端能明说它不是秘密 |
| `PIN_DIGITS` | `6` | PIN 位数。**六而非五**：蓝牙 passkey entry 定义为六位值 000000–999999（核心规范 LE Secure Connections），BlueZ 原样交给 agent；五位就得在某处补零，双方会为「12345」还是「012345」争执——那将是无人能诊断的支持工单 |

## 4. 类型与方法逐项解析

### `struct Config`（`Serialize`/`Deserialize`/`Default`/`Clone`/`Debug`）

磁盘上的 JSON 结构：

- `name: Option<String>`，`#[serde(skip_serializing_if = "Option::is_none")]`
- `pairing_pin: Option<String>`，同样在为 `None` 时跳过序列化

即未设置的字段不会写入文件。

### `pub struct Store`

- 字段 `path: PathBuf`：配置文件路径（`main.rs` 传入 `<state-dir>/config.json`）。
- 字段 `fallback: String`：文件还没有名字时使用的默认名。由调用方提供而非本类型计算——它派生自硬件（`identity` 把 SoC 序列号变成 `duck-7f3a`），而 Store 只是一个文件，不该知道硬件。
- `pub fn new(path, fallback) -> Self`：构造器。

### 读方法

- `pub fn name() -> String`：返回机器人名字；文件缺失或不可解析时**回退到 fallback 并记 `warn`**，而不是报错。理由：未预置的板子必须带着可用名字启动；因可选配置损坏而拒绝启动的守护进程，远程排查困难得多（与 `robotd.toml` 可选同理）。
- `pub fn pairing_pin() -> String`：返回 PIN，缺省为 `DEFAULT_PIN`。
- `pub fn name_and_pin_result() -> duck_ipc_proto::PairingPinResult`：一次返回 PIN 与 `is_default`（与常量就地比较），让「是否默认」的判断与它比较的常量放在一起，而不是散落在每个调用方。
- `fn read() -> std::io::Result<Config>`：读文件并反序列化；文件不存在（`NotFound`）返回 `Config::default()`，其余错误透传，JSON 解析错误包装为 `InvalidData`。

### 写方法

- `pub fn set_name(requested: &str) -> io::Result<String>`：经 `sanitise` 清洗；若结果为空，返回 `InvalidInput`（「名字至少要有一个可打印字符」）。否则读入现有配置（损坏则用默认）、写入、**返回实际存储值**——调用方必须展示返回值而非它发送的值，因为 trim/截断会使二者不同。
- `pub fn set_pairing_pin(requested: &str) -> io::Result<String>`：先 `trim`；必须**恰好 6 个 ASCII 数字**（长度等于 `PIN_DIGITS` 且 `is_ascii_digit`），否则 `InvalidInput`（错误信息标明范围 000000–999999）。PIN 以字符串保存，**前导零保留**。

### `fn write(&self, config: &Config) -> io::Result<()>`

崩溃安全写盘序列：

1. `create_dir_all` 确保父目录存在。
2. 在 `config.lock`（`with_extension("lock")`）上 `flock` 串行化写者；锁持在锁文件而非配置本体上，使锁能活过替换配置的那次 rename。
3. 序列化为 pretty JSON，末尾补一个换行。
4. 写 `config.tmp` → `sync_all`（fsync 文件）→ `rename` 到目标 → **对目录再 `fsync`**（`File::open(dir)?.sync_all()`）。目录 fsync 是常被遗忘的一步：没有它 rename 可能扛不住断电，而机器人被人直接拔墙电是常态而非异常。与更新日志同一纪律（§8.2）。

### `fn sanitise(requested: &str) -> String`

- `trim()` 去首尾空白；
- 用 `chars().filter(|c| !c.is_control())` **剔除控制字符**（换行/NUL 会割裂日志记录、破坏广播）；
- 用 `chars().take(MAX_NAME)` **按字符边界**截断到 24 个字符（按字节切会在 UTF-8 多字节字符中途 panic）；
- 最后再 `trim_end`。

## 5. 单元测试说明

测试辅助 `fn store(dir) -> Store` 以 `radxa-zero3` 为 fallback 在临时目录构造 Store。

| 测试 | 验证内容 |
| --- | --- |
| `a_missing_file_yields_the_fallback` | 无文件时名字取 fallback |
| `a_name_survives_a_write_and_a_reread` | 写入后重读一致；第二个 Store 实例读同一文件也一致——这是文件而非进程状态 |
| `a_corrupt_file_yields_the_fallback_rather_than_an_error` | `{not json` 损坏文件不阻止启动，回退默认名 |
| `control_characters_are_stripped` | `"Duck\ny\u{0}"` 存为 `"Ducky"` |
| `a_long_multibyte_name_is_truncated_without_panicking` | 80 个 `é` 按字符截断到 `MAX_NAME`，不 panic |
| `whitespace_is_trimmed_and_reported` | `"  Ducky  "` 实际存 `"Ducky"` 并如实返回 |
| `an_empty_name_is_refused` | 纯空白名报 `InvalidInput`，且什么都不写，旧名保留 |
| `a_missing_pin_is_the_factory_default` | 无 PIN 时为 `000000` |
| `a_pin_keeps_its_leading_zeros` | `"012345"` 以字符串存取，不变成 `"12345"` |
| `a_pin_must_be_exactly_six_digits` | `12345`、`1234567`、空串、`12 456`、`abcdef`、含阿拉伯-印度数字 `٦` 的 `12345٦` 全部拒绝，且默认 PIN 不受影响 |
| `a_name_and_a_pin_coexist` | 名字与 PIN 共享一个文件，写一个不得丢失另一个（改名后 PIN 仍在） |
| `writing_leaves_no_temp_file` | 写后不残留 `config.tmp`，以免被后续读取误认 |

## 6. 要点小结

- 普通 JSON 文件即存储，`flock` 串行化、临时文件 + rename + 双 fsync 抗断电。
- 名字受 BLE 31 字节广播预算约束（上限 24），按字符截断、剔除控制字符、空名拒绝。
- PIN 恰好 6 位 ASCII 数字、字符串保存保留前导零，默认 `000000` 并通过 `is_default` 明示其公开性。
- 文件缺失或损坏一律回退而非阻止服务；不存任何凭据。
