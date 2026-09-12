# main.rs 文件解析

## 文件位置

`d:\microduck\duckctl\src\main.rs`

## 核心设计决策

`duckctl` 是从笔记本电脑操控机器人的工具，是手机 App 的替代品，也是唯一能在真实无线电上测试 `btd` 的途径。

### 1. 蓝牙只是到达机器人的手段，不是工具的本质

今天通过 BLE 到达机器人，但 `mediad` 还提供了 WebRTC 作为第二条传输通道，且按设计到达不同的方法集（`robot.move` 经 BLE 被拒绝、经 WebRTC 允许；`net.connect` 反之）。因此命名按"哪台机器人"而非"哪种无线电"。曾用名 `duck-btctl`。

### 2. 机器人上没有任何东西依赖此 crate

这把 `btleplug` 排除在发布版之外。它曾是 `btd` 的一个 example，放在 example 目录下使其依赖自然成为 dev-dependency，从而无法进入发布产物——这是一种真实的保证，靠目录位置的副作用获得。独立成 crate 后直接声明该保证。真正随机器人发布的工具是 `robotctl`，它在板上走 Unix socket。

### 3. 用 `btleplug` 而非 `bluer`

因为这运行在开发者机器上：macOS 上是 CoreBluetooth、Linux 上是 BlueZ、Windows 上是 WinRT。`bluer` 会把客户端限制在 Linux 上，违背了"跨平台一个工具"的初衷。

### 4. 刻意复用 `btd::framing`

此处的分块是机器人所用同一模块的**客户端**半部分。若分帧不对称，此工具就无法工作——这使其成为对协议的真实测试，而非可能"自说自话"的重实现。

### 5. main 自己打印错误而非返回

`main` 返回 `Err` 时，Rust 的 `Termination` impl 会用 `Debug` 格式打印错误，把多行提示里的换行渲染为字面量 `\n` 并包在引号中，使为逐行阅读而写的指引变成一整块转义文本。所以 `main` 改为 `eprintln!("{e}")`。

## 常量分析

| 常量 | 值 | 含义 |
|---|---|---|
| `SCAN_TIME` | 8s | 查找机器人的最长时间。BLE 发现确实慢，机器人按 BlueZ 选择的间隔广播，太短会误报"无机器人" |
| `SCAN_POLL` | 250ms | 等待期间轮询扫描结果的间隔。单次快照曾间歇失败（`no robot found` 但下次又成功），轮询让常见情况远少于 1 秒结束 |
| `REPLY_TIMEOUT` | 15s | **空闲**而非总时长的请求超时。每个进度通知都会重置时钟，使 update 可被观察；卡住的镜像仍在数秒内失败 |
| `SLOW_REPLY_TIMEOUT` | 60s | 慢调用预算（`net.connect` 轮询 NM 最长 45s，`net.scan` 重新扫描） |
| `UPDATE_IDLE_TIMEOUT` | `UPDATE_MAX_SILENCE_SECONDS + 60` | 更新的静默预算。推导自 proto 的钩子上限（pre-install 钩子装 ONNX Runtime、约 100MB apt），低于上限会把正常更新误报为机器人停止应答 |
| `FOLLOW_TIMEOUT` | 24h | `update watch` 跟随进度直到中断，一天是任意上限，保持回复循环形态统一 |
| `LINK_POLL` | 2s | 等待期间检查链路是否仍在的间隔。没有它，掉线与机器人静默无法区分，掉线要等满 `UPDATE_IDLE_TIMEOUT` 才报错 |
| `CONNECT_TIMEOUT` | 20s | 连接预算 |
| `DISCOVER_TIMEOUT` | 20s | 服务发现预算 |
| `READ_TIMEOUT` | 15s | 首次读取 API 版本预算，此读触发配对 |
| `LISTED_DEVICES` | 12 | 失败列出设备数上限，适配终端 |
| `DEFAULT_PIN` | `"000000"` | 工厂默认 PIN，每个机器人在被设置前都用它 |

## 类型与关键函数

### `Address` 枚举

机器人关于其 IPv4 地址所说的内容，是**三种**而非两种答案：
- `At(Ipv4Addr)` — 广播了具体地址
- `Unassigned` — 字段存在且为 `0.0.0.0`，机器人无网络
- `Unsaid` — 完全没有该字段，旧版 `btd` 或非机器人设备

用 `Option<Ipv4Addr>` 会把后两种空白合并，但它们指向不同的下一步：无网络是 wifi 问题，无字段是需更新。`note()` 中 `Unsaid` 渲染为空而非"unknown"，避免满屏耳机列都显示"unknown"。

`Address::read` 仅在广播了 duck service UUID 时才解析公司数据 `0xFFFF` 中的地址，因为 `0xFFFF` 是 SIG 向所有人开放的，非机器人设备上的四字节是别人的。

### `Target` 结构体

要连接的机器人，以及是否有人输入了名字。
- `name: Option<String>` — 要找的名字，空字符串不算名字
- `from_env: bool` — 名字是否来自环境而非 `--name`

关键方法：
- `new(flag, var)` — `--name` 优先于 `DUCK_ROBOT`；空值视为未设置（因此不用 clap 的 `env`，它会把 `DUCK_ROBOT=` 当值）
- `wanted()` — 用于匹配的名字
- `marks(local_name)` — 此设备是否是名字指向的
- `source()` — `"DUCK_ROBOT"` 或 `"--name"`
- `provenance()` — 失败时说明名字来源（仅环境来源时非空）
- `stale_after_rename(command)` — 重命名后若 `DUCK_ROBOT` 仍指向旧名则提示

"携带来源而非重算"：默认值让工具更严格（抑制已连接回退层、把"第一个找到"变成"找不到"），所以关于"没人命名的机器人"的每条消息都要说明名字来自哪。

### `answers_to(reported, wanted)`

判断设备报告名是否匹配期望名。

**一个外设可同时以两个名字到达**：CoreBluetooth 暴露缓存的 GAP 名（`CBPeripheral.name`，从 `0x2A00` 读得，BlueZ 适配器别名，主机名派生为 `radxa-zero3`）与广告中的本地名。btleplug 在两者不同时拼接：`radxa-zero3 [duck-c51b]`。`btd` 现在把别名设为广告名，但旧版本与缓存仍可能出现拼接形式。

因此接受任一半：用 `rsplit_once(" [")` 取最后一组作为广告半（`rsplit_once` 保证 GAP 名自身含括号时取最后一组）。

### `choose(found, target)`

从候选中选择要对话的机器人，泛型于载荷以便测试（`Peripheral` 无法脱离无线电构造）。

**核心安全规则**：匹配多个候选的名字被拒绝而非猜测。两台不可区分，任选其一意味着写入落到扫描先报告的那台上——`net.connect` 会把 wifi 密码送给那台。原因（见 `identity.rs`）：bootloader 留空 `serial-number` 的板回退到主机名，从同一镜像刷出的所有板都叫 `radxa-zero3`。

没有名字时第一个候选仍赢——这是省略 `--name` 的本意，变成错误会破坏双板台上的简写。

### `worth_connecting(advertised, named, connected, target)`

发现是否找到了目标，还是应继续听到截止。

- 无名字：任一非空层即停止（已配对机器人可能不再广播服务，等满截止只是八秒空等）
- 有名字：必须等到有设备应答该名字才停。否则会停在无线电先报告的机器人上，导致 `scan`（跑满截止）与命名命令对"在范围内"的判断不一致

### `step(what, hint, budget, f)`

带预算运行一步，超时时命名该步骤。btleplug 不约束这些阶段，没有它"connecting to …"后什么都不打印。

### `run()` 主流程

1. 解析 CLI，建立 `Target` 与 `pin`
2. `ScanFilter::default()` 无过滤扫描——曾用 `ScanFilter { services: [SERVICE_UUID] }`，但 CoreBluetooth 严格执行过滤，已配对机器人常以空服务列表报告，导致 `--name` 回退层永远匹配不到
3. 构建三层候选：`advertised`（带服务 UUID）→ `named`（名字匹配但无 UUID）→ `connected`（已连接，仅无名字时）
4. `list_only`（scan）跑满截止并打印列表；否则在 `worth_connecting` 或截止时停止
5. `resolving`（ip/open）优先从广告取地址，无则回退连接问 `net.status`
6. 连接 → 服务发现 → 找特征 → **先读** API 版本（触发配对加密）→ 订阅 → **认证** PIN → 发请求 → 重组 NDJSON 回复
7. 空闲超时重置；`next_chunk` 同时检查链路掉线（macOS 掉线后 `notifications()` 不结束，必须轮询 `is_connected`）

### 关键流程细节

- **先读后订阅再写**：机器人写需认证加密链接，但订阅不需要加密；不先读，中央会愉快订阅、首次写被拒、macOS 上既无提示也无错误。读被确认，未配对链接在此失败从而触发 CoreBluetooth 配对
- **API 版本不拒绝，仅警告**：`API_VERSION` 是单块板上二进制之间的协议，笔记本不是板上二进制且常领先于机器人版本。`configd` 对 `net.*`/`system.*` 不检查版本，`updaterd` 在 `update.status` 前不需握手。真正的不匹配代价是方法参数形状变化，返回 JSON-RPC 错误命名该方法——比锁死的门更好
- **认证用 `WriteType::WithResponse`**：ATT Write Command 无回复，拒绝（如加密不足）不可见，请求静默不到达
- **分块 20 字节**：btleplug 不暴露协商 MTU，20 字节是每条 BLE 链路保证的下限；好链路上偏慢，所有链路上正确

### `Command` 枚举（子命令）

`Scan`、`Ip`、`Open { print, port }`、`Status`、`Version`、`Update(Update)`、`Info`、`Health`、`Wifi(Wifi)`、`Name { name }`、`Reboot`、`Call { method, params }`

`Update` 子命令：`Check`、`Apply`（支持 `--version`/`--ref`/`--staging`/`--dry-run`，version 与 ref 互斥）、`Status`、`Versions`、`Log`、`Rollback`、`Select`、`Watch`

`Wifi` 子命令：`Status`、`Scan`、`Connect { ssid, psk }`、`Forget { ssid }`

`--name` 参数的 id 显式设为 `"robot"`：因为 `name` 子命令的位置参数会派生出同名 id，两者都叫 `name` 时位置参数胜出，导致 `--name duck-c51b name leduckpierre` 去搜索 `leduckpierre`。

### `request_line(command)`

把一个命令变成一条 JSON-RPC 行 + 等待时长。update 命令用 `duck_ipc_proto` 自身类型构造（`Target` 是外部标签枚举，手写形状易错成 `PARSE_ERROR`）。

### `dropped(command)` 与 `silence(idle)`

链路掉线与机器人静默的不同诊断。对 update 尤其重要：`btd` 在 apply 应答后约 5 秒重启（`docs/design/restart-order.md` §1），掉线是**成功**更新的形态之一，说成"机器人停止应答"是把工作正常的机器人描述成死的。

## 单元测试描述

- `a_single_name_answers_to_itself` — Linux 单名普通情况
- `either_half_of_a_macos_composite_answers` — macOS 拼接名的两半都可匹配
- `other_devices_are_listed_when_they_are_the_diagnosis` — 无机器人时其他设备是诊断，必须列出
- `a_name_selects_the_one_robot_that_answers_to_it` — 名字选中唯一应答的机器人，含拼接形式
- `a_name_matching_two_robots_is_refused_rather_than_guessed` — **安全规则**：同名两机器人被拒绝，列出两者并提示 `set-name`
- `a_collision_on_a_default_says_where_the_name_came_from` — 环境来源的碰撞提示 `DUCK_ROBOT`
- `without_a_name_the_first_candidate_still_wins` — 无名字时第一个赢，保持简写
- `a_name_nobody_answers_to_lists_the_robots_that_were_there` — 无人应答时列出在场的机器人
- `a_named_robot_is_waited_for_rather_than_the_first_one_reported` — 有名字时等名字出现而非停在第一个报告的
- `a_miss_and_a_collision_are_not_the_same_failure` — 未命中与碰撞是不同失败
- `without_a_name_the_first_candidate_still_stops_the_scan` — 无名字时第一个候选仍停止扫描
- `a_rename_still_selects_the_robot_by_the_name_it_has_now` — 重命名用现名选机器人
- `the_environment_names_the_robot_when_the_flag_does_not` — 环境命名机器人并说明来源
- `the_flag_beats_the_environment` — `--name` 优先于环境
- `an_empty_value_is_no_default_at_all` — 空值不算默认（`DUCK_ROBOT=` 转义）
- `a_rename_says_when_it_leaves_the_default_stale` — 重命名后提示 `DUCK_ROBOT` 过期
- `a_listing_marks_the_robot_the_default_names` — 列表标记默认名指向的机器人
- `the_console_url_is_the_address_and_the_port` — 控制台 URL 格式
- `an_advertised_address_is_chosen_by_name` — 广告地址按名字选择
- `a_robot_with_no_network_is_told_how_to_get_one` — 无网络机器人被告知 `wifi connect`
- `open_takes_a_port_and_can_print_instead` — `open` 的 `--port` 与 `--print`
- `a_robot_broadcasts_where_it_is` — 机器人广播地址
- `no_wifi_and_no_field_read_differently` — 无 wifi 与无字段渲染不同
- `only_a_robot_is_read_for_an_address` — 仅机器人解析地址
- `the_pin_falls_back_through_the_environment_to_the_factory_default` — PIN 回退链
- `the_split_needs_the_shape_it_looks_for` — 拼接分割需有对应形状
- `apply_asks_for_the_target_the_flags_named` — apply 各目标的 JSON 形状
- `a_ref_and_a_version_cannot_both_be_named` — ref 与 version 互斥
- `every_update_command_asks_for_its_own_method` — 每条 update 命令方法名正确
- `select_names_a_version_and_defaults_the_component` — select 传版本并默认组件
- `an_update_is_given_the_longest_silence` — update 获最长静默预算
- `progress_prints_as_a_line` — 进度打印为一行
- `a_restart_is_announced_only_when_the_release_changed` — 仅实际切换时宣布重启
- `a_drop_during_an_apply_points_at_the_record` — apply 期间掉线指向记录
- `the_link_is_checked_long_before_a_wait_gives_up` — 链路检查远早于等待放弃

## 关键摘要

`duckctl` 是开发侧的机器人客户端，跨 macOS/Linux/Windows，经 BLE 复用 `btd::framing` 与机器人说同一套 JSON-RPC。核心设计围绕"诊断精确性"：把扫描、连接、发现、读取各阶段分别命名超时；用空闲而非总时长做超时使 update 可观察；显式区分"无网络"与"无字段"、"链路掉线"与"机器人静默"；API 版本只警告不拒绝（因为笔记本常领先于机器人，且拒绝会锁死修复版本偏差的 `wifi connect`）。最重要的安全规则是同名多机器人被拒绝而非猜测，避免 wifi 密码送错机器人。
