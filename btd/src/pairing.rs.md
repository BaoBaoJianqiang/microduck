# `pairing.rs` 文件解析 —— 谁有权通过 BLE 与机器人通话

## 1. 文件定位

- 路径：`src/pairing.rs`
- 角色：处理配对 PIN 的获取。§4.2 规定 BLE 授权 = "物理在场 + 配对"，§7 要求携带 wifi 凭据的特征值必须配对并加密。两者都满足，但 PIN 校验位于**链路层之上**——这是被迫的，而非偏好。

## 2. 模块文档：为什么 BLE 自身做不到固定 PIN 校验

最初设计是让机器人用存储的 PIN 应答 BlueZ 的 passkey 请求。在无外设（headless）机器人上不可行：

- LE passkey entry 中，一方**显示** passkey，另一方**输入**，角色由各方声明的 IO 能力决定；
- 实现 `request_passkey` 等于声明"本设备能输入"，于是 macOS 取显示角色，生成随机六位码，等待有人把它键入一台没有键盘的机器人；
- 反过来也失败：用 `DisplayPasskey` 时机器人取显示角色，但**passkey 由 BlueZ 生成**（规范规定显示方随机选择），贴纸上的固定 PIN 根本无法表达。

## 3. 折中方案：just-works 配对 + 传输层 PIN

- 配对采用 just-works（agent 所有处理函数为 `None`，BlueZ 解读为 `NoInputNoOutput`），因此链路**加密但不认证**；
- RPC 特征值的读操作要求加密，从而触发绑定（bond）；
- 之后 `btd` 在客户端通过 `system.authenticate` 证明 PIN 之前不提供任何服务。这也是该调用由传输层自己应答、而非转发的原因（见 `session.rs`）。

明确陈述的代价：PIN 经过一条加密但未认证的链路，**配对当时在场的攻击者可能截获它**。备选方案是完全不认证，或 BlueZ 几乎不支持、也没有手机 App 驱动的带外 QR 流程。对家用机器人这是更好的权衡，而且因为校验现在掌握在自己手中而非规范手中，将来可在不动传输层的前提下重新审视。

- **出厂 PIN 是 `000000`，且在本仓库中人人可读**。因此开箱状态下它只证明物理在场，与 just-works 等价；但机制、存储和六位契约都已就位，把它变成真正的秘密只是一个配网（provisioning）改动，而非重新设计。每台机器人独立、贴在机身下的 PIN 才是真正的安全保障，属于 `updater-design.md` §5.7 的每设备状态。

## 4. 关于"不设配对窗口"

机器人只要在广播就可配对。物理按钮 + 限时窗口是常见做法，经考虑后被否决：

- **每机唯一 PIN 已经携带了窗口能增加的属性**：PIN 唯一且贴在机身下，知道它就需要物理接触，而能读到贴纸的人本来也能把机器人抱起；
- 窗口只能在出厂默认 PIN 仍生效时防御范围内的人，对此的正解是换上真实 PIN，而不是加按钮；
- 按钮额外能提供的（可见的同意时刻、PIN 丢失时的恢复路径、贴纸被拍照时的纵深防御）v1 都不需要，且将来可叠加——带按钮的外壳可以在不改这里的情况下门控 `set_pairable`。

安全因此完全依赖 PIN 每机唯一，这是一项配网义务（生成、打印、记录）。文档特别指出：机器人虽有从 SoC 序列号派生的每设备身份（`configd::identity`），但 PIN **不能**从它派生——该身份是公开的（广播中任何人可采集），由它算出的 PIN 一旦派生方式泄露就等于公开；只有名字挂在身份之下。

仍未解决的较小问题：**没有绑定管理**。所有配对过的手机会一直保持配对，暂无撤销 API，手动手段是 `bluetoothctl untrust`。

## 5. 代码项详解

### 5.1 常量 `PIN_TIMEOUT = 3s`

等待 `configd` 返回 PIN 的时限。取较短值：BlueZ 正持有一次配对交换，手机在显示转圈；`configd` 若 3 秒内答不上来也就不会答了。

### 5.2 `async fn pin(config_socket) -> Result<PairingPinResult, String>`

向 `configd` 请求配对 PIN：

- 通过 `upstream::ask` 发送 `Call::SystemPairingPin`，超时 `PIN_TIMEOUT`，再 `result_as()` 反序列化，错误统一转成字符串；
- **每次配对请求都现取**，不在启动时缓存——使 `robotctl system set-pin` 在下一次配对（而非下次重启）即生效；一次 socket 往返相对于人要花数秒的配对交换可以忽略；
- 原样按字符串返回而非解析为数字：`000042` 与 `42` 是不同 PIN，数字解析会把它们等同；
- `is_default` 一并返回，让调用方能明确说出"出厂 PIN 可认证任何读过本仓库的人"；
- **PIN 绝不写日志**。它今天几乎不算秘密，但每机 PIN 注定是秘密，journal 不是放它的地方。

## 6. 单元测试（4 个）

| 测试 | 验证内容 |
| ---- | -------- |
| `a_pin_with_leading_zeros_is_the_right_passkey` | `000000→0`、`000042→42`、`123456`、`999999` 按 `u32` 解析为正确数值（passkey 在线上只能是数字形态） |
| `an_absent_configd_fails_rather_than_hanging` | `configd` 缺失时返回错误且信息含 `cannot reach configd`，而非挂起 |
| `the_pin_is_fetched_over_the_socket` | 真实 socket 全链路：假 configd 应答，请求方法必须是 PIN 方法；结果按字符串保留前导零（`000042`），`is_default=false` |
| `a_refusal_is_not_treated_as_a_default_pin` | configd 返回 PERMISSION_DENIED 时错误被上报（含 `refused`），不会被吞掉并默认成 `000000`（否则等于放任任何人配对） |

## 7. 本文件要点小结

1. BLE passkey 机制无法表达贴纸上的固定 PIN，故 PIN 校验上移到传输层 `system.authenticate`；
2. 链路层用 just-works 加密（不认证），真正的授权靠六位 PIN；
3. PIN 每次配对现取、按字符串比较、永不入日志；
4. 出厂 PIN `000000` 仅证明物理在场；每机唯一 PIN 是配网义务，且不能从公开的设备身份派生；
5. 不设配对窗口；暂无绑定撤销 API。
