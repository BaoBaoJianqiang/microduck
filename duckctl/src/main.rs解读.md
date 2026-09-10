# `main.rs`（duckctl）解读

## 概述

`duckctl` 是从笔记本电脑通过 BLE 与机器人通信的命令行工具，是手机 App 的替代者，也是唯一能在真实无线电上测试 `btd` 的方式。

**核心设计哲学**：
- **蓝牙是到达机器人的方式，不是工具的本质**。`mediad` 给机器人提供了第二种传输（WebRTC），不同传输允许不同方法（`robot.move` 在 BLE 上被拒绝、在 WebRTC 上允许；`net.connect` 反之），所以工具名标识哪个机器人而非哪种无线电。
- **机器人上没有任何东西依赖这个 crate**，这使得 `btleplug` 不会进入发布版。它曾经是 `btd` 的 example，现在是独立 crate。
- **使用 `btleplug` 而非 `bluer`**：因为这运行在开发者机器上——macOS 上是 CoreBluetooth、Linux 上是 BlueZ、Windows 上是 WinRT。`bluer` 会把客户端限制在 Linux。
- **故意复用 `btd::framing`**：这里的分块是机器人所用同一模块的*客户端*一半，如果分块不对称这就不会工作——这使它成为协议的真实测试而非可以自洽的重新实现。

### 用法示例

```text
cargo run -p duckctl -- scan          # 范围内的机器人及其地址
cargo run -p duckctl -- status
cargo run -p duckctl -- wifi scan
cargo run -p duckctl -- wifi connect "Pollen" --psk secret
cargo run -p duckctl -- name "Ducky"
cargo run -p duckctl -- call robot.health
```

环境变量 `DUCK_ROBOT` 和 `DUCK_PIN` 是 `--name` 和 `--pin` 的默认值。

---

## 关键常量

### 扫描相关

```rust
const SCAN_TIME: Duration = Duration::from_secs(8);      // 扫描最长 8 秒
const SCAN_POLL: Duration = Duration::from_millis(250);   // 每 250ms 重新读取扫描结果
```

- **8 秒**：慷慨，因为 BLE 发现确实慢，机器人以 BlueZ 选择的任何间隔广告。更短会让只是运气不好的笔记本报告"没有机器人"。
- **250ms 轮询**：过去是固定睡眠后单次快照，间歇性失败——BLE 广告是周期性的，CoreBluetooth 对已配对外围设备的视图时有时无，所以机器人是否在那一次快照中部分靠运气。轮询直到出现东西也使常见情况在远不到一秒内完成。

### 回复超时（空闲而非总计）

```rust
const REPLY_TIMEOUT: Duration = Duration::from_secs(15);       // 普通调用 15s
const SLOW_REPLY_TIMEOUT: Duration = Duration::from_secs(60);  // 慢调用 60s
const UPDATE_IDLE_TIMEOUT: Duration =
    Duration::from_secs(duck_ipc_proto::UPDATE_MAX_SILENCE_SECONDS + 60);  // 更新空闲超时
const FOLLOW_TIMEOUT: Duration = Duration::from_secs(24 * 3600);  // watch 跟随 24 小时
```

**关键设计：空闲超时而非总计超时**。
- 一个 apply 需要多久就多久（下载、验证、解压、交换、钩子、健康门），所以总计预算要么切断正在工作的更新，要么等待死机器人。
- 但有用的信号已经在到达：每个进度通知都是机器人活着且在工作的证明，所以每个都重新启动时钟。停滞的镜像仍然在几秒内失败。
- `UPDATE_IDLE_TIMEOUT` 比任何其他调用都长：钩子的阶段通知在钩子*之前*到达而非期间，所以预算是更新可以合法拥有的最长间隙，而非更新可以花的最长时间。这个间隙是安装前钩子的上限（安装 ONNX Runtime、约 100MB apt 的 GStreamer 栈），从 `duck_ipc_proto::UPDATE_MAX_SILENCE_SECONDS` 派生而非在此硬编码。加 60 秒余量给钩子之后的回复。

### 连接阶段超时

```rust
const CONNECT_TIMEOUT: Duration = Duration::from_secs(20);   // 连接 20s
const DISCOVER_TIMEOUT: Duration = Duration::from_secs(20);  // 服务发现 20s
const READ_TIMEOUT: Duration = Duration::from_secs(15);       // 首次读取 15s
```

每个步骤都有自己的预算和自己的消息。btleplug 对这些都不设限，没有它们的话，在"找到机器人"和"发送请求"之间任何地方卡住都会打印 `connecting to …` 然后什么都没有——只说了有问题，没说是什么。

### 其他

```rust
const LINK_POLL: Duration = Duration::from_secs(2);  // 每 2s 检查链接是否还在
const LISTED_DEVICES: usize = 12;                      // 扫描结果最多列 12 个设备
const DEFAULT_PIN: &str = "000000";                    // 出厂默认 PIN
```

- `LINK_POLL`：没有它的话，断开连接与机器人安静无法区分——通知流只是停止产出，所以等待运行到空闲预算然后报告机器人"停止回答"。在 `update apply` 之后这错了两次——机器人回答了，是链接断了——而且要等 `UPDATE_IDLE_TIMEOUT` 才说。2 秒远低于这里每个预算，每次只花一次便宜的 CoreBluetooth 查询。

---

## 核心数据结构

### `Seen` — 扫描到的设备

```rust
struct Seen {
    peripheral: Peripheral,
    identity: String,        // 标识（macOS 上用 id，Linux 上用地址）
    local_name: Option<String>,  // 广告中的本地名称
    services: usize,         // 广告的服务数量
    duck: bool,              // 是否携带 duck 服务 UUID
    address: Address,        // 广播的 IPv4 地址
}
```

名称按到达时保留——`None` 当广告没带名称时——而非作为地址回退，因为"报告时没有名称"本身就是诊断，回退会隐藏它。

### `Address` — 三态而非两态

```rust
enum Address {
    At(Ipv4Addr),     // 有地址
    Unassigned,        // 字段在但说是 0.0.0.0——机器人没有网络
    Unsaid,            // 根本没有字段——旧版 btd 或非机器人设备
}
```

**为什么不是 `Option<Ipv4Addr>`**：两个空白会被合并成一个，但它们把读者送到不同地方：
- 广播 `0.0.0.0` 的机器人没有网络——wifi 问题
- 什么都没广播的机器人是在此功能存在之前的发布版——更新问题

`Unsaid` 在列表中渲染为空白而非"unknown"：每个非机器人行都是 `Unsaid`，一屋子耳机旁边一列"unknown"是噪音。

### `Target` — 要连接哪个机器人

```rust
struct Target {
    name: Option<String>,  // 要找的名称，空不是名称
    from_env: bool,        // 是否来自环境变量而非 --name
}
```

**为什么不用 clap 自己的 `env` 支持**：clap 用 `env::var_os` 读取变量并把 `DUCK_ROBOT=` 当作值，所以 shell profile 中导出的变量只能通过 unset 来转义——而需要转义的命令正是现在正在输入的命令，在一个有别人机器人的工作台上。空意味着未设置，所以 `DUCK_ROBOT= duckctl scan` 是转义舱口，用 shell 已有的形状。

**来源被携带而非重新计算**：一个默认使工具*更严格*——它抑制已连接回退层，把"第一个找到的机器人赢"变成"范围内没有叫 duck-c51b 的机器人"——这是编辑 shell profile 六周后令人困惑的失败，特别是当同一消息列出一个就在眼前的机器人时。所以每条关于没人命名的机器人的消息都说名称来自哪里。

### `Waited` — 等待通知的三种结局

```rust
enum Waited {
    Chunk(ValueNotification),  // 字节到达
    Dropped,                    // 链接断了
    Silent,                     // 预算到期但链接还在
}
```

三种结局而非两种，因为"什么都没到"隐藏了可诊断的那个：连接着但不说话的机器人与这台 Mac 不再连接的机器人需要不同的下一步。

---

## 扫描与发现逻辑

### 三层候选（按证据强度排序）

```rust
let mut advertised: Vec<(Peripheral, String)> = Vec::new();  // 广告了服务 UUID
let mut named: Vec<(Peripheral, String)> = Vec::new();        // 名称匹配但没广告服务
let mut connected: Vec<(Peripheral, String)> = Vec::new();    // 已连接（仅无名称时）
```

- **advertised**：广告了 duck 服务 UUID——最强证据，但已配对机器人经常停止向这台 Mac 广告服务。
- **named**：名称匹配但没广告服务——已配对机器人的常见情况，名称是唯一证据。
- **connected**：已连接的外围设备——最后手段，仅在没有名称时。未过滤扫描看到 Mac 上每个已连接外围设备，所以这一层充满键盘和耳机。每个都要花一次连接和服务发现才能排除，这就是为什么显式名称完全抑制这一层而非合并进去。

### 无过滤扫描（关键踩坑点）

```rust
adapter.start_scan(ScanFilter::default()).await?;
```

**过去传 `ScanFilter { services: [SERVICE_UUID] }`**，理论上忙办公室不会用耳机淹没机器人。但 CoreBluetooth *严格*遵守该过滤：当前广告不带 UUID 的外围设备根本不会被报告。已配对机器人经常以空服务列表报告——所以 `--name` 回退只能匹配过滤扫描已经返回的东西，这在它存在的 exactly 情况下成了死重。这就是"一次运行 no robot found，下一次成功"的完整解释。

**所以：报告一切，在这里区分，规则是我们的。**

### `worth_connecting` — 何时停止扫描

```rust
fn worth_connecting(advertised, named, connected, target) -> bool
```

- **没有名称时**：第一个候选就赢，停在那里是重点——已配对机器人可能永远不再向这台 Mac 重新广告服务，所以等截止日期找更好的只是八秒什么都没有。
- **有名称时**："一个候选"不等于"那个候选"。层在名称应用之前构建——`advertised` 持有每个带服务 UUID 的机器人，不管它是谁——所以停在第一个非空层就是停在无线电碰巧先报告的那个机器人。工作台上有两个的话这是抛硬币，输的那个根本不会被扫描：失败然后读 `no robot named "olducky" in range` 并列出另一个机器人作为证据，这是关于八秒的声明却在两百毫秒后做出。`scan`（跑满截止日期）报告两个——两个命令对范围内有什么不一致就是症状。
- **所以有名称意味着：继续听直到有东西回答它。** 代价是名称指定的命令其机器人不在范围内时要付完整 `SCAN_TIME` 才失败，这是正确的权衡——那个失败的全部内容就是无线电找了八秒什么都没找到。

### `choose` — 选择机器人

```rust
fn choose<T>(found: Vec<(T, String)>, target: &Target) -> Result<(T, String), String>
```

**一个名称匹配多个候选时拒绝，不解析。** 两个从这里无法区分，所以没有什么可偏好的，选任何一个意味着写落在扫描碰巧先报告的那个上——`net.connect` 把 wifi 密码放在那个机器人上。`identity.rs` 命名了这在没人做错任何事时如何发生：bootloader 留下空 `serial-number` 的板子回退到主机名，所以从一个镜像刷出的每个板子都回答 `radxa-zero3`。

**没有名称时第一个候选仍然赢。** 那条路径故意不变——在没人命名的机器人之间选择正是省略 `--name` 所要求的，把它变成错误会破坏任何有两个板子的工作台上的简写。

### `answers_to` — macOS 复合名称匹配

```rust
fn answers_to(reported: &str, wanted: &str) -> bool
```

**一个外围设备可以同时以两个名称到达。** CoreBluetooth 暴露*缓存的 GAP 名称*——`CBPeripheral.name`，通过在更早连接上读 `0x2A00` 学到——与广告中的本地名称分开，btleplug 在它们不同时把它们连在一起报告：`radxa-zero3 [duck-c51b]`。

它们过去在每个机器人上都不同，因为两个名称来自不同地方：GAP 名称是 BlueZ 的适配器别名，从主机名派生所以每个从一个镜像刷出的板子都是 `radxa-zero3`，而广告携带 `configd` 拥有的名称。`btd` 现在把别名设为广告名称，所以当前发布版的机器人不管怎么问都报告一个名称。

**但这仍然必须接受两者**，只要工作台上有旧版机器人。旧版发布版的板子有旧别名；在机器人更新前缓存了旧 GAP 名称的客户端也有，直到 `bluetoothctl remove <mac>` 或在 macOS 蓝牙设置中忘记它清除。精确匹配连接字符串意味着**两种**人会输入的拼写都被拒绝——然后失败列出机器人作为它不在范围内的证据。

**所以任一半都被接受。** 广告的一半是机器人的真实名称，手机 App 必须匹配的那个；GAP 一半被接受因为它是 macOS 蓝牙设置显示的。

用 `rsplit_once`，所以本身包含方括号的 GAP 名称把*最后*一组当作广告的一半——那是 btleplug 追加的那个。

---

## 连接与认证流程

### 连接三步骤（每步独立超时）

```rust
step("connecting", hint, CONNECT_TIMEOUT, peripheral.connect()).await?;
step("service discovery", hint, DISCOVER_TIMEOUT, peripheral.discover_services()).await?;
let (request, response) = characteristics(&peripheral)?;
```

`step` 包装一个有预算的步骤，预算用完时命名它。每个失败有不同的提示：
- 连接失败：如果 macOS 显示已配对，在那里忘记它并重试；`sudo pkill bluetoothd` 也清除半成品配对。
- 服务发现失败：检查机器人上 `journalctl -u btd -b` 看 GATT 应用是否注册。

### 先读（触发配对的关键）

```rust
let read = step("reading the API version", hint, READ_TIMEOUT, peripheral.read(&response)).await;
```

**先读是承重的而非礼节。** 机器人要求经过认证的加密链接才能*写*，但订阅不需要加密——所以没有这个的话，中心愉快地订阅，第一次写被拒绝，在 macOS 上既看不到提示也看不到错误。读是被确认的，所以未配对链接在这里失败，这就是让 CoreBluetooth 开始配对的东西。

读到的值是机器人的 API 版本，被报告而非强制执行。见下方 `warn_about_skew`。

### 订阅先于写入

```rust
peripheral.subscribe(&response).await?;
let mut notifications = peripheral.notifications().await?;
```

btd 的会话在第一次写时开始，所以这里的顺序不只是防御性的：通知一半必须存在，会话才有地方回答。

### PIN 认证（just-works 加密 + 应用层 PIN）

```rust
let auth = serde_json::json!({
    "jsonrpc": "2.0",
    "id": 0,
    "method": "system.authenticate",
    "params": { "pin": pin },
});
write_line(&peripheral, &request, &auth).await?;
let reply = read_line(&mut notifications, REPLY_TIMEOUT).await?;
if parsed["result"]["authenticated"] != serde_json::json!(true) {
    // 报告剩余尝试次数
}
```

**证明 PIN 先于任何其他东西。** 配对是 just-works，所以它加密链接但不认证任何人；机器人在这成功之前什么都不服务。见 `btd/src/pairing.rs` 了解为什么检查在这里而非配对中。

PIN 即使在 verbose 模式下也不打印——终端也是日志。

### `resolve_pin` — PIN 解析

```rust
fn resolve_pin(flag: Option<String>, var: Option<String>) -> String
```

优先级：`--pin` → `DUCK_PIN` → 出厂默认 `000000`。

空意味着这里也未设置，原因同 `Target`：脚本遗留的 `DUCK_PIN=` 否则会用空字符串认证并被报告为错误 PIN。

与 `--name` 不同，空值被*跳过*而非最终。没有"无 PIN"状态要表达——每个请求都带一个——所以 `--pin ''` 只能意味着"不是这个"。

---

## 请求与响应

### 分块写入（20 字节地板）

```rust
for chunk in framing::chunks(&line, 20) {
    peripheral.write(&request, &chunk, WriteType::WithoutResponse).await?;
}
```

由机器人使用的同一代码分块。btleplug 不暴露协商的 MTU，所以 20 字节——每个 BLE 链接保证的地板——是安全假设：在好链接上比必要慢，在每个链接上都正确。

**注意：认证用 `WithResponse`，普通请求用 `WithoutResponse`。** 见 `write_line`。

### `write_line` — 确认写入

```rust
async fn write_line(peripheral, characteristic, line) -> Result<()> {
    for chunk in framing::chunks(line, 20) {
        peripheral.write(characteristic, &chunk, WriteType::WithResponse).await?;
    }
}
```

**确认写入，这不是细节。** ATT Write *Command*（`WithoutResponse`）不带回复，所以拒绝——比如加密不足——是不可见的：请求静默地永远不到达，客户端等完超时不知道为什么。这正是这个第一次针对工作完全正常的机器人的表现。

### 空闲超时循环

```rust
let mut deadline = tokio::time::Instant::now() + timeout;
loop {
    let notification = match next_chunk(&peripheral, &mut notifications, deadline).await {
        Waited::Chunk(notification) => notification,
        Waited::Dropped => return Err(dropped(&cli.command).into()),
        Waited::Silent => return Err(silence(timeout).into()),
    };
    deadline = tokio::time::Instant::now() + timeout;  // 每个通知重新启动时钟
    // 重组、解析、判断是否是答案
}
```

**截止日期是空闲的，不是总计的**：每个到达的通知把它推回去，因为发进度的机器人是在工作的机器人。

### `next_chunk` — 带链接检查的等待

```rust
async fn next_chunk(peripheral, notifications, deadline) -> Waited
```

循环：计算剩余时间，如果为零返回 `Silent`；用 `remaining.min(LINK_POLL)` 超时等待下一个通知。
- 到达 → `Chunk`
- 流结束（btleplug 放弃了外围设备）→ `Dropped`
- 超时 → 检查 `peripheral.is_connected()`，只有明确的"否"才结束等待（适配器在问题上出错没有说链接断了，把那当作丢弃会过早结束工作调用——而相信已经断了的链接最多花这个已经容忍的静默时间）

**为什么需要轮询**：仅流不能报告第二个：在 macOS 上，调用中途断开的外围设备让 `notifications()` 挂起而非结束，所以知道链接断了的唯一方式是问。

### 进度通知 vs 答案

```rust
let is_answer = value.get("id").is_some_and(|id| !id.is_null());
if !is_answer {
    if value["method"] == "update.progress" {
        eprintln!("· {}", progress_line(&value["params"]));
    } else {
        eprintln!("· {}", serde_json::to_string(&value)?);
    }
    continue;
}
```

没有 `id` 的通知是进度流，不是答案；报告它们并继续等待关闭调用的响应。

`update.progress` 渲染为一行（组件: 阶段 百分比 — 细节），其他未知通知打印完整 JSON——这个工具不知道的通知值得完整看。

### `progress_line` — 进度渲染

```rust
fn progress_line(params: &serde_json::Value) -> String
```

进度像所有非答案的东西一样去 stderr，所以 `duckctl … > reply.json` 把两者分开。过去打印为漂亮 JSON，每下载百分之一就放十几行标点在 stdout 上。

阶段总在；百分比只在下载期间；细节很少。`None` 百分比不能打印为 `null%`。

---

## 命令分发

### `request_line` — 命令→JSON-RPC

```rust
fn request_line(command: &Command) -> Result<(String, Duration), Box<dyn std::error::Error>>
```

每个命令映射到一个 JSON-RPC 方法、参数和超时：

| 命令 | 方法 | 超时 |
|------|------|------|
| `Status` | `update.status` | 15s |
| `Ip` / `Open` | `net.status` | 15s |
| `Version` | `hello` | 15s |
| `Info` | `system.info` | 15s |
| `Health` | `robot.health` | 15s |
| `Name` | `system.setName` | 15s |
| `Reboot` | `system.reboot` | 15s |
| `Wifi::Status` | `net.status` | 15s |
| `Wifi::Scan` | `net.scan` | 60s |
| `Wifi::Connect` | `net.connect` | 60s |
| `Wifi::Forget` | `net.forget` | 15s |
| `Update::*` | 见下 | 各不同 |
| `Call` | 用户指定 | 60s |

- `Wifi::Scan`：让 NetworkManager 重新扫描，在安静的无线电上要几秒。
- `Wifi::Connect`：configd 轮询 NM 最多 45s 才称加入超时，所以这必须等比那更久，否则工具在机器人决定之前就放弃。
- `Call`：任意方法，参数为 JSON，默认 `{}`。

### `Ip` 和 `Open` 的广告捷径

```rust
if resolving {
    match choose(std::mem::take(&mut addresses), &target) {
        Ok((Address::At(address), _)) => return deliver(&cli.command, &address.to_string()),
        Ok((Address::Unassigned, name)) => return Err(no_address(&name).into()),
        Ok((Address::Unsaid, name)) => eprintln!(...),  // 回退到连接
        Err(_) => { ... }  // 回退到连接
    }
}
```

`ip` 和 `open` 想要广告中的一个字段，所以它们像 `scan` 一样读它——与 `scan` 不同的是，当没有广告携带地址时它们终究会连接。便宜读先，调用后：没有回退的话这两个命令会在最常使用它们的笔记本上失败，因为已配对这台 Mac 的机器人经常停止向它广告服务。

- `Address::At` → 直接交付地址，不连接
- `Address::Unassigned`（广播了 0.0.0.0）→ 机器人没有网络，告诉怎么用 wifi connect
- `Address::Unsaid`（旧版没广播地址）→ 回退到连接问 `net.status`
- 没广告服务 → 回退

### `deliver` — 地址交付

```rust
fn deliver(command: &Command, address: &str) -> Result<()>
```

- **`ip` 只打印地址，什么都不**，因为工具的分工是诊断在 stderr、数据在 stdout：`ssh radxa@$(duckctl ip)` 只有在那是 stdout 的全部时才工作。这个命令发出的每个注释都去 stderr 出于同样原因。
- `open`：构建 `http://<address>:<port>/`，`--print` 只打印 URL，否则用 `webbrowser::open` 打开浏览器。

### `console_url` — 纯 HTTP

```rust
fn console_url(address: &str, port: u16) -> String {
    format!("http://{address}:{port}/")
}
```

纯 `http`，因为 LAN 上的机器人没有证书可提供，而从 `https` 页面来的 `ws://` 被直接阻止为混合内容。`webrtc-console.md` §1.3 说了这代价是什么——麦克风，在某些浏览器上还有游戏手柄，两者都是安全上下文 API——以及何时改变。

### `no_address` — 无网络诊断

```rust
fn no_address(name: &str) -> String
```

两种方式到达这里且是同一情况：广告携带了 `0.0.0.0`，和 `net.status` 没有 `ip4`。修复在两种情况下都通过无线电，而且必须——`net.connect` 被设计为在 WebRTC 上拒绝，因为从没见过网络的机器人不能通过那个网络被配置。

---

## 更新命令

### `update_request_line` — 用 proto 类型构建

```rust
fn update_request_line(update: &Update) -> Result<(String, Duration), Box<dyn std::error::Error>>
```

从 `duck_ipc_proto` 自己的类型构建而非手写 JSON。`update.apply` 的目标是外部标签枚举——`"latest"`、`{"exact":"0.5.1"}`、`{"ref":"my-branch"}`——手写那个形状出错是机器人的 `PARSE_ERROR`，里面没有线索。序列化守护进程反序列化的类型不可能错。

### `Update::Apply` 目标组合

```rust
let target = match (version.clone(), git_ref, staging) {
    (Some(version), _, true) => proto::Target::StagingExact(version),
    (Some(version), _, false) => proto::Target::Exact(version),
    (None, Some(git_ref), _) => proto::Target::Ref(git_ref.clone()),
    (None, None, true) => proto::Target::Staging,
    (None, None, false) => proto::Target::Latest,
};
```

五种目标形态，由 `--version`、`--ref`、`--staging` 三个标志组合决定。`--version` 和 `--ref` 互斥（clap `conflicts_with`）。

dev build（`--ref`）只有团队密钥在受信集且 `allow_dev_keys` 开时机器人才接受——客户机器人拒绝。

### 更新超时

| 命令 | 超时 |
|------|------|
| `check` | 60s（到达网络，更新中立即回 BUSY） |
| `apply` | `UPDATE_IDLE_TIMEOUT`（空闲，非总计） |
| `status` | 15s |
| `versions` | 15s |
| `log` | 15s |
| `rollback` | `UPDATE_IDLE_TIMEOUT` |
| `select` | `UPDATE_IDLE_TIMEOUT` |
| `watch` | 24 小时（跟随进度直到中断，Ctrl-C 结束） |

### `restart_note` — 更新后连接断开预告

```rust
fn restart_note(command: &Command, reply: &serde_json::Value) -> Option<&'static str>
```

不是失败，也不应该读起来像失败：`updaterd` 和 `btd` 是更新在飞行中从不重启的两个单元——`btd` 可能是它到达的传输——所以两者都在回复发出后约五秒重启（`docs/design/restart-order.md` §1）。每个客户端都必须预期这一点，手机 App 应该把它显示为一个步骤而非错误。

只在：
- 命令是 `apply`/`rollback`/`select` 且组件是 `daemon`（`btd` 只在 daemon 发布版中）
- 结果是 `applied` 或 `rolled_back`（`already_current` 和 `dry_run_passed` 什么都不重启）

### `dropped` — 链接断开诊断

```rust
fn dropped(command: &Command) -> String
```

与 `silence` 不同的诊断，区别在更新时最大：`btd` 在 apply 回答后约五秒重启，所以断开是*成功*更新的形状之一。称那为"机器人停止回答"把工作完美的机器人描述成死的。

如果是 daemon 组件的 `apply`/`rollback`/`select`：说更新重启机器人的守护进程，这和失败一样可能是更新完成。重新连接并运行 `duckctl update status`：`last_attempt` 携带运行结果。
否则：重新连接并重试。机器人已经开始的任何东西——特别是更新——没有这个连接也继续。

### `silence` — 机器人安静诊断

```rust
fn silence(idle: Duration) -> String
```

预算是静默而非总计，所以"180s 内无回复"会是关于一个跑了十分钟然后停滞的更新的谎言。只有链接还在时才到达：断开是 `dropped`，`LINK_POLL` 一注意到就回答。

---

## API 版本不匹配处理

### `warn_about_skew` — 警告而非拒绝

```rust
fn warn_about_skew(theirs: u8) {
    eprintln!("warning: the robot speaks API v{theirs} and this client speaks v{}, ...");
}
```

**这曾经是拒绝，而拒绝错了两次。**

1. **它读错了它在读的东西。** `API_VERSION` 是一个板子上二进制之间的协议——`robotctl` 和 `updaterd` 来自一个发布版，`updaterd` 对 `Hello` 的精确 `!=` 是强制执行它的东西。笔记本不是板子上的二进制，而且 routinely 比它正在说话的机器人超前一个发布版，正因为它是构建发布版的机器。这链接远端没有任何东西同意拒绝：这个工具从不发 `Hello`，`configd` 对 `net.*` 或 `system.*` 不检查版本，`updaterd` 在 `update.status` 之前不需要握手。所以拒绝阻止的每个调用本来都会被回答。

2. **它在何时该严格上错了。** BLE 是没有网络的机器人的传输，`wifi connect` 是那个机器人获得网络的方式——所以在版本偏差上拒绝带走了修复偏差的命令，在唯一需要它的时刻。一个有陈旧发布版且没有 wifi 的机器人不能被其存在理由就是这个情况的工具给 wifi。

**没有门时真正不匹配的代价**是一个参数形状变了的方法，回来是命名该方法的 JSON-RPC 错误——被打印，通过退出状态报告。那是比这个更差的消息，但是比锁着的门好得多的结果。

---

## CLI 结构

### `Cli` — 全局参数

```rust
struct Cli {
    #[arg(long = "name", id = "robot", value_name = "ROBOT_NAME", global = true)]
    name: Option<String>,  // 按广告名称连接机器人
    #[arg(long, global = true)]
    verbose: bool,         // 打印每行收发，scan 列出所有设备
    #[arg(long, global = true)]
    pin: Option<String>,   // 机器人配对 PIN
    #[command(subcommand)]
    command: Command,
}
```

**`--name` 的 id 显式拼写**而非从字段派生，因为 clap 按 id 键控参数，而 `name` 子命令有一个派生出同一个 id 的位置参数。两个都叫 `name` 时位置参数赢，所以 `--name duck-c51b name leduckpierre` 搜索 `leduckpierre`——它正要设置的名称——然后报告站在它面前的机器人不在范围内。`value_name` 保持帮助行读 `--name <ROBOT_NAME>` 而非把 id 泄露进去。

**`--pin` 没有 `default_value`**，因为默认必须在环境*之后*应用否则它会遮蔽环境：clap 填入 `default_value`，下游没有东西能 then 区分输入的 `000000` 和假设的 `000000`。它在上方帮助文本中拼写出来。

### `Command` — 子命令

```rust
enum Command {
    Scan,                                    // 列出范围内机器人
    Ip,                                      // IPv4 地址到 stdout
    Open { print: bool, port: u16 },        // 浏览器打开控制台
    Status,                                  // 版本握手 + 更新状态
    Version,                                 // API 版本、发布版、修订
    Update(Update),                          // 更新子命令
    Info,                                    // 名称、序列号、运行时间
    Health,                                  // 控制循环健康？
    Wifi(Wifi),                              // Wifi 子命令
    Name { name: String },                  // 重命名机器人
    Reboot,                                  // 重启
    Call { method: String, params: Option<String> },  // 任意方法
}
```

### `Update` — 更新子命令

```rust
enum Update {
    Check { component: String },             // 有更新吗？
    Apply { ... },                           // 安装发布版
    Status,                                  // 每组件状态
    Versions { component: String },          // 板上有哪些发布版
    Log { limit: u32 },                      // 最近更新尝试
    Rollback { component: String },          // 回退到上一个
    Select { version, component: String },   // 激活已在板上的发布版
    Watch,                                   // 跟随进度直到中断
}
```

命名与 `robotctl update` 相同的词相同的顺序，所以在机器人上学到的东西转移到无线电再回来。不同的是组件：`robotctl` 把它作为位置参数因为操作员可能更新模型包，这里它是有默认的标志，因为手机有一个组件要关心而今天机器人恰好有一个。

### `Wifi` — Wifi 子命令

```rust
enum Wifi {
    Status,              // wifi 在做什么
    Scan,                // 机器人能看到的网络
    Connect { ssid, psk: Option<String> },  // 加入网络
    Forget { ssid },     // 忘记存储的网络
}
```

---

## 错误输出设计

### `main` 打印错误而非返回

```rust
#[tokio::main]
async fn main() -> std::process::ExitCode {
    match run().await {
        Ok(()) => std::process::ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("{e}");
            std::process::ExitCode::FAILURE
        }
    }
}
```

**这不是风格偏好。** 返回 `Err` 的 `main` 被 Rust 的 `Termination` impl 报告，它 **`Debug`-格式化**错误：这个文件里每个提示都是多行的，字符串上的 `Debug` 把换行渲染为字面 `\n` 并把全部包在引号里。所以写来被当作行读的指导到达时是一个转义 blob——对列出无线电看到了什么的失败最差，那是十几行。

---

## 测试要点

文件包含约 50 个测试，覆盖：

### 名称匹配
- `a_single_name_answers_to_itself` — 普通情况
- `either_half_of_a_macos_composite_answers` — macOS 复合名称任一半都匹配
- `the_split_needs_the_shape_it_looks_for` — 拆分需要它寻找的形状

### 设备列表
- `other_devices_are_listed_when_they_are_the_diagnosis` — 其他设备在是诊断时被列出

### 选择逻辑
- `a_name_selects_the_one_robot_that_answers_to_it` — 名称选择回答它的那个机器人
- `a_name_matching_two_robots_is_refused_rather_than_guessed` — **安全规则**：一个名称匹配两个机器人时拒绝而非猜测（bootloader 空 serial-number → 所有板子都叫 radxa-zero3）
- `a_collision_on_a_default_says_where_the_name_came_from` — 默认上的碰撞说名称来自哪里
- `without_a_name_the_first_candidate_still_wins` — 没有名称时第一个候选仍然赢
- `a_name_nobody_answers_to_lists_the_robots_that_were_there` — 没人回答的名称列出在那里的机器人

### 扫描停止逻辑
- `a_named_robot_is_waited_for_rather_than_the_first_one_reported` — **这个替换的 bug**：命名机器人被等待而非第一个报告的
- `a_miss_and_a_collision_are_not_the_same_failure` — 错过和碰撞不是同一失败
- `without_a_name_the_first_candidate_still_stops_the_scan` — 没有名称时第一个候选仍然停止扫描

### Target 环境变量
- `the_environment_names_the_robot_when_the_flag_does_not` — 环境在标志不命名机器人时命名它
- `the_flag_beats_the_environment` — 标志击败环境
- `an_empty_value_is_no_default_at_all` — 空值根本不是默认
- `a_rename_says_when_it_leaves_the_default_stale` — 重命名说何时它让默认变陈旧
- `a_listing_marks_the_robot_the_default_names` — 列表标记默认命名的机器人

### 地址
- `the_console_url_is_the_address_and_the_port` — 控制台 URL 是地址和端口
- `an_advertised_address_is_chosen_by_name` — 广告地址按名称选择
- `a_robot_with_no_network_is_told_how_to_get_one` — 无网络机器人被告知怎么获得一个
- `a_robot_broadcasts_where_it_is` — 机器人广播它在哪里
- `no_wifi_and_no_field_read_differently` — 无 wifi 和无字段读起来不同
- `only_a_robot_is_read_for_an_address` — 只有机器人被读地址（0xFFFF 公司 ID 对非机器人是别人的字节）

### PIN
- `the_pin_falls_back_through_the_environment_to_the_factory_default` — PIN 通过环境回退到出厂默认

### CLI 解析
- `a_rename_still_selects_the_robot_by_the_name_it_has_now` — 重命名仍然按机器人现在有的名称选择它
- `open_takes_a_port_and_can_print_instead` — open 接受端口并可以改为打印

### 更新
- `apply_asks_for_the_target_the_flags_named` — apply 要求标志命名的目标（五种 JSON 形状）
- `a_ref_and_a_version_cannot_both_be_named` — ref 和 version 不能都命名
- `every_update_command_asks_for_its_own_method` — 每个更新命令要求自己的方法
- `select_names_a_version_and_defaults_the_component` — select 命名版本并默认组件
- `an_update_is_given_the_longest_silence` — 更新被给予最长静默
- `progress_prints_as_a_line` — 进度打印为一行
- `a_restart_is_announced_only_when_the_release_changed` — 重启只在发布版改变时宣布
- `a_drop_during_an_apply_points_at_the_record` — apply 期间断开指向记录
- `the_link_is_checked_long_before_a_wait_gives_up` — 链接在等待放弃前很久被检查

---

## 与其他模块的关系

- **`btd::adv`**：广告数据解析（公司 ID、地址字段）
- **`btd::framing`**：分块/重组，客户端与机器人共用同一模块
- **`btd::gatt`**：`SERVICE_UUID`（duck 服务）和 `RPC_UUID`（RPC 特征）
- **`duck_ipc_proto`**：JSON-RPC 方法名、参数类型、`API_VERSION`、`UPDATE_MAX_SILENCE_SECONDS`
- **`btleplug`**：跨平台 BLE 库（CoreBluetooth/BlueZ/WinRT）
- **`clap`**：CLI 解析
- **`serde_json`**：JSON-RPC 序列化/反序列化
- **`webbrowser`**：打开浏览器
- **`tokio`**：异步运行时

---

## 关键踩坑点总结

1. **CoreBluetooth 严格服务过滤**：`ScanFilter { services: [SERVICE_UUID] }` 会让已配对机器人（停止广告服务）完全不可见，必须用无过滤扫描然后在应用层区分。

2. **macOS 复合名称**：`radxa-zero3 [duck-c51b]`——GAP 名称和广告名称被 btleplug 连接在一起，必须接受任一半。

3. **空环境变量是未设置**：`DUCK_ROBOT=` 必须被当作未设置而非空字符串，否则无法在有别人机器人的工作台上转义。clap 自己的 `env` 支持做不到这一点。

4. **一个名称匹配两个机器人时拒绝**：bootloader 空 serial-number → 所有板子都叫 `radxa-zero3`，猜一个可能把 wifi 密码发到错误的机器人。

5. **先读触发配对**：just-works 加密不认证，写需要加密链接但订阅不需要——不先读的话写被静默拒绝。

6. **确认写入**：`WithoutResponse` 的拒绝不可见，请求静默丢失。

7. **空闲超时而非总计超时**：更新可能跑十分钟，每个进度通知重新启动时钟。

8. **链接断开 vs 机器人安静**：macOS 上断开的外围设备让通知流挂起，必须每 2s 轮询 `is_connected()`。

9. **API 版本不匹配警告而非拒绝**：笔记本 routinely 比机器人超前，拒绝会带走 `wifi connect`——修复不匹配的命令。

10. **20 字节分块地板**：btleplug 不暴露 MTU，用 BLE 保证的 20 字节地板。

11. **错误用 Display 而非 Debug 打印**：多行提示被 Debug 格式化为转义 blob。

12. **更新后 btd 重启是正常的**：apply 回答后约 5 秒 `btd` 重启，链接断开是成功更新的形状之一。
#（注：内容由AI生成）
