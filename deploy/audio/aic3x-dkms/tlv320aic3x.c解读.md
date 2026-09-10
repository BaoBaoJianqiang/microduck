# `tlv320aic3x.c` 核心驱动解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Linux 内核模块源码（C，约 1888 行） |
| 职责 | TLV320AIC3x 系列 codec 的 **ASoC 核心驱动**：寄存器、DAPM 音频路径、混音器控件、PLL 配置、电源管理、型号差异 |
| 作者 | Vladimir Barinov, MontaVista Software（2007），基于 `wm8753.c`（Liam Girdwood） |
| 许可证 | GPL-2.0-only |
| 对应模块 | `snd-soc-tlv320aic3x.ko` |
| 支持型号 | aic31 / aic32 / aic33（全功能）、aic3007（带 Class-D）、aic3104（本项目）、aic3106 |
| 导出符号 | `aic3x_probe`、`aic3x_remove`（`EXPORT_SYMBOL`）、`aic3x_regmap`（`EXPORT_SYMBOL_GPL`） |

**型号兼容性说明**（文件头注释）：驱动完整实现 aic33 功能；aic32/aic3007 无 MONO_LOUT；aic31 额外缺少 LINE1/2 和 MIC3 输入。机器层应通过 `snd_soc_dapm_disable_pin()` 禁用不支持的引脚。

---

## 二、私有数据结构与电源

### 1. 四路电源

```c
#define AIC3X_NUM_SUPPLIES 4
static const char *aic3x_supply_names[] = {
    "IOVDD",   // I/O 电压
    "DVDD",    // 数字核心电压
    "AVDD",    // 模拟 DAC 电压
    "DRVDD",   // ADC 模拟与输出驱动电压
};
```

- 四路独立电源通过 `devm_regulator_bulk_get()` 批量请求，`regulator_bulk_enable/disable()` 批量开关。
- 每路电源注册了 `regulator notifier`（`aic3x_regulator_event`），响应电源状态变化。

### 2. `aic3x_priv` 私有数据

| 字段 | 类型 | 含义 |
|---|---|---|
| `component` | `snd_soc_component *` | ASoC component 指针 |
| `regmap` | `regmap *` | 寄存器映射（带缓存） |
| `supplies[4]` | `regulator_bulk_data` | 四路电源 |
| `disable_nb[4]` | `aic3x_disable_nb` | 四路电源的 notifier block |
| `setup` | `aic3x_setup_data *` | GPIO 功能配置（来自平台数据/设备树） |
| `sysclk` | `unsigned int` | 系统时钟频率（PLL 计算的基准） |
| `dai_fmt` | `unsigned int` | DAI 格式（I2S/左对齐/右对齐/DSP + 主从 + 极性） |
| `tdm_delay` / `slot_width` | `unsigned int` | TDM 槽延迟与槽宽 |
| `master` | `int` | 是否位时钟/字时钟主模式 |
| `gpio_reset` | `int` | 硬件复位 GPIO（-1 表示无） |
| `power` | `int` | 当前电源状态 |
| `model` | `u16` | 型号 ID（AIC3X_MODEL_3104 等） |
| `micbias_vg` | `enum` | MICBIAS 电压（2.0V/2.5V/AVDD/OFF） |
| `ocmv` | `u8` | 输出共模电压档位 |
| `list` | `list_head` | 全局 reset_list 节点（用于共享 reset GPIO 检测） |

---

## 三、regmap 与寄存器缓存

### 1. 默认值表

`aic3x_reg[]` 定义了全部 **110 个寄存器**（地址 0–109）的上电默认值，与数据手册一致。例如：
- 寄存器 3（PLL_PROGA）默认 `0x10`
- 寄存器 15/16（ADC PGA）默认 `0x80`
- 寄存器 43/44（DAC 音量）默认 `0x80`
- 寄存器 102（CLKGEN）默认 `0x02`

### 2. volatile 判定

```c
static bool aic3x_volatile_reg(struct device *dev, unsigned int reg) {
    switch (reg) {
    case AIC3X_RESET: return true;   // 只有软件复位寄存器是 volatile
    default: return false;
    }
}
```

- 只有复位寄存器（写触发复位，读无意义）不缓存，其余全部缓存。
- 缓存类型 `REGCACHE_RBTREE`（红黑树，适合稀疏寄存器空间）。

### 3. `aic3x_regmap` 导出

```c
const struct regmap_config aic3x_regmap = {
    .max_register = DAC_ICC_ADJ,      // 109
    .reg_defaults = aic3x_reg,
    .num_reg_defaults = 110,
    .volatile_reg = aic3x_volatile_reg,
    .cache_type = REGCACHE_RBTREE,
};
EXPORT_SYMBOL_GPL(aic3x_regmap);
```

- 总线层（I2C）复制此配置并设置 `reg_bits=8 / val_bits=8`，避免重复定义缓存策略。

---

## 四、自定义 DAPM 与 MICBIAS

### 1. 输入混音器的特殊 put 函数

```c
static int snd_soc_dapm_put_volsw_aic3x(...)
```

- **问题**：AIC3x 的所有输入路由使用 `0xf` 位域——非零值表示连接，`0xf` 表示断开（与常规"1=连接"相反）。
- **解决**：自定义 put 函数把用户写入的非零值统一映射为 `0xf`（连接），零映射为断开，并正确更新 DAPM 电源状态。
- 通过宏 `SOC_DAPM_SINGLE_AIC3X` 应用到所有输入混音器控件。

### 2. MICBIAS 事件

```c
static int mic_bias_event(struct snd_soc_dapm_widget *w, ...)
```

- **POST_PMU**（上电后）：把 MICBIAS 电压设为用户配置值（2.0V/2.5V/AVDD）。
- **PRE_PMD**（掉电前）：MICBIAS 电压清零。
- MICBIAS 的电源位与电压位共享同一寄存器字段，所以必须在通电后再写电压、断电前先清电压——这是该事件函数存在的原因。

---

## 五、控件体系

### 1. 枚举控件（Enum）

| 控件 | 寄存器 | 选项 |
|---|---|---|
| 左/右 DAC mux | `DAC_LINE_MUX` | DAC_L1/L3/L2、DAC_R1/R3/R2 |
| 左/右 HPCOM mux | `HPLCOM_CFG`/`HPRCOM_CFG` | 差分/恒定 VCM/单端/外部反馈 |
| Line1/Line2 输入模式 | 各路由寄存器 | 单端 / 差分 |
| ADC HPF | `CODEC_DFILT_CTRL` | 关闭 / 0.0045Fs / 0.0125Fs / 0.025Fs |
| AGC 电平/攻击/衰减 | `LAGC_CTRL_A`/`RAGC_CTRL_A` | -5.5~-24dB、8~20ms、100~500ms |
| 上电时间/斜坡步长 | `HPOUT_POP_REDUCTION` | 0us~4s、0~4ms |

### 2. TLV dB 刻度

| 刻度 | 范围 | 步长 | 备注 |
|---|---|---|---|
| `dac_tlv` | -63.5 ~ 0 dB | 0.5 dB | DAC 数字音量 |
| `adc_tlv` | 0 ~ 59.5 dB | 0.5 dB | ADC PGA 增益 |
| `output_stage_tlv` | -59 ~ 0 dB | 0.5 dB（低端非线性） | 输出级混音器音量；寄存器值 100 显示 -50dB（实际 -50.3），117 显示 -58.5dB（实际 -78.3） |
| `out_tlv` | 0 ~ 9 dB | 1 dB | 输出引脚音量（Line/HP/HPCOM Playback Volume） |
| `classd_amp_tlv` | 0 ~ ... | 6 dB | 仅 3007 的 Class-D 功放 |

### 3. kcontrol 数组

驱动定义了多组控件，按功能和型号分组：

| 数组 | 内容 |
|---|---|
| `aic3x_snd_controls[]` | 通用控件：PCM 音量、各输出端 PGA/DAC 旁路音量、输出引脚 Volume+Switch、AGC Switch |
| `aic3x_extra_snd_controls[]` | 额外控件（DAC mux、HPCOM mux、ADC HPF、AGC 参数、上电时间等） |
| `aic3x_mono_controls[]` | 单声道输出控件（3X/33/3106） |
| `*_line_mixer_controls[]` / `*_hp_mixer_controls[]` / `*_hpcom_mixer_controls[]` / `*_pga_mixer_controls[]` | 各输出端的输入混音器控件（左右声道分开） |
| `aic3104_left/right_pga_mixer_controls[]` | **3104 专用** PGA 混音器（3104 的 PGA 输入路由与其他型号不同） |

**AGC 警告**（代码注释）：启用 AGC 需谨慎——它可能在 ADC 开启时把 PGA 调到最大值且永不回落。

---

## 六、DAPM 音频路径

### 1. Widget 分类

| 类别 | Widget |
|---|---|
| 输入 | `MIC3L`、`MIC3R`、`LINE1L`、`LINE1R`、`LINE2L`、`LINE2R` |
| 放大器 | `Left PGA`、`Right PGA` |
| ADC | `Left ADC`、`Right ADC` |
| DAC | `Left DAC`、`Right DAC` |
| 输出 | `HPLOUT`、`HPROUT`、`HPLCOM`、`HPRCOM`、`LLOPM`、`RLOPM`、`MONOLOPM` |
| 电源 | `LLOPM Pulldown` 等输出端下拉 |
| 其他 | `Mic Bias`（带事件）、`AIC3X Headset Detect` |

### 2. 型号专用 Widget

| 数组 | 适用型号 |
|---|---|
| `aic3x_dapm_widgets[]` | 所有型号的基础 widget |
| `aic3x_extra_dapm_widgets[]` | 3X/33/3106 的额外 widget |
| `aic3104_extra_dapm_widgets[]` | **3104 专用**（PGA 输入路由差异） |
| `aic3x_dapm_mono_widgets[]` | 单声道输出（3X/33/3106） |
| `aic3007_dapm_widgets[]` | 3007 的 Class-D 输出 |

### 3. 路由表（intercon[]）

主路由表定义了完整的音频信号流：

```
输入源 → PGA Mixer → PGA → ADC → AIF Capture
AIF Playback → DAC → 输出 Mixer → 输出端（HP/Line/HPCOM/MONO）
```

- `intercon_extra[]`：3X/33/3106 的额外路由。
- `intercon_extra_3104[]`：**3104 专用路由**（MIC3 到 PGA 的连接方式不同）。
- `intercon_mono[]` / `intercon_3007[]`：单声道 / Class-D 路由。

### 4. `aic3x_add_widgets()`

按型号动态添加 widget 和路由：
- 所有型号：基础 widgets + 基础路由
- 3X/33/3106：extra widgets + extra 路由 + mono widgets + mono 路由
- 3104：3104 extra widgets + 3104 extra 路由
- 3007：3007 widgets + 3007 路由

---

## 七、核心操作函数

### 1. `aic3x_hw_params()` —— PLL 配置（最复杂的函数）

**目标**：根据采样率和位宽，配置 codec 时钟，优先 bypass PLL，否则用 PLL 生成最接近的时钟。

**步骤**：

1. **设置数据字长**（16/20/24/32 位）到 `ASD_INTF_CTRLB`。
2. **确定 Fsref**：采样率能被 11025 整除则为 44100，否则为 48000。
3. **尝试 bypass PLL**：遍历 Q=2..17，若 `sysclk / (128*Q) == fsref`，则 bypass PLL（CODEC_CLKIN 走 CLKDIV），禁用 PLL。
4. **否则启用 PLL**：CODEC_CLKIN 走 PLLDIV。
5. **配置数据路径**：左 DAC→左声道、右 DAC→右声道；≥64kHz 启用双速模式。
6. **配置采样率寄存器**。
7. **PLL 参数穷举搜索**（若未 bypass）：
   - 目标 `codec_clk = (2048 * fsref) / (sysclk / 1000)`（sysclk 除以 1000 防溢出）。
   - **先试 d=0**：遍历 r=1..16、p=1..8、j=4..55，找最接近 `codec_clk` 的组合；精确匹配则提前退出。
   - **再试 d≠0**：j 限制在 4..11，计算 d 为小数部分，同样找最接近值。
   - 若找不到任何有效值，报 `unable to setup PLL` 并返回 `-EINVAL`。
8. **写入 PLL 寄存器**：P、R、J、D（分 MSB/LSB 两个寄存器）。

### 2. `aic3x_set_power()` —— 电源管理

**上电**：
1. `regulator_bulk_enable()` 使能四路电源。
2. 若有 reset GPIO：`udelay(1)` 后拉高（释放复位）。
3. `regcache_cache_only(false)` + `regcache_sync()`：把缓存的寄存器值同步到硬件。
4. **PLL D 寄存器重写**：检查 PLL_PROGC/PLL_PROGD 是否为默认值，若是则重写——防止缓存同步跳过其中一个导致另一个也不写。
5. `mdelay(50)`：等待寄存器同步完成，减少爆音。

**下电**：
1. 写 `SOFT_RESET`：清除可能的 VDD 漏电流（即使电源调节器保持开启）。
2. `regcache_mark_dirty()`：标记缓存为脏，下次上电需重新同步。
3. `regcache_cache_only(true)`：停止硬件写入。
4. `regulator_bulk_disable()`：关闭四路电源。

### 3. `aic3x_init()` —— codec 初始化

1. 选 PAGE0，写软件复位。
2. **DAC**：设默认音量 + 静音（`DEFAULT_VOL | MUTE_ON`）。
3. **DAC→输出路由**：DAC 到 HPLOUT/HPROUT/HPLCOM/HPRCOM/LLOPM/RLOPM 设默认音量 + `ROUTE_ON`。
4. **取消所有输出静音**：写 `UNMUTE` 到六个输出控制寄存器。
5. **ADC**：设默认增益（`DEFAULT_GAIN`），默认路由 Line1 到 PGA。
6. **PGA→输出旁路**：设默认音量，断开路由。
7. **3104 特殊处理**：Line2→HP/Line 的寄存器在 3104 上是保留寄存器，**必须不写**——所以用 `if (model != 3104)` 包裹。
8. **按型号初始化**：3X/33/3106 调用 `aic3x_mono_init()`；3007 清零 Class-D 控制寄存器。
9. **设置输出共模电压 OCMV**（默认 1.5V）。

### 4. `aic3x_probe()` —— 驱动探测入口

1. 分配 `aic3x_priv`，保存 regmap，开启 cache-only 模式。
2. **解析平台数据或设备树**：
   - pdata：直接取 `gpio_reset`、`setup`、`micbias_vg`。
   - 设备树：读 `reset-gpios`（优先）或已废弃的 `gpio-reset`（会打 warning）；读 `ai3x-gpio-func`；读 `ai3x-micbias-vg`（1=2.0V, 2=2.5V, 3=AVDD）。
   - 都没有：`gpio_reset = -1`。
3. 保存型号 ID。
4. 请求 reset GPIO（若非共享），设为输出低电平（保持复位）。
5. 请求四路电源调节器。
6. 配置 OCMV。
7. 3007 型号：注册 Class-D 寄存器补丁。
8. 注册 ASoC component（`devm_snd_soc_register_component`）。
9. 加入全局 `reset_list`（用于共享 reset GPIO 检测）。

### 5. `aic3x_component_probe()` —— ASoC component 探测

1. 为四路电源注册 regulator notifier。
2. `regcache_mark_dirty()` + `aic3x_init()`：执行硬件初始化。
3. 若有 GPIO 配置且非 3104：写 GPIO1/GPIO2 功能寄存器。
4. 3104 不支持 GPIO 功能，会打 warning。

### 6. `aic3x_set_dai_fmt()` —— DAI 格式

支持：
- **协议**：I2S、左对齐、右对齐、DSP（A/B 模式）。
- **主从**：`BIT_CLK_MASTER` / `WORD_CLK_MASTER` 独立配置。
- **时钟极性**：位时钟反相、字时钟反相。

### 7. `aic3x_is_shared_reset()` —— 共享复位检测

- 遍历全局 `reset_list`，若已有其他 codec 实例使用同一个 GPIO，则认为复位引脚共享。
- 共享时不重复 `gpio_request` / `gpio_free`，避免多 codec 冲突。

---

## 八、型号差异处理

| 型号 | 差异 |
|---|---|
| **3104**（本项目） | PGA 输入路由专用 widget/控件/路由表；Line2→HP/Line 寄存器为保留，**禁止写入**；不支持 GPIO 功能 |
| **3007** | 带 Class-D 功放（`CLASSD_CTRL`，地址与 MONOLOPM_CTRL 重叠）；专用 widget/路由；注册 Class-D 寄存器补丁 |
| **3X/33/3106** | 有 MONO_LOUT 单声道输出；支持 extra widget/路由；支持 GPIO 功能 |
| **31/32** | 无 MONO_LOUT；31 额外缺少 LINE1/2 和 MIC3 |

---

## 九、关键设计点与踩坑点

### 1. PLL D 寄存器的缓存同步陷阱

- `regcache_sync()` 可能跳过写 PLL_PROGC 或 PLL_PROGD（如果它认为值未变），而这两个寄存器是配对的——一个不写会导致另一个也不生效。
- **解决**：上电后检查这两个寄存器是否为默认值，若是则强制重写。

### 2. 3104 保留寄存器禁止写入

- 3104 的 Line2→HP/Line 路由寄存器是保留寄存器，写入会导致未定义行为。
- **解决**：`aic3x_init()` 中用 `if (model != AIC3X_MODEL_3104)` 包裹这些写入。

### 3. 输入混音器的 0xf 位域

- AIC3x 输入路由用 `0xf` 表示断开、非零表示连接，与常规 DAPM 语义相反。
- **解决**：自定义 `snd_soc_dapm_put_volsw_aic3x()`，统一映射。

### 4. MICBIAS 电源位与电压位共享

- MICBIAS 控制寄存器的电源位和电压位在同一字段，上电时必须先通电再设电压、断电时先清电压再断电。
- **解决**：`mic_bias_event()` 在 POST_PMU 和 PRE_PMD 分别处理。

### 5. 输出级音量的非线性刻度

- 输出级寄存器在低音量区实际衰减比标称值大得多（寄存器 117 标称 -58.5dB，实际 -78.3dB）。
- **解决**：TLV 刻度只保证 -55~0dB 范围准确，低端标注为近似值。

### 6. AGC 可能永不回落

- 代码注释明确警告：AGC 可能在 ADC 开启时把 PGA 拉到最大值且永不回落。

### 7. 共享 reset GPIO

- 多 codec 共享一个复位引脚时，重复 `gpio_request` 会失败。
- **解决**：全局 `reset_list` + `aic3x_is_shared_reset()` 检测。

### 8. 下电时的软复位

- 即使电源调节器保持开启（共享电源场景），下电时也写软复位，清除 VDD 漏电流。

---

## 十、DAI 能力

| 维度 | 规格 |
|---|---|
| DAI 名 | `tlv320aic3x-hifi` |
| 通道 | 立体声（2ch，播放和捕获均为 2） |
| 采样率 | 8000 ~ 96000 Hz（`SNDRV_PCM_RATE_8000_96000`） |
| 格式 | S16_LE / S20_3LE / S24_3LE / S24_LE / S32_LE |
| 操作 | hw_params / prepare / mute_stream / set_sysclk / set_fmt / set_tdm_slot |
| 特性 | `no_capture_mute = 1`（捕获流不做静音） |

---

## 十一、与其他文件的关联

| 关联点 | 说明 |
|---|---|
| `tlv320aic3x.h` | 本文件 `#include` 该头，使用全部寄存器/位域/型号宏 |
| `tlv320aic3x-i2c.c` | I2C 层调用本文件导出的 `aic3x_probe` / `aic3x_remove` / `aic3x_regmap` |
| `<sound/tlv320aic3x.h>` | UAPI 头，定义 `aic3x_pdata` / `aic3x_setup_data` / `aic3x_micbias_voltage`，本文件用于解析平台数据 |
| `Makefile` | 本文件编译为 `tlv320aic3x.o`，链接成核心模块 |
| `dkms.conf` | 核心模块名 `snd-soc-tlv320aic3x` 与本驱动对应 |
| `aic3104-init.sh` | 依赖本驱动 probe 成功后注册的 ALSA 声卡，再设置混音器电平 |
| `robotd.toml` `[audio]` | `device = "plughw:aic3104"` 引用本驱动创建的声卡；`pet_detect` 依赖本驱动的捕获路径 |
| 设备树 overlay | `compatible = "ti,tlv320aic3104"`、`reset-gpios`、`ai3x-micbias-vg` 等属性由本驱动解析 |

---

## 十二、速查表

| 维度 | 内容 |
|---|---|
| 模块名 | `snd-soc-tlv320aic3x` |
| 代码行数 | ~1888 |
| 寄存器数 | 110（0–109） |
| 电源路数 | 4（IOVDD/DVDD/AVDD/DRVDD） |
| 支持型号 | 5 个（3X/33/3007/3104/3106） |
| 本项目型号 | AIC3X_MODEL_3104 |
| 缓存策略 | REGCACHE_RBTREE，仅 reset 寄存器 volatile |
| PLL 策略 | 优先 bypass（Q=2..17），否则穷举 j/d/r/p 找最接近值 |
| 采样率 | 8k–96kHz，Fsref 44.1k/48k，双速模式 ≥64k |
| 位宽 | 16/20/24/32 |
| DAC 音量范围 | -63.5 ~ 0 dB（0.5 dB 步） |
| ADC 增益范围 | 0 ~ 59.5 dB（0.5 dB 步） |
| 输出引脚音量 | 0 ~ 9 dB（1 dB 步） |
| MICBIAS 电压 | 2.0V / 2.5V / AVDD / OFF |
| OCMV | 1.35 / 1.5 / 1.65 / 1.8 V（默认 1.5V） |
| 导出符号 | `aic3x_probe`、`aic3x_remove`、`aic3x_regmap` |
| 作者 | Vladimir Barinov (MontaVista, 2007) |
#（注：内容由AI生成）
