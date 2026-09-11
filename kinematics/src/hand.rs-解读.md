# hand.rs 源码解读与架构梳理

> 分析对象：`hand.rs`（357 行），kinematics crate 的特雷门琴手势追踪器——从 ToF 8×8 区域中找到"喙前方最近的东西"，作为乐器想要的输入。
> 分析方法：SCOPE → ROUTE → EFFECT → BREAK → SHIP。

---

## 一、执行摘要

**核心判断**：`hand.rs` 提供 `Tracker`——从 ToF 传感器的 8×8 距离帧中检测"手"，输出距离、接近度（closeness）、覆盖区域数、是否为保持记忆。这是一个**从复杂到简单的教训**：第一版很聪明——背景捕获、平面拟合排除墙壁、慢漂移让房间可重新布置——全部在测试中工作，但没有一个在鸭子上活下来，原因值得写下来：**传感器的输入不够稳定，无法做那么复杂的推理。** 哪些区域携带可用状态逐帧变化，半秒前捕获的背景每次都不同，同一个手势这一刻 arm 下一刻被拒绝。在噪声输入上叠加聪明会放大噪声，而非过滤它。

所以这是简单的东西：特雷门琴是显式模式，没有背景也没有 arming。乐器举起时，可演奏频段内最近的回波就是手。指向空旷空间它静音，指向 40cm 外的墙它演奏稳定音符——这是正确的，因为你故意打开了模式，故意打开的模式允许做明显的事。

**关键设计决策**：
- **哪些 status 字节算数**：ST 称 5 和 9 为"valid"，但在这个传感器 15Hz 下，超过 ~30cm 的手经常返回 4 或 13——*一致性失败*，sigma 太高——但携带的距离对音高完全够用。只接受 5 和 9 是第一版死在 30cm 的原因。可接受集合是 `Config::statuses`，是配置而非常量。
- **保持（hold）**：在 15Hz 下，一个在可用/不可用之间闪烁的区域会把音符切成碎石。`Config::hold` 保持最后一只手几帧，掉帧听不见，而手真正离开时音符仍及时停止。
- **低百分位而非最小值**：单区域飞点（比真实值短几厘米）在这里很常见，作为音高输入会是一声啁啾。用 `in_band[len/5]`（20 百分位）而非最小值。
- **没有地板过滤和重投影**：故意的。两者都需要头部 FK 和 IMU，都是让音符因为玩家看不见的原因消失的另一种方式。一个演奏原始光束的乐器是可以预测的。

---

## 二、身份与范围（SCOPE）

| 项 | 内容 |
|----|------|
| 对象 literal path | 附件 hand.rs |
| 文件类型 | Rust 公开模块（pub mod hand，lib.rs L24） |
| 所属 crate | kinematics |
| 行数 / 已读范围 | 357 行，全文已完整读取（L1–L357） |
| 主要证据 | 文件本身；tof.rs（ROWS/COLS）；math.rs（未直接使用） |
| 不可读 / 未提供 | 特雷门琴音频合成代码、tofd 协议、第一版实现的 git 历史 |

---

## 三、证据矩阵

| # | 事实 | 定位 | 观察 | 状态 |
|---|------|------|------|------|
| F1 | 第一版有背景捕获/平面拟合/慢漂移，全部在鸭子上失败 | L3–L11 | 模块文档 | confirmed |
| F2 | 失败原因：传感器输入不稳定，聪明放大噪声 | L7–L11 | 模块文档 | confirmed |
| F3 | 当前版本：无背景/无 arming，频段内最近回波就是手 | L13–L17 | 模块文档 | confirmed |
| F4 | Config：near_m=0.10, far_m=0.70, min_zones=2, statuses=[4,5,6,9,10,12,13], hold=250ms | L62–L77 | Default impl | confirmed |
| F5 | near_m=0.10 是可信度下限（盖玻璃串扰），不是品味 | L46–L48 | 注释 | confirmed |
| F6 | statuses 包含 7 个状态码，不只是 ST 的 5/9 | L68–L73 | 注释解释每个码 | confirmed |
| F7 | Hand：range_m, closeness, zones, held | L87–L102 | 结构体 | confirmed |
| F8 | closeness：far_m→0, near_m→1，越近越高（音高和嘴都向这个方向） | L92–L95 | 注释 | confirmed |
| F9 | Tracker 有状态：last: Option<(Hand, Instant)> | L108–L112 | 结构体 | confirmed |
| F10 | track() 接受原始 distance_mm 和 status 数组（可能比网格短） | L129 | 签名 + 注释 | confirmed |
| F11 | 负距离是收敛失败，不是测量 | L135–L137 | 注释 + 实现 | confirmed |
| F12 | in_band 区域数 < min_zones 时触发 hold 逻辑 | L146–L158 | 实现 | confirmed |
| F13 | hold 从最后*看到*的手计时，不是从第一次掉帧 | L150, L294–L308 | 实现 + 测试 | confirmed |
| F14 | range_m = in_band[len/5]（20 百分位），排序后 | L160–L163 | 实现 | confirmed |
| F15 | closeness = (far_m - range_m) / (far_m - near_m)，clamp [0,1] | L164–L167 | 实现 | confirmed |
| F16 | reset() 清除保持的手（放下乐器时） | L175–L179 | 实现 | confirmed |
| F17 | status_histogram()：诊断用，按出现次数排序 | L182–L198 | 函数 | confirmed |
| F18 | 测试：所有默认 status 码都能演奏（pin 第一版的 bug） | L219–L239 | 测试 | confirmed |
| F19 | 测试：closeness 越近越高，端点精确 0/1 | L243–L260 | 测试 | confirmed |
| F20 | 测试：掉帧保持音符，手真正离开停止 | L264–L292 | 测试 | confirmed |
| F21 | 测试：持续看到的手永不过期（hold 从最后看到的手计时） | L296–L308 | 测试 | confirmed |
| F22 | 测试：一个杂散区域不是手 | L311–L319 | 测试 | confirmed |
| F23 | 测试：坏帧（负距离/短数组/不匹配）不是音符 | L324–L337 | 测试 | confirmed |
| F24 | 测试：直方图按出现次数排序，不越界读 | L342–L355 | 测试 | confirmed |

---

## 四、源码逐段解读

### 4.1 模块文档（L1–L35）

这是整个文件最重要的部分——记录了从复杂到简单的设计演变和教训：

**第一版为什么失败**：
- 背景捕获：特雷门琴 arm 时捕获背景，让鸭子面对墙也能区分手和墙。
- 平面拟合：排除墙壁被误认为一只巨大的手。
- 慢漂移：让房间可以重新布置。
- 全部在测试中工作，没有一个在鸭子上活下来。
- **原因**：传感器输入不够稳定。哪些区域携带可用状态逐帧变化，半秒前的背景每次都不同，同一个手势这一刻 arm 下一刻被拒绝。
- **教训**：在噪声输入上叠加聪明会放大噪声，而非过滤它。

**当前版本的哲学**：
- 特雷门琴是显式模式，没有背景也没有 arming。
- 乐器举起时，可演奏频段内最近的回波就是手。
- 指向空旷空间静音，指向 40cm 墙演奏稳定音符——正确，因为你故意打开了模式。

**保留的两个鲁棒性**（关于传感器而非房间）：
1. **哪些 status 字节算数**：接受 7 个状态码而非 ST 的 2 个"valid"。
2. **保持（hold）**：250ms 抗闪烁。

**故意不做的**：地板过滤和重投影——都需要头部 FK 和 IMU，都是让音符因为玩家看不见的原因消失的方式。

### 4.2 Config（L43–L84）

| 字段 | 默认值 | 说明 |
|------|--------|------|
| near_m | 0.10 | 最近可演奏范围。低于 ~10cm 传感器盖玻璃串扰产生虚假短回波，是可信度下限 |
| far_m | 0.70 | 最远可演奏范围 |
| min_zones | 2 | 频段内多少个区域构成一只手。低值：远端的手只有少数区域，且传感器只给其中一些可用状态 |
| statuses | [4,5,6,9,10,12,13] | 相信距离的 ST 状态码 |
| hold | 250ms | 掉帧时保持最后一只手的时长 |

**statuses 每个码的含义**（L68–L73）：
- 5, 9：ST 的 "valid" 和 "valid, large pulse"
- 6：第一范围，未执行回绕检查
- 10：有效范围，前一个范围什么都没看到
- 12：被锐利边缘模糊的目标——手的边缘就是这样
- 4, 13：移动手超过 30cm 产生的一致性失败，音高不需要毫米级精度

`believes(status)` 方法：检查 status 是否在集合中。

### 4.3 Hand（L86–L102）

```rust
pub struct Hand {
    pub range_m: f64,    // 鲁棒距离（低百分位，不是单最近区域）
    pub closeness: f64,  // 在频段中的位置：far_m→0, near_m→1
    pub zones: usize,    // 覆盖区域数（区分手离开 vs 传感器闪烁）
    pub held: bool,      // 是否是保持的记忆而非当前测量
}
```

`closeness` 越近越高——音高和嘴都向这个方向移动。`zones` 报告因为它是说明掉帧是手离开还是传感器眨眼的数字。`held` 让读数可以显示桥接掉帧而非假装看到了东西。

### 4.4 Tracker（L104–L180）

```rust
pub struct Tracker {
    config: Config,
    last: Option<(Hand, Instant)>,
}
```

唯一有状态的部分是 hold：帧判定的其他一切只依赖该帧本身，这是乐器可预测的原因。

**track()（L129–L173）**：
1. 遍历 64 个区域，收集在频段内且状态可信的距离：
   - 负距离 → 收敛失败，跳过（不管 status 说什么）。
   - status 不可信 → 跳过。
   - 距离在 [near_m, far_m] 内 → 加入 in_band。
2. 如果 in_band.len() < min_zones：
   - 有 last 且未过期 → 返回 held 版本的 Hand。
   - 否则 → 清除 last，返回 None。
3. 否则（检测到手）：
   - 排序 in_band。
   - `range_m = in_band[len/5]`（20 百分位，不是最小值）。
   - `closeness = (far_m - range_m) / (far_m - near_m)`，clamp [0,1]。
   - 更新 last = Some((hand, now))。
   - 返回 hand（held=false）。

**为什么 20 百分位**（L161–L162）：单区域飞点比真实值短几厘米是常规，作为音高输入会是一声啁啾。低百分位比最小值鲁棒。

**reset()（L175–L179）**：清除保持的手——放下乐器时，重新拿起不会以之前的音符开头。

### 4.5 status_histogram（L182–L198）

```rust
pub fn status_histogram(status: &[u8]) -> Vec<(u8, usize)>
```

每个状态字节出现了多少区域，按出现次数降序（同次数按状态码升序）。纯诊断，但这是最重要的诊断："它在 30cm 后停止工作"和"status 4 覆盖了帧的 31 个区域"是同一句话，但只有后者告诉你该改什么。渲染到实时读数中，bench 会话永远不必猜传感器实际在说什么。

### 4.6 测试（L200–L356）

1. **the_statuses_a_real_hand_arrives_with_are_believed**（L219–L239）：pin 第一版的 bug——所有 7 个默认 status 码在 40cm 处都必须演奏。status 255（传感器说什么都没看到）和 status 1（没人相信的）必须不演奏。
2. **closeness_rises_as_the_hand_approaches**（L243–L260）：near_m→1.0, far_m→0.0，中间在 (0,1)。频段外无音符（不是 clamp 后的音符）。
3. **a_dropped_frame_holds_the_note_and_a_withdrawn_hand_ends_it**（L264–L292）：3 帧掉帧（66ms 间隔，共 198ms < 250ms hold）保持音符且 held=true；超过 hold 后静音；hold 不会复活。
4. **a_hand_that_keeps_being_seen_never_expires**（L296–L308）：持续看到的手 15Hz×60 秒（900 帧）永不过期——hold 从最后*看到*的手计时，不是从第一次掉帧。
5. **one_stray_zone_is_not_a_hand**（L311–L319）：1 个区域不是手（min_zones=2），2 个区域是。
6. **a_bad_frame_is_not_a_note**（L324–L337）：负距离、空数组、不匹配的 distance/status 长度都不是音符。
7. **the_histogram_leads_with_what_dominates_the_frame**（L342–L355）：直方图按出现次数排序，不越界读（长数组只取前 N_ZONES）。

---

## 五、控制流（ROUTE）

```
distance_mm &[i16] + status &[u8] + now: Instant
    → 遍历 64 区域:
        负距离或 status 不可信 → 跳过
        距离在 [near_m, far_m] → 加入 in_band
    → in_band.len() < min_zones?
        是 → hold 逻辑:
            last 存在且未过期 → Hand { held: true, ..last }
            否则 → None
        否 → 检测到手:
            排序 in_band
            range_m = in_band[len/5]  (20 百分位)
            closeness = (far_m - range_m) / span, clamp [0,1]
            last = Some((hand, now))
            → Hand { held: false }
```

---

## 六、效果主张与责任闭合卡（EFFECT）

### 6.1 主张一："掉帧不会切断音符，手真正离开会停止"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Tracker 的 hold 机制 | L146–L158, L264–L292 |
| 触发者 | in_band.len() < min_zones 的帧 | 传感器闪烁 |
| 当前装配/选择/开关 | hold=250ms，从最后看到的手计时 | L74, L150, L171 |
| 实际执行者 | Instant 比较 | 标准库 |
| 成功副作用与观察点 | 短掉帧返回 held Hand，长掉帧返回 None | 测试 L264–L292 |
| 失败是否返回且被检查 | 无失败路径；过期后 last=None | L153–L156 |
| 不能覆盖的对象 | ① 250ms 是调优值，不同传感器闪烁模式可能需要调整；② 手快速离开后 250ms 内仍有音符 | 设计权衡 |
| status | **confirmed**（3 帧掉帧保持，过期后停止，持续看到永不过期） | — |

### 6.2 主张二："超过 30cm 的手仍能演奏"

| 格 | 内容 | 证据定位 |
|----|------|----------|
| 对象/范围 | Config::statuses 包含 4 和 13 | L68–L73, L219–L239 |
| 触发者 | 超过 30cm 的移动手返回一致性失败状态码 | 传感器行为 |
| 当前装配/选择/开关 | statuses = [4,5,6,9,10,12,13] | L73 |
| 实际执行者 | Config::believes() 检查 | L81–L83 |
| 成功副作用与观察点 | status 4/13 的距离被接受 | 测试 L219–L239 |
| 失败是否返回且被检查 | 不在集合中的 status 被跳过 | L137 |
| 不能覆盖的对象 | ① 其他状态码（如 7/8/11）可能也携带有用距离但未被接受；② 不同传感器固件可能改变状态码含义 | 范围外 |
| status | **confirmed**（7 个码全部测试通过；是否需要更多码需 bench 验证） | — |

---

## 七、边界与反例（BREAK）

1. **不做地板过滤**：故意的。但如果头向下看且手在地板附近，地板回波可能被误判为手。特雷门琴模式下用户接受这个行为。
2. **不做重投影**：距离是沿波束的斜距，不是到躯干的水平距离。如果头偏转，"最近"可能不是物理上最近的物体。
3. **20 百分位用 len/5 索引**：整数除法。对于少量区域（如 2 个），len/5=0，就是最小值。对于大量区域，更接近中位数。
4. **hold 期间返回的 Hand 有 held=true 但 range_m 不变**：如果手在 hold 期间缓慢移动，返回的是旧位置。音频合成方需要检查 held 标志。
5. **min_zones=2 是硬编码默认**：一个非常近的大手可能覆盖很多区域，一个远端的小手可能只有 1-2 个。min_zones 太低会误报杂散，太高会漏检远端手。
6. **statuses 是 Vec<u8>，contains 是线性搜索**：64 区域 × 7 元素 = 最多 448 次比较/帧。15Hz 下可接受，但不是最优。
7. **不处理传感器温度漂移**：ToF 距离可能随温度变化，这里不做校准。
8. **track() 接受可能比网格短的数组**：`distance_mm.get(zone)` 和 `status.get(zone)` 用 Option 处理。但如果数组比网格长，多余部分被忽略。
9. **负距离总是跳过**：不管 status 说什么。如果传感器在负距离下携带其他信息，这里丢失了。
10. **reset() 只清除 last**：不清除 config。如果需要重置配置，必须构造新的 Tracker。
11. **closeness 的 span 用 max(1e-6)**：防止 near_m==far_m 时除零。但这种配置本身是错误的。
12. **status_histogram 只取前 N_ZONES 个元素**：如果 wire 数组更长，多余部分被忽略。

---

## 八、结论（按状态分级）

### confirmed
- C1：Tracker 从 ToF 8×8 帧中检测手，输出 range_m/closeness/zones/held。
- C2：第一版的复杂方案（背景/平面拟合/漂移）在鸭子上失败，因为传感器输入不稳定。
- C3：当前版本无背景/无 arming，频段内最近回波就是手。
- C4：接受 7 个 status 码（含一致性失败 4/13），不只是 ST 的 5/9。
- C5：hold=250ms 抗闪烁，从最后看到的手计时。
- C6：range_m 用 20 百分位（in_band[len/5]），不是最小值。
- C7：closeness 越近越高，far_m→0, near_m→1。
- C8：故意不做地板过滤和重投影——演奏原始光束的乐器可预测。
- C9：status_histogram 是关键诊断工具，按出现次数排序。
- C10：测试覆盖：所有 status 码、closeness 端点、hold 机制、持续手不过期、杂散区域、坏帧、直方图。

### inferred
- I1：Tracker 被特雷门琴音频合成代码使用。
- I2：distance_mm/status 数组来自 tofd 的 wire 格式。
- I3：特雷门琴是机器人的一个交互模式（"instrument is up"）。
- I4：tof.rs 的 beams() 可能被第一版的平面拟合使用，当前版本未使用。

### unknown
- U1：特雷门琴音频合成的具体实现（音高映射、音量包络）。
- U2：第一版实现的 git 历史和具体失败数据。
- U3：status 码 4/13 在不同距离/光照下的出现频率。
- U4：hold=250ms 的调优过程和是否可配置。

---

## 附录 A　资料来源

1. 原文件：hand.rs（本地附件，357 行，全文已读）。
2. 同 crate 文件：tof.rs（ROWS/COLS）、lib.rs。
3. 外部参考：ST VL53L5CX status 码定义。
4. 分析方法：doubao-coding-analyze-codebase Skill。

---

*报告生成时间：2026-09-10 | 分析方法：SCOPE→ROUTE→EFFECT→BREAK→SHIP*
#（注：内容由AI生成）
