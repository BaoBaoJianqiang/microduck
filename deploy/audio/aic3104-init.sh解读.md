# `aic3104-init.sh` 脚本解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Bash 初始化脚本 |
| 作用对象 | TLV320AIC3104 音频编解码器（codec）的 ALSA 混音器 |
| 职责边界 | **只**应用默认混音器电平和麦克风路由；PLL / DAC / 线路输出配置由内核 codec 驱动通过设备树 overlay 处理 |
| 关联配置 | `robotd.toml` 中 `[audio] device = "plughw:aic3104"`、`setup-board.sh` 启用该 codec |
| 触发时机 | 启动后由相关服务调用（脚本自带 `sleep 2` 与重试循环以应对声卡注册延迟） |

---

## 二、整体执行流程

```
启动
  │
  ├─ sleep 2                        # 等待系统与声卡初始化
  │
  └─ for i in 1..15:                # 最多重试 15 次
       │
       ├─ amixer -c aic3104 info    # 探测声卡是否已注册
       │    │
       │    ├─ 成功 → 依次设置扬声器 / 麦克风 / 捕获路径
       │    │         → echo "TLV320AIC3104 mixer levels set"
       │    │         → break（退出循环）
       │    │
       │    └─ 失败 → sleep 1，进入下一次重试
       │
       └─ 15 次均失败 → 脚本静默结束（不报错、不设置任何电平）
```

**重试窗口设计**：窗口给得较宽（15 次 × 1 秒 + 初始 2 秒 ≈ 最多 17 秒），因为在 Radxa 板卡上，声卡探测被**推迟**到 DKMS codec 模块自动加载之后才完成——`amixer` 在启动早期找不到卡是正常现象，而非故障。

---

## 三、扬声器路径配置

扬声器功放挂在 codec 的线路输出（`LEFT_LOP` / `RIGHT_LOP`）上。脚本设置四项：

| 控件（`amixer cset name=...`） | 值 | 含义 |
|---|---|---|
| `PCM Playback Volume` | `127,127` | PCM 数字播放音量，左右声道均拉满 |
| `Line DAC Playback Volume` | `118,118` | 线路 DAC 播放音量，左右声道 118 |
| `Line Playback Switch` | `on,on` | **LOP 输出级 MUTE 开关**（见下方踩坑点） |
| `Line Playback Volume` | `9,9` | LOP 级增益，范围 **0..9 dB**，左右声道均设为 9 dB |

### ⚠ 历史踩坑点：`Line Playback Switch` 曾被设为 `off`

- **这个开关是 LOP 输出级的 MUTE，不是 line-in 旁路开关**。
- 脚本**曾经把它设为 `off`**，结果是**整机静音**——机器人完全发不出声音。
- **为什么树莓派（Pi）上没暴露这个 bug**：`alsa-restore` 服务恰好重新应用了一份旧的、已保存的状态，而那份状态里这个开关是 `on`——等于用历史状态"修复"了脚本的错误，掩盖了问题。
- 当前版本明确设为 `on,on`，并在注释中永久记录了这个教训，防止后人再次误判其语义。

---

## 四、板载麦克风路由配置

目标：**只把 `Mic3R` 路由到 Right PGA（右声道可编程增益放大器）**，其余所有输入源全部关闭。

| 控件（`amixer sset ...`） | 值 | 作用 |
|---|---|---|
| `Right PGA Mixer Mic3R` | `on` | **唯一开启**：Mic3R 接入右 PGA |
| `Left PGA Mixer Mic3R` | `off` | 禁止 Mic3R 进入左 PGA |
| `Left PGA Mixer Mic3L` | `off` | 禁止 Mic3L 进入左 PGA |
| `Right PGA Mixer Mic3L` | `off` | 禁止 Mic3L 进入右 PGA |
| `Right PGA Mixer Line1R` | `off` | 禁止 Line1R 进入右 PGA |
| `Right PGA Mixer Line1L` | `off` | 禁止 Line1L 进入右 PGA |
| `Left PGA Mixer Line1R` | `off` | 禁止 Line1R 进入左 PGA |
| `Left PGA Mixer Line1L` | `off` | 禁止 Line1L 进入左 PGA |
| `Right PGA Mixer Line2R` | `off` | 禁止 Line2R 进入右 PGA |

**设计意图**：显式关闭所有未使用的输入源，避免悬空输入引入噪声、串扰或意外的立体声混合。板载麦克风只走单声道（右 PGA），这与 `robotd.toml` 中麦克风从 `<device>,0` 采集、用于抚摸检测（`pet_detect`）的用途一致。

---

## 五、捕获路径配置

| 控件（`amixer cset name=...`） | 值 | 含义 |
|---|---|---|
| `PGA Capture Switch` | `on,on` | 启用 PGA 捕获开关（左右声道） |
| `PGA Capture Volume` | `60,60` | PGA 捕获增益，范围 **0..119**，左右声道均设为 60 |

- 增益 60 是一个中间偏保守的值，兼顾灵敏度与底噪；抚摸检测分类器在该增益下工作。
- 与 `robotd.toml` 的 `[audio]` 段配合：codec 启用后，`pet_detect` 分类器才可能从板载麦克风读取音频。

---

## 六、关键设计决策总结

| 决策 | 理由 |
|---|---|
| 职责收窄（只设 mixer，不碰 PLL/DAC） | 内核驱动 + 设备树 overlay 已处理硬件层配置，脚本重复设置会与驱动冲突且不可移植 |
| 15 次重试 + `sleep 2` 预热 | Radxa 上声卡探测被推迟到 DKMS 模块自动加载，启动早期 `amixer` 找不到卡是常态 |
| 显式关闭所有未用麦克风输入 | 防止悬空输入噪声与串扰，确保单声道麦克风路由干净 |
| `Line Playback Switch` 注释永久记录踩坑 | 该开关语义易被误读为 line-in 旁路，曾导致整机静音；注释防止回归 |
| 失败时静默退出（不报错） | 由调用方/日志判断是否成功；脚本本身不做错误传播 |

---

## 七、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `robotd.toml` `[audio] device = "plughw:aic3104"` | robotd 的 ALSA 播放设备名正是本脚本初始化的这张卡；麦克风从 `<device>,0` 采集 |
| `setup-board.sh` | 负责启用 TLV320AIC3104 codec（设备树 overlay / DKMS 模块），是本脚本能找到卡的前提 |
| `alsa-restore` | 曾无意中"修复"过 `Line Playback Switch=off` 的 bug，也是为什么该 bug 长期未被发现 |
| `pet_detect.onnx` | 抚摸检测模型依赖本脚本配置的麦克风捕获路径与增益 |

---

## 八、配置速查表

| 类别 | 控件 | 值 | 范围/备注 |
|---|---|---|---|
| 扬声器 | `PCM Playback Volume` | `127,127` | 数字音量拉满 |
| 扬声器 | `Line DAC Playback Volume` | `118,118` | 线路 DAC 音量 |
| 扬声器 | `Line Playback Switch` | `on,on` | LOP 输出级 MUTE（**必须 on，否则静音**） |
| 扬声器 | `Line Playback Volume` | `9,9` | LOP 增益，0..9 dB |
| 麦克风路由 | `Right PGA Mixer Mic3R` | `on` | 唯一开启的输入源 |
| 麦克风路由 | 其余 8 个 Mixer | `off` | 全部显式关闭 |
| 捕获 | `PGA Capture Switch` | `on,on` | 启用捕获 |
| 捕获 | `PGA Capture Volume` | `60,60` | 增益，0..119 |
| 重试 | 循环次数 | 15 | 每次间隔 1 秒，前置 `sleep 2` |
#（注：内容由AI生成）
