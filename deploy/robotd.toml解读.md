# `/etc/robot/robotd.toml` 配置解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件路径 | `/etc/robot/robotd.toml` |
| 文件类型 | TOML 配置文件（每机器人参数） |
| 读取时机 | 启动时读取一次，**不支持热加载**；修改任何配置需 `systemctl restart robotd`（live reload 已推迟，见 docs/design/robotd-design.md §4.2） |
| 与发布物关系 | 位于 `/etc/robot/` 而非 `/opt/robot/daemon/releases/<ver>/`——这是**每机器人配置**而非出厂默认，必须在**更新和回滚后都存活**（docs/design/architecture.md §3）；安装器从不覆盖它 |

**关键特性**：

1. **文件可整体删除**：未配置的板卡用同样的内置默认值启动，而不是拒绝启动——一个"起不来的守护进程"极难远程诊断，而"用默认值运行"很容易。
2. **注释即文档**：文件中所有被注释掉的值显示的是**内置默认值**。`install.sh` 只复制一次、永不覆盖，因此一个被取消注释的值会在板卡上**永久冻结**，而发布物后续更新默认值也不会生效——这正是"机群全部停在 kP 120、而发布默认已是 160"的教训来源。
3. **与 `updater.toml` 的联动**：本文件的 `[update_gate]` 就是更新系统的"健康判决"来源——updater 的自动回滚只有建立在真实测量上才有意义。

---

## 二、`[bus]` 总线

```toml
port = "/dev/ttyS2"
```

- 15 个舵机与 `imu_to_dxl` 板共享的串口。
- `/dev/ttyS2` 是 Radxa Zero 3W 的固定接线；不同接线的板卡可用 `robotd --port` 在**单次调用**时覆盖——这是开发逃生舱口，而非此处该改的东西。

---

## 三、`[control]` 控制环

| 配置项 | 值 | 说明 |
|---|---|---|
| `hz` | `50` | 控制环频率（Hz），**继承自原型**（在 Raspberry Pi Zero 2W 上选定），从未在 Radxa 上重新推导——信任前请用 `bench_dynamixel_bus` 实测实际频率、抖动、总线利用率和错误计数 |
| `cmd_alpha`（注释默认） | `0.2` | 速度指令的每 tick EMA 平滑：`cmd += alpha * (target - cmd)`，把摇杆突变成步态可跟随的斜坡；`1.0` 为直通 |
| `head_alpha`（注释默认） | `0.2` | 同上，作用于头部目标与身体姿态 |

---

## 四、`[update_gate]` 更新健康门（与 updater 联动的核心）

> "This is the whole point of the daemon in its current form" —— 这是 robotd 当前形态存在的全部意义。

**设计定位**：
- 它决定 `healthy` 判决，进而决定更新系统是否**回滚**。
- 刻意命名为 `update_gate` 而非 `health`：`robot.health` 报告的内容（电池、电机温度、环路/总线计数器）**没有一项可以下判决**——"发布绝不能因为落地板卡的状态而被回滚"。所以这里只配置与"这次更新是否把机器人搞坏了"相关的三个阈值。

| 配置项（注释默认） | 值 | 含义 |
|---|---|---|
| `min_achieved_hz` | `45.0` | 实际速率下限，低于此即 unhealthy。45 = 默认 50Hz 的 90%：松到不会因一个慢 tick 误判，紧到"每 10 个周期掉 1 个"不会被叫健康。**应随 `control.hz` 同步上调** |
| `stall_periods` | `25` | 连续多少个周期完全没有 tick 才算 **WEDGED（卡死）**。25 周期 = 50Hz 下的 500ms：沉默这么久的环路才是真死了。它只检测**死掉的环路**而非**慢的环路**（慢归 `min_achieved_hz` 管），两者必须分开。历史教训：最初为 3（60ms），忙碌板卡的普通调度抖动就会触发——一个因抖动而误报的健康检查会**回滚掉好的发布** |
| `max_consecutive_errors` | `10` | 容忍的连续失败总线读取次数。一次丢包在串口总线上很常见；连续一串说明**总线没了**——读不到自己关节的机器人，无论环路转多快都不健康 |

**与 updater.toml 健康门的对应**：updater 侧 `health.probe = "socket"` + `timeout = "30s"` 只是探测手段和等待窗口，**真正的判决标准来自本文件**；updater 侧"只回滚 unhealthy、不回滚 degraded"的语义，正由本文件"慢环路与死环路分开判定"来支撑。

---

## 五、`[policy]` 策略

### 1. 开关与模式

```toml
enabled = true
mode    = "walk"
```

- `enabled = false` 是**合法配置而非降级配置**：环路照跑、保持初始姿态、依然健康。它是 bench 上反复做 install/rollback/断电测试时最安全的状态——故意破坏的发布落地时不会摔坏任何东西。
- 它与"想要策略但加载失败"**刻意区分**：后者报 unhealthy，updater 据此回滚发布。`robotd --no-policy` 可从命令行设置。
- `mode`：`"walk"`（腿，默认）或 `"roller"`（轮子）。模式会改变加载哪些策略 **和** 下面的调参默认值——把机器人切到 roller 就是这一行加 `systemctl restart robotd`。roller 预设：`roller.onnx` 作为运动策略、地面拾取触发下蹲（3.0s / 0.8）、`action_scale 0.8`，其余同 walk；只有站立网络不加载（轮子上跳过站立转换）。

### 2. 策略文件

- 未设置时使用该模式在 `/opt/robot/policies/current` 的默认——该目录**故意在发布物之外**：步态重训不应需要发布守护进程版本，守护进程修复也不应重发 6MB 未变的权重。发布物仍携带文件并播种该目录（`scripts/seed-policies.sh`），直到有人发布真正的策略包。
- 不用手工编辑：`robotctl policy load <slot> <file>` 热切换网络，`robotctl policy reset [slot]` 移除。
- **统一形状约束**：所有策略必须是 **61 输入 / 14 输出**（`obs[1,61] -> actions[1,14]`），robotd 在**加载时**检查而非运行中途发现。microduck_runtime 还带 51-D 旧家族（3 值命令），robotd 加载时直接拒绝并报 "observation width is 51, expected 61"。
- walk 模式默认策略清单（均在 `/opt/robot/policies/current/`）：

| Slot | 文件 | 用途 |
|---|---|---|
| walk | `alpha_walking.onnx` | velstand 步态 |
| stand | `alpha_stand.onnx` | 站立 + 身体姿态 |
| sitstand | `alpha_sitstand.onnx` | 坐↔站，姿态标志 |
| ground_pick | `alpha_ground_pick.onnx` | A 键拾取 |
| kick_left / kick_right | `ball_kick_left/right.onnx` | 左右踢球 |
| roulade | `roulade.onnx` | X 键前滚翻 |

- ONNX Runtime 是 **dlopen 而非链接**：缺库是启动失败而非构建失败，robotd 通过 `robot.health` 报告并保持姿态；`ORT_DYLIB_PATH` 可覆盖查找位置。

### 3. 调参默认值（均注释，按模式解析）

| 配置项（注释默认） | walk | roller | 说明 |
|---|---|---|---|
| `action_scale` | `0.9` | `0.8` | 策略原始输出到关节偏移的缩放 |
| `standing_action_scale` | `1.0` | — | 站立策略按整体应用 |
| `standing_gain_ratio` | `0.8` | — | 站立时以 `gain` 的该比例运行（更柔） |
| `gain` | `200` | — | 运行时的位置 P 增益 |
| `head_lowpass` / `legs_lowpass` | `0.5` / `0.7` | 同左 | 头部/腿部关节目标一阶低通混合因子（0,1]，1.0 直通。**必须与训练值一致**（mjlab ACTION_LOW_PASS_*），否则迁移退化 |
| `ground_pick_period` | `4.0` | `5.0` | 拾取周期（秒），未设置时取自清单 |
| `ground_pick_action_scale` | `1.0` | `0.8` | 拾取期间动作缩放 |
| `ground_pick_gain_ratio` | `1.0` | — | 拾取期间增益倍率 |
| `kick_duration` | `0.5` | — | 踢球窗口停留在踢球网络上的秒数 |
| `roulade_duration` / `action_scale` / `gain_ratio` | `1.0` / `1.0` / `1.0` | — | 一个前滚翻的时长；持 X 链式翻滚，空中的滚动总是完成 |
| `voltage_adapt` | `false` | — | 按电池电压缩放动作（`action_scale * (nominal/measured EMA，钳位 6.0~9.5V)`），补偿舵机 kP 随供电变化；默认关，同原型 |
| `nominal_voltage` | `7.4` | — | 电压自适应标称电压 |

---

## 六、`[safety]` 安全

### 1. 摔倒判定（报告，不拒绝）

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `fall_gravity_z` | `-0.5` | 躯干坐标系投影重力：直立约 -1.0，侧躺近 0 |
| `fall_debounce_ms` | `200` | 去抖，防止坚实脚步的冲击被读成摔倒 |

- **它只是 REPORT**：显示在 `robotctl monitor` 与状态流中，不拒绝任何操作——摔倒的机器人仍可 enable、init、被驱动、接收技能，因为"倒下了"恰恰是有人需要做这些事的时刻。daemon 对摔倒*实际做*的事是下面的 `limp_fall`，且它用自己的探测器。

### 2. 看门与软着陆

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `deadman_ms` | `500` | 失去速度意图多久后**停止**（stop 而非 limp）：失去客户端连接时双足应保持站立——站立是它的安全态。失稳是另一事件，由 `limp_fall` 应答 |
| `gain_limp` | `50` | limp-fall 让步时的增益——低到"让路"而非"和地板较劲" |

### 3. `limp_fall` 软着陆机制（默认开启）

**为什么需要它**：站立策略是"好起身、坏摔倒"——从静止（无论面朝下还是面朝上）能干净站起来；动态摔倒中它会以步行增益反复尝试、对地板硬扛，电机为每次尝试买单。

**机制流程**：
1. **接管**：一旦检测到正在倒下（落地前），增益降到 `gain_limp`，关节被推到哪里跟到哪里——机器人**以瘫软状态落地**。
2. **等待静止**：停止移动后，在 `limp_fall_pose_ms` 内斜坡回站立姿态。
3. **交还**：交还给站立策略时它面对的是"已知姿态的静止机器人"——这正是它擅长的情形。交还只是交还：无人驾驶时，指令幅值选择站立网络，那就是起身。

**触发判据**（这是整个特性的核心，两种错误方式都是真实的：太早→机器人从本可走开的倾斜中瘫倒，每次误报都是机器人*自己造成*的摔倒，比僵硬落地更糟；太晚→照样僵硬落地，模式白买）：

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `limp_fall_tilt_z` | `-0.90` | 已倾斜约 26°——普通行走达不到，这排除脚步冲击和推搡 |
| `limp_fall_predict_z` | `-0.5` | 外推预测值落点 |
| `limp_fall_lookahead_ms` | `300` | 预测前瞻：不用位置判断（重力到达阈值时机器人已在地板上），而是用投影重力的**变化率**（恰为陀螺仪同 12 字节 IMU 块的 `-(w×g)`）外推 |
| `limp_fall_debounce_ms` | `60` | 50Hz 下 3 个 tick：长于脚步冲击脉冲，短到把摔倒的大部分留给瘫软过程 |

**落地与善后**：

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `limp_fall_still_rate` | `1.0` | 静止判定速率 |
| `limp_fall_still_ms` | `200` | 静止判定的持续时间——从**陀螺仪**读取而非计时，因为绊倒和从桌上摔落耗时差异很大 |
| `limp_fall_max_ms` | `1500` | 无论陀螺仪如何都结束瘫软——被人捧在手里的机器人不会永远瘫着 |
| `limp_fall_pose_ms` | `600` | 姿态斜坡时长：0.6s 是在真机上敲定的（完整 1s 在起身前大半是死时间，因为关节空载移动而非抬升机器人；0.3s 也有效，此值留有余量） |
| `limp_fall_pose_gain` | `160` | 姿态斜坡增益，非 limp 增益——关节要真正在地面上移动 |

- **默认开启的原因**：整个意义在于"机群软着陆"，而"每块板都要单独 opt-in 的模式"等于"大多数板都没有的模式"。需要旧行为的机器人可设 false。

### 4. 低电量关机

```toml
# battery_empty_shutdown = true
```

- 电池 EMA 到达空电下限（**6.6V**，即电量百分比映射到 0% 的同一值）时**优雅坐下并关机**。
- EMA 移动约 10 秒，负载骤降不会误触发；到 6.6V 意味着电池确实耗尽。

---

## 七、`[audio]` 音频

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `enabled` | `true` | 总开关。全部是"可选设备"：无 codec 的板卡行走一致、保持安静，**没有一项达到健康判决** |
| `device` | `plughw:aic3104` | ALSA 播放设备（TLV320AIC3104 codec，由 setup-board.sh 启用）；麦克风从 `<device>,0` 采集 |
| `bank` | `/var/lib/robot/sounds` | 语音库位置（发布物 postinstall 的 `sounds ensure-bank` 渲染） |
| `greet` | `true` | 控制环启动时叫一声——无头板卡上这是可听的"robotd 已在运行" |
| `pet_detect` | `true` | 抚摸检测（opt-in，未设置为关——常开"咕咕"会腻）。模型随发布物（`models/pet_detect.onnx`）；阈值是分类器的滞回（进入高于 enter、离开低于 exit）。麦克风在 [audio] 开启时持续读取，但分类器只在"从环境声里突出来"的音频上运行 |
| `pet_enter_threshold` / `pet_exit_threshold` | `0.95` / `0.85` | 抚摸检测滞回阈值 |

---

## 八、`[theremin]` 特雷门琴（ToF 距离→音符）

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `enabled` | `true` | 只决定"能否玩"——已知 ToF 有问题的鸭子可以直接拒绝。**需要 [audio] 开启**（没声音的特雷门琴只是无意义张嘴） |
| `socket` | `/run/tofd/tof.sock` | tofd 的深度流 |
| `near_m` / `far_m` | `0.10` / `0.70` | 可演奏带宽（米）。近于 near_m 是传感器盖板玻璃串扰虚构的短返回；远于 far_m 是噪声返回，会变成低音 |
| `min_zones` | `2` | 构成一只手的**最少区域数**：带宽远端一只手只占少数 zone，且只有部分 zone 带可用状态 |
| `statuses` | `[4, 5, 6, 9, 10, 12, 13]` | **决定乐器音域的关键字段**。ST 只把 5/9 记为 "range valid"，但相信它们会在约 30cm 处丢失手（移动的手返回 4/13——一致性失败、sigma 过高——但距离值对音高完全可用）；6/10/12 是实践中可用的代码（12 是"被锋利边缘模糊"，即手的边缘）；255 是"看了但没找到"，永不入列。`robotctl theremin` 最后一列是每代码计数（带 * 标记本列表相信的），即调参手段 |
| `hold_ms` | `250` | 传感器丢帧时音符保持时长（ms）：防止 zone 在可用/不可用间闪烁把音符切碎成砂砾；太长则收手后余音不断 |

---

## 九、`[detect]` 鸭子检测

- **读取方是 `mediad` 而非 `robotd`**：帧在 mediad 的流水线上，感知应靠近传感器——但开关放在这里，因为 `robotctl configure` 编辑此文件，机器人只有一处设置。
- **默认关闭**：模型随发布物（NPU 用 `duck_detect.rknn`、CPU 用 `duck_detect.onnx`），没被要求找鸭子的机器人不该为它每半秒付出约 60ms 的工作。

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `enabled` | `true` | 开关（默认关，注释显示默认值） |
| `model` | 未设置 | 未设置时用发布物自带模型，**NPU 优先**；板卡 NPU 在设备树中关闭时回退 CPU（Armbian 上 Radxa Zero 3 正是如此）。`setup-npu.sh` 可启用，需重启 |
| `hz` | `2.0` | 每秒检测次数。**2 是热限制而非偏好**：全速在 Radxa Zero 3 上可达 95°C，CPU 节流到 408MHz——"为看清而走不好"的机器人 |
| `threshold` | `0.35` | 该模型自有尺度上的置信阈值。出厂模型量化为 INT8，分数**不是概率**：真实检测约读 1.3，其余什么都不读（输出张量与框坐标共享同一量化尺度）。所以这是 yes/no 阈值而非旋钮 |

---

## 十、`[chorale]` 合唱

```toml
# accept = false
```

- **默认关闭，且这就是整个章节的意义**：合唱不只是声音——它动嘴、动头。一只因为别的机器人走进房间就开始做动作的机器人，是在别人的客厅里做没人要求的动作；咖啡馆里两人的鸭子不该擅自配对。
- **关闭意味着"隐身"而非"礼貌拒绝"**：未 opt-in 的鸭子完全不向空气中发射任何东西。
- 需要 [audio] 开启（没嗓子无歌可唱）。
- 机制：无共享时钟的蓝牙四声部合唱，指挥鸭的节拍计数器是时基，声部按每只鸭自己的音域推算；`robotctl chorale` 启动寻找同伴。

---

## 十一、`[media]` 媒体（mediad 读取）

> 由 **mediad** 启动时读取，改动需 `systemctl restart mediad`（非 robotd）。这些曾是 mediad.service ExecStart 行的 flag、由发布安装器重写——唯一的改法曾是 systemd drop-in，而没人会为"为什么视频发糊"去写 drop-in。

| 配置项（注释默认） | 值 | 说明 |
|---|---|---|
| `camera` | `true` | 流式输出头部相机；关则输出测试图案（无相机的板卡所需）——流水线仍启动，所以 WebRTC **控制**通道仍存在。与视频轨捆绑：流水线起不来则两者皆失 |
| `quality` | `"720p30"` | 帧尺寸+帧率单键值（1080p30 / 720p30 / 720p15 / 360p30）。**不用宽/高/fps 三键**：采集路径产不出的档位 = 起不来的 mediad；所有档位都是 16:9，"更小"永不等于"被裁剪"。720p30 是 mediad 所有测量的基准档（ISP 主路径 29.3fps，采集格式与缓冲深度花了三个 bench 会话）；传感器钉在 1920x1080@30、ISP 向下缩放，所以 1080p30 完全不做缩放——未测量的是 2.25 倍像素下采集与编码能否维持 30fps；撑不住的档位**减速运行而非失败** |
| `bitrate` | `2000000` | 起始视频码率（bps）。未设置时随档位（1080p30→4Mb/s、720p30→2Mb/s、720p15→1Mb/s、360p30→800kb/s）。webrtcsink 拥塞控制从此处起跳，所以是**起点而非天花板**。单位是 bit：低于 100000 被拒绝（有人想写 kbps） |
| `congestion_control` | `"gcc"` | 是否随链路自适应及方式：`gcc` / `homegrown` / `disabled`。**这是 CPU 设置也是网络设置**：估计器是 mediad 最大的单消费者（单 peer 时 `rtpgccbwe` 占 7.6% 核 vs v4l2src 的 0.3%——它按包工作，采集按 DMABuf 句柄工作）。`disabled` 删掉该线程，代价是自适应能力（这正是把原始视频而非预编码 H.264 交给 webrtcsink 的全部原因）；`disabled` 也让 `bitrate` 名副其实（没人移动它，它就是速率）。`gcc` 本就是 webrtcsink 默认，此行不改变任何行为，放在这里是为了**可被发现** |

---

## 十二、与 `updater.toml` 的联动关系

| 维度 | updater.toml | robotd.toml |
|---|---|---|
| **健康判决** | `health.probe = "socket"`、`timeout = "30s"`——探测手段与等待窗口 | `[update_gate]` 三个阈值——**判决标准本身** |
| **回滚语义** | 只回滚 unhealthy、不回滚 degraded | 死环路（stall_periods）与慢环路（min_achieved_hz）分开判定，支撑上述语义 |
| **重启范围** | `on_apply.units = ["robotd", "configd"]`——robotd 是更新后重启的单元 | 配置变更需 `systemctl restart robotd`；restart 由 updater 触发 |
| **配置持久性** | 更新不触碰 `/etc`（allow_dev_keys 等只可本地覆盖） | 本文件在 `/etc/robot/`，安装器从不覆盖——更新与回滚后都存活 |
| **失败可见性** | models 组件宁缺毋滥，避免"永报失败" | 策略加载失败报 unhealthy 触发回滚；而"策略刻意关闭"是合法健康状态 |

**一句话总结**：`updater.toml` 决定"何时、以什么权限、安装什么、装完是否回滚"，`robotd.toml` 决定"机器人当前是否真的健康"——前者依赖后者给出的判决，后者通过前者的重启与回滚机制获得可信的发布通道。两份文件一个在信任与发布侧、一个在运行与健康侧，共同构成系统的更新闭环。

---

## 十三、核心设计思想总结

| 设计原则 | 落地方式 |
|---|---|
| **每机器人配置与发布物分离** | 配置在 `/etc`、策略在 `/opt/robot/policies/current`、发布物在 `/opt/robot/daemon/releases/`，更新/回滚互不污染 |
| **健康判决必须真实可测** | update_gate 只含三个阈值，其余指标（电池/温度/计数器）一律不得参与判决 |
| **失败可见性** | 缺配置用默认值启动而非拒绝启动；坏策略加载时报 unhealthy；慢环路与死环路严格分开 |
| **安全状态优先** | 失去连接→站立（deadman stop）；摔倒→软着陆（limp_fall）；低电量→优雅关机 |
| **功耗与热约束显式化** | 检测频率 2Hz 是热限制；码率自适应是 CPU 设置也是网络设置 |
| **默认安全/默认不打扰** | 合唱默认关闭且隐身；抚摸检测 opt-in；鸭子检测默认关 |
| **调参与训练一致** | 低通系数必须匹配训练值，否则迁移退化；策略形状在加载时强校验 |

---

## 十四、配置速查表

| 模块 | 配置项 | 值 | 一句话含义 |
|---|---|---|---|
| bus | `port` | `/dev/ttyS2` | 舵机与 IMU 板共享串口（Radxa Zero 3W） |
| control | `hz` | `50` | 控制环频率（继承自原型，待实测复核） |
| control | `cmd_alpha` / `head_alpha` | `0.2` / `0.2`（默认） | 指令/头部 EMA 平滑 |
| update_gate | `min_achieved_hz` | `45.0`（默认） | 实际速率下限，低于即 unhealthy |
| update_gate | `stall_periods` | `25`（默认） | 500ms 无 tick 判定环路卡死 |
| update_gate | `max_consecutive_errors` | `10`（默认） | 连续总线读取失败上限 |
| policy | `enabled` | `true` | 加载策略（false 为合法 bench 配置） |
| policy | `mode` | `"walk"` | walk / roller 模式 |
| policy | 默认策略 | 61 输入/14 输出 | 加载时强校验形状 |
| policy | `action_scale` | `0.9` / `0.8` | 输出缩放（walk/roller） |
| policy | `gain` | `200`（默认） | 位置 P 增益 |
| policy | `head_lowpass` / `legs_lowpass` | `0.5` / `0.7`（默认） | 低通系数（须匹配训练） |
| policy | `voltage_adapt` | `false`（默认） | 电池电压自适应（默认关） |
| safety | `fall_gravity_z` / `fall_debounce_ms` | `-0.5` / `200`（默认） | 摔倒判定（仅报告） |
| safety | `deadman_ms` | `500`（默认） | 失去意图后停止（不瘫软） |
| safety | `limp_fall` | `true`（默认） | 软着陆接管（默认开） |
| safety | `limp_fall_*` | 多参数 | 触发/瘫软/起身全套参数 |
| safety | `battery_empty_shutdown` | `true`（默认） | 6.6V 优雅关机 |
| audio | `enabled` / `device` / `bank` | `true` / `plughw:aic3104` / `/var/lib/robot/sounds` | 语音与拾音 |
| audio | `pet_detect` | `true`（opt-in） | 抚摸检测（默认关） |
| theremin | `enabled` / `statuses` / `hold_ms` | `true` / `[4,5,6,9,10,12,13]` / `250` | ToF 特雷门琴（关键字段 statuses） |
| detect | `enabled` / `hz` / `threshold` | `true`（默认关）/ `2.0` / `0.35` | 鸭子检测（默认关，2Hz 为热限制） |
| chorale | `accept` | `false`（默认） | 合唱 opt-in（默认关且隐身） |
| media | `camera` / `quality` / `bitrate` / `congestion_control` | `true` / `"720p30"` / `2000000` / `"gcc"` | 视频流参数（mediad 读取） |
#（注：内容由AI生成）
