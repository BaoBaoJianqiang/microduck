# `pad.rs` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Rust 源文件 |
| 行数 | 454 行 |
| 角色 | 游戏手柄——作为 trait + 假实现 |
| 平台 | 跨平台（trait 可测试，真实实现见 `bluez.rs`） |

## 二、为什么 configd 拥有此功能而 padd 不

### padd 的定位

- `padd` 读取手柄并通过 `robotd` 的 socket 发送意图，**完全没有特权访问**——这是它作为独立进程的大部分原因。
- 它每天锻炼与手机应用和 SDK 将使用的相同 API，因此该 API 不能悄悄腐烂。
- 让它配置 BlueZ 会恰好撤销该属性。

### 配对属于 configd

- 因此配对住在这里，与 wifi 并列，原因与 wifi 相同：
  - 它是关于无线电的*配置*问题。
  - 它需要 root。
  - 当机器人本身不工作时它必须可回答（`architecture.md` §3.1）。
- 它也可以从手机到达——`btd` 将 `pad.*` 转发到此 socket——这是"配对控制器"长期所属的位置。

## 三、替换了什么

### 之前

- 配对手柄意味着知道其 MAC 地址并按非显而易见的顺序运行三个 `bluetoothctl` 命令：
  - `connect` 在 `pair` 之前。
  - 先 `pair` 返回 `AuthenticationCanceled`。
- 该顺序现在在一个地方——`crate::bluez`——原因在旁边，而非在配置脚本的注释和某人的 shell 历史中。

### trait 存在的原因

- 与 `crate::net::Net` 相同：测试套件在没有无线电的笔记本上运行，值得测试的逻辑是调度、授权和"这些设备中哪个是游戏手柄"的决定——不是 BlueZ。

## 四、常量

| 常量 | 值 | 说明 |
|---|---|---|
| `DEFAULT_PAIR_TIMEOUT` | 15 秒 | 调用者未指定时查找手柄的时间。15 秒因为有人站在那里拿着同步按钮，这大约是一个人在得出"不工作"结论之前会等待的时间 |
| `MAX_PAIR_TIMEOUT` | 120 秒 | 调用者可要求的最长窗口。上限而非礼貌：发现整个窗口都开着，要求一小时的客户端会在键入它的人走开后很久还让适配器扫描 |

## 五、`Pads` trait

```rust
#[async_trait]
pub trait Pads: Send + Sync {
    async fn status(&self) -> PadResult<Vec<proto::Pad>>;
    async fn pair(&self, mac: Option<&str>, timeout: Duration) -> PadResult<proto::PadPairResult>;
    async fn forget(&self, mac: &str) -> PadResult<proto::PadForgetResult>;
}
```

| 方法 | 说明 |
|---|---|
| `status()` | 此机器人绑定的每个手柄，已连接的在前 |
| `pair(mac, timeout)` | 配对处于配对模式的任何手柄，或 `mac` 指定的那个 |
| `forget(mac)` | 删除绑定，使此手柄停止重连 |

### `PadResult<T> = Result<T, String>`

- 出了什么问题，用调用者可采取行动的术语。

### `pair()` 的拒绝 vs 错误

- **拒绝**（未找到、两个候选、BlueZ 说不）是携带 `PadPairResult::Failed` 的 `Ok`。
- `Err` 保留给机制损坏。
- 与 `crate::net::Net` 相同的拆分，调度器将其转为结果或 `INTERNAL_ERROR`。

## 六、`pair_timeout(requested: Option<u32>) -> Duration`

- 将调用者的超时钳制到适配器应该被要求做的事情。
- `Some(seconds)` → `Duration::from_secs(seconds).min(MAX_PAIR_TIMEOUT)`。
- `None` → `DEFAULT_PAIR_TIMEOUT`。
- **零是"看一次"而非"永远看"**：脚本化重试循环应该能够询问手柄现在是否在那里而不保持发现打开。

## 七、`looks_like_a_gamepad()`——四信号启发式

### 为什么需要启发式

- 没有单一信号出现在每个手柄上，两个方向出错都以不同方式糟糕：
  - 太窄：配对模式中的手柄不可见。
  - 太宽：`pad.pair` 将机器人绑定到同事的耳机。

### 四个信号（按优先级）

#### 1. `icon` — BlueZ 自己的分类

- 从类别或外观派生。
- 当它说 `input-gaming` 时问题解决。
- 这是 Xbox 控制器在此板上实际呈现的信号——从其 LE 外观，因为该手柄没有类别。

#### 2. `class` — BR/EDR 设备类别

- 位 8-12 是主要设备类别，`0x05` 是外设。
- 次要字段的位 6-7 区分键盘、定点设备和游戏手柄：
  - 位 2-5 中的 `0x01` 且键盘/指针位清除 = 操纵杆或游戏手柄。
- 经典手柄存在，仅 BLE 的不存在——到目前为止尝试的每个手柄都是仅 LE 的，因此此分支来自规范且从未在硬件上触发。

#### 3. `appearance` — BLE 等效

- 类别 15（`0x03C0..=0x03C4`）是 HID，`0x03C4` 专门是游戏手柄。
- 许多手柄从不设置它，这就是它不能独立的原因。
- 只有游戏手柄值计数，因此广告通用 HID 的 LE 手柄落到其名称。

#### 4. 名称——最后且故意

- 当其他三个都不存在时工作的信号，这对于仍在配对模式的手柄很常见。
- 它是可能出错的那个。
- 不区分大小写：BlueZ 按设备广告的名称报告，手柄在大写方面不一致。
- 关键词：`controller`, `gamepad`, `joystick`, `dualsense`, `dualshock`。

### 逃生舱口

- 不满足这些的设备不作为手柄提供。
- `mac` 是某人配对此不识别的硬件的方式，这是保持启发式不承重的逃生舱口。

## 八、`FakePads`——内存中的假实现

### 用途

- 被测试和 `--fake-pads` 使用。
- 这使得整个 `pad.*` 表面——以及用真实硬件难以安排的失败，如同时两个手柄在配对模式——可从没有无线电的笔记本锻炼。

### `FakeState`

```rust
struct FakeState {
    visible: Vec<proto::Pad>,
}
```

- 无线电可以看到的每个手柄，绑定与否——这是 BlueZ 报告的形状，也是这是一个列表而非两个的原因。

#### 为什么不分开"配对模式"和"已绑定"

- 分开建模看起来更整洁，但隐藏了重要的 bug：
  - 已绑定的手柄出现在*每次*扫描中，因此选择规则必须偏好未绑定的，否则机器人永远无法被给予第二个手柄。

### 构造函数

| 函数 | 说明 |
|---|---|
| `FakePads::new()` | 一个配对模式中的手柄，无绑定——新鲜机器人旁边控制器同步灯闪烁 |
| `FakePads::with(visible)` | 自定义可见列表 |
| `unpaired(mac, name)` | 配对模式中的手柄：可见，未绑定 |
| `bonded(mac, name)` | 机器人已有的手柄，在范围内且已连接 |

### `pair()` 的选择规则（与 `bluez.rs` 相同）

1. 未绑定的手柄获胜——那是某人刚放入配对模式的那个。
2. 已绑定的手柄不是竞争答案，将其视为一个是使第二个手柄无法添加的原因。
3. 没有新的配对模式 → 回答已绑定的手柄（幂等重运行，也是修复丢失的 `Trusted` 的方式）。
4. 多个未绑定 → 拒绝（`Ambiguous`），而非猜测。

### `forget()`

- `RemoveDevice` 删除对象，不只是键，因此手柄根本不再可见，直到有东西重新发现它。
- 再次忘记不是错误，客户端不能将其呈现为错误。

## 九、测试（11 个）

| 测试 | 验证内容 |
|---|---|
| `bluez_own_classification_is_enough` | `icon = input-gaming` 足够识别 Xbox 控制器 |
| `a_peripheral_joystick_class_is_a_gamepad` | 经典手柄的 class-of-device 识别（合成，从未在硬件上触发） |
| `keyboards_and_mice_are_not_gamepads` | 更重要的方向：键盘/鼠标/耳机不被识别为游戏手柄 |
| `only_the_gamepad_appearance_counts` | BLE appearance 只有 `0x03C4` 计数，其他 HID 不计数 |
| `a_pad_is_recognised_by_name_when_nothing_else_is_set` | 名称回退，不区分大小写；手机不被识别 |
| `pairing_bonds_the_pad_and_forgetting_removes_it` | 完整流程：配对→绑定+信任→忘记；再次忘记不是错误 |
| `an_absent_pad_is_not_found_rather_than_an_error` | 无手柄在配对模式是最常见失败，必须与"东西坏了"区分 |
| `a_bonded_pad_does_not_block_pairing_a_new_one` | **承重测试**：已绑定手柄不阻止配对新的，否则两人想驾驶的机器人只能忘记工作的手柄 |
| `re_running_with_nothing_new_reports_the_pad_already_bonded` | 无新手柄时重运行报告已绑定的，修复丢失的 Trusted，使命令幂等 |
| `two_pads_in_pairing_mode_are_refused_not_guessed` | 两个手柄在配对模式：拒绝而非猜测，命名一个解决它 |
| `a_requested_timeout_is_clamped` | 超时被钳制，零是"看一次"，超大值被钳到 MAX |
#（注：内容由AI生成）
