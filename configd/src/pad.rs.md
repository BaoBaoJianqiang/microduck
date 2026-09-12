# pad.rs 文件解析

## 1. 文件定位

- **路径**：`configd/src/pad.rs`
- **角色**：把「游戏手柄」抽象为 `Pads` trait + 内存假实现 `FakePads`，并包含**判断一个蓝牙设备是否为手柄**的纯函数启发式 `looks_like_a_gamepad` 与配对超时裁剪函数 `pair_timeout`。生产实现是 Linux 下的 `bluez::BlueZ`。

## 2. 核心设计决策（模块文档）

### 为什么配对归 `configd` 而不是 `padd`

- `padd` 读手柄并把 intent 发到 `robotd` socket，**完全没有特权访问**——这正是它独立成进程的主要理由。它每天走的是手机 App 与 SDK 将使用的同一套 API，使该 API 无法悄悄腐化；让它去配置 BlueZ 会恰好毁掉这一属性。
- 所以配对与 wifi 放在一起、理由相同：它是关于无线电的*配置*问题、需要 root，且必须在机器人本体不工作时仍可应答（`architecture.md` §3.1）。它也因此能从手机到达（`btd` 把 `pad.*` 转发到本 socket）。

### 替代了什么

- 以前配对手柄意味着知道 MAC、按不直观的顺序敲三条 `bluetoothctl`（`connect` 在 `pair` 之前；先 `pair` 会得到 `AuthenticationCanceled`）。现在该顺序与理由集中在 `crate::bluez` 一处，而不是活在预置脚本注释和某人的 shell 历史里。
- trait 的存在理由同 `Net`：测试套件跑在没有无线电的笔记本上，值得测的是分发、授权与「这些设备里哪个是手柄」的判断，而非 BlueZ。

## 3. 类型、常量与函数

### `pub type PadResult<T> = Result<T, String>`

错误以可行动字符串表达；与 `Net` 同一套「`Err`=机器坏了，`Ok(Failed)`=业务拒绝」的分工。

### 超时常量

| 常量 | 值 | 说明 |
| --- | --- | --- |
| `DEFAULT_PAIR_TIMEOUT` | 15 秒 | 调用方未指定时的寻找时长。约是一个按着同步键的人愿意等待、判定失败的时间；够 BlueZ 上报每隔几秒才广播一次的设备，又不至于让手机一直转圈 |
| `MAX_PAIR_TIMEOUT` | 120 秒 | 调用方可要求的最长窗口。这是上限而非礼貌：整个窗口内发现都开着，允许一小时会在人走后让适配器久久扫描 |

### `pub trait Pads: Send + Sync`

| 方法 | 语义 |
| --- | --- |
| `status() -> PadResult<Vec<Pad>>` | 所有已绑定手柄，已连接的排前 |
| `pair(mac: Option<&str>, timeout: Duration) -> PadResult<PadPairResult>` | 与处于配对模式的手柄（或指定 `mac`）配对。*拒绝*（没找到、两个候选、BlueZ 说不）是携带 `Failed` 的 `Ok`；`Err` 专留给机器损坏 |
| `forget(mac: &str) -> PadResult<PadForgetResult>` | 删除绑定，使手柄不再回连 |

### `pub fn pair_timeout(requested: Option<u32>) -> Duration`

- `None` → `DEFAULT_PAIR_TIMEOUT`。
- `Some(seconds)` → `Duration::from_secs(seconds).min(MAX_PAIR_TIMEOUT)`。
- **零表示「只看一次」而非「永远等待」**：脚本式重试循环需要能立即询问手柄在不在，而不挂起发现。

### `pub fn looks_like_a_gamepad(name, icon, class, appearance) -> bool`

四类信号综合判断，因为没有任何单一信号在每个手柄上都存在；过窄则配对中的手柄不可见，过宽则会把同事的耳机绑给机器人：

1. **`icon`**：BlueZ 依据 class 或 appearance 给出的自身分类；为 `input-gaming` 即直接判定。这是 Xbox 手柄在板上实际呈现的信号（来自 LE appearance，该手柄没有 class）。
2. **`class`（BR/EDR CoD）**：位 8–12 是主设备类，`0x05` 为外设；minor 的位 2–5 中 `0x01`（joystick）、`0x02`（gamepad）命中，且键盘位（`0x10`）与指点设备位（`0x20`）不得命中。具体实现：`major = (class >> 8) & 0x1f`，`minor = (class >> 2) & 0x3f`，要求 `major == 0x05 && (minor & 0x0f) ∈ {0x01,0x02}`。该分支来自规范，迄今所有手柄都是 LE-only，真机上从未触发。
3. **`appearance`（BLE）**：类别 15（`0x03C0..=0x03C4`）是 HID，但**只有 `0x03C4`（Gamepad）算数**；蓝牙键盘（如 `0x03C1`）同属类别 15，不能被当成手柄。
4. **名字（最后手段，大小写不敏感）**：包含 `controller`、`gamepad`、`joystick`、`dualsense`、`dualshock` 任一即命中。它在其他三者缺失时有效（配对中的手柄常如此），也是唯一可能误判的信号。

都不满足则不作为手柄。`mac` 是配对不被识别硬件的逃生口，使该启发式不必成为承重假设。

## 4. `FakePads`：内存手柄集合

供测试与 `--fake-pads` 使用，让整个 `pad.*` 表面（包括「两个手柄同时配对」「一个都没有」这类需真机才能摆出来的失败）可在无无线电的笔记本上演练。

- `FakeState { visible: Vec<proto::Pad> }`：无线电能看到的所有手柄（绑不绑定都在），这正是 BlueZ 上报的形状。把「配对中」与「已绑定」分开建模看似更整洁，却会掩盖关键 bug：**已绑定手柄出现在每次扫描中**，因此选择规则必须优先未绑定者，否则机器人永远无法再加第二个手柄。
- 构造器：`new()`（一个处于配对模式的未绑定 Xbox 手柄，MAC `78:86:2E:BB:13:28`）、`with(visible)`、`Default`。
- 辅助构造：`pub fn unpaired(mac, name)`（`paired/trusted/connected` 全 false）与 `pub fn bonded(mac, name)`（三者全 true）。

### trait 实现行为

- `status`：只返回 `visible` 中 `paired` 的手柄。
- `pair`：
  1. 按 `mac` 过滤候选（`None` 时全部；`Some` 时忽略 ASCII 大小写比对 MAC）。
  2. 在候选中挑出**未绑定**的 `fresh`：
     - 恰好 1 个 fresh → 选它；
     - 候选为空 → `Failed { NotFound }`；
     - 无 fresh 但有已绑定候选 → 幂等重跑：选 MAC 最小者（也用于修复丢失的 `Trusted`）；
     - 多个 fresh → `Failed { Ambiguous, "{n} pads are in pairing mode" }`。
  3. 选中者置 `paired/trusted/connected = true`，替换列表中旧条目后返回 `Paired { pad }`。
- `forget`：按 MAC（忽略大小写）从 `visible` 移除（`RemoveDevice` 会连对象一起删，手柄在被重新发现前彻底不可见），`removed` 反映是否真的少了条目。

## 5. 单元测试说明

| 测试 | 验证内容 |
| --- | --- |
| `bluez_own_classification_is_enough` | icon 为 `input-gaming` 即判定手柄 |
| `a_peripheral_joystick_class_is_a_gamepad` | CoD `0x000504`（外设+joystick）、`0x000508`（+gamepad）命中 |
| `keyboards_and_mice_are_not_gamepads` | `0x000540`（键盘）、`0x000580`（鼠标）、以及名为 `WH-1000XM4`/`audio-headset`/音视频主类的耳机均**不**命中 |
| `only_the_gamepad_appearance_counts` | appearance `0x03C4` 命中，`0x03C1`（键盘）不命中 |
| `a_pad_is_recognised_by_name_when_nothing_else_is_set` | Xbox/8BitDo/DualSense/Joystick 等仅凭名字命中；`Pierre's iPhone` 不命中 |
| `pairing_bonds_the_pad_and_forgetting_removes_it` | 完整弧线：配对成功且 **trusted 而非仅 paired**（仅 paired 重启后不回连）；forget 后 removed，再次 forget 为 false 且非错误 |
| `an_absent_pad_is_not_found_rather_than_an_error` | 空列表配对报业务失败 `NotFound`（答案应是「按住同步键」而非「报 bug」） |
| `a_bonded_pad_does_not_block_pairing_a_new_one` | **已绑定手柄不得阻止新增**：fresh 的新手柄必须胜出，且最终拥有两个手柄 |
| `re_running_with_nothing_new_reports_the_pad_already_bonded` | 无新设备时重跑返回已绑定手柄并恢复 `trusted`（命令幂等） |
| `two_pads_in_pairing_mode_are_refused_not_guessed` | 两个 fresh 报 `Ambiguous` 而非乱猜；用显式 MAC 可消除歧义并配上指定者 |
| `a_requested_timeout_is_clamped` | `None→15s`、`Some(30)→30s`、`Some(0)→ZERO`、`Some(9999)→120s` |

## 6. 要点小结

- 配对是配置问题，归特权的 `configd`；读手柄的 `padd` 保持零特权。
- 四类信号（icon / class / appearance / name）识别手柄，显式 MAC 是识别失败时的逃生口。
- 选择候选时**未绑定者优先**，已绑定者既不会阻塞新增手柄，也支持幂等重跑修复 `Trusted`。
- 超时默认 15 秒、上限 120 秒、零表示只看一次。
- 业务拒绝（NotFound/Ambiguous）与机器故障（`Err`）严格区分。
