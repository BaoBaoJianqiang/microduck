# 解析：`audio/aic3x-dkms/tlv320aic3x.h`

## 这是什么

TLV320AIC3x 编解码器驱动的**私有头文件**（301 行）。它只被本目录的两个 `.c` 包含，作用有两个：向 I2C 胶水层声明核心驱动导出的接口；集中定义芯片寄存器空间、位域与几个小 API 的常量。上游出处：作者 Vladimir Barinov（MontaVista Software, 2007），许可证 GPL-2.0-only。

注意区分：本文件是 DKMS 目录里的**驱动私有头**；驱动同时包含内核树内的 UAPI 头 `<sound/tlv320aic3x.h>`（`tlv320aic3x.c` 第 48 行），平台数据结构 `struct aic3x_pdata`、`struct aic3x_setup_data`、MICBIAS 电压枚举、`AIC3X_RATES`/`AIC3X_FORMATS` 等定义在那边。

## 第一部分：跨编译单元的接口声明（第 12-17 行）

```c
extern const struct regmap_config aic3x_regmap;
int aic3x_probe(struct device *dev, struct regmap *regmap, kernel_ulong_t driver_data);
void aic3x_remove(struct device *dev);
```

这三个符号由 `tlv320aic3x.c` 定义并导出，`tlv320aic3x-i2c.c` 调用——这正是"I2C 层极薄、核心共享"结构的接口面。参数 `driver_data` 是型号枚举。

## 第二部分：型号枚举（第 19-23 行）

```c
#define AIC3X_MODEL_3X 0
#define AIC3X_MODEL_33 1
#define AIC3X_MODEL_3007 2
#define AIC3X_MODEL_3104 3
#define AIC3X_MODEL_3106 4
```

五个芯片型号，I2C ID 表与设备树匹配表各配五条。本项目实际使用 `AIC3X_MODEL_3104`（TLV320AIC3104）。

## 第三部分：寄存器空间定义（第 25-173 行）

`AIC3X_CACHEREGNUM = 110`：寄存器缓存总量。以下按功能组列出代表性寄存器（第 0~109 页 0 寄存器）：

| 组 | 代表寄存器 | 用途 |
|---|---|---|
| 基础控制 | `AIC3X_PAGE_SELECT`(0)、`AIC3X_RESET`(1)、`AIC3X_SAMPLE_RATE_SEL_REG`(2) | 页选择（芯片寄存器分页）、软复位、采样率 |
| PLL | `AIC3X_PLL_PROGA~PROGD_REG`(3~6)、`AIC3X_OVRF_STATUS_AND_PLLR_REG`(11) | PLL P/Q/R/J/D 四组编程寄存器 |
| 数据通路 | `AIC3X_CODEC_DATAPATH_REG`(7)、`AIC3X_CODEC_DFILT_CTRL`(12) | 数据通路设置（FSREF 44.1k/48k 选择等）、数字滤波/ADC HPF |
| 音频接口 | `AIC3X_ASD_INTF_CTRLA/B/C`(8~10) | 音频串行数据接口（I2S 等）主从与格式 |
| 输入 | `LADC_VOL`/`RADC_VOL`(15/16)、`MIC3LR_2_L/RADC_CTRL`(17/18)、`LINE1*`/`LINE2*` 系列(19~24)、`MICBIAS_CTRL`(25)、AGC 系列(26~31) | ADC 增益、MIC3/Line1/Line2 到左右 ADC 的路由、MICBIAS、AGC |
| 输出混音 | `DAC_PWR`(37)、`HPOUT_SC`(40)、`DAC_LINE_MUX`(41)、`LDAC_VOL`/`RDAC_VOL`(43/44)，以及 `LINE2x_2_*`/`PGAx_2_*`/`DACx1_2_*` 大组(45~93) | 每个 Output stage（HPLOUT/HPLCOM/HPROUT/HPRCOM/MONOLOP+M/LLOP/RLOP）各有一组"来源选择 + 音量"寄存器；`CLASSD_CTRL`(73) 是 aic3007 的 Class-D 扬声器驱动（与 MONOLOP 寄存器号重叠，按型号区分） |
| GPIO/IRQ/时钟 | `AIC3X_STICKY_IRQ_FLAGS_REG`(96)~`AIC3X_GPIOB_REG`(101)、`AIC3X_CLKGEN_CTRL_REG`(102) | 中断标志、GPIO1/2 配置、时钟生成（Codec_ClkIn、PLL_CLKIN、CLKDIV_IN 的源选择） |
| 后段 | `LAGCN_ATTACK/DECAY`(103~106)、`NEW_ADC_DIGITALPATH`(107)、`PASSIVE_BYPASS`(108)、`DAC_ICC_ADJ`(109) | 新 AGC 攻击/衰减、ADC 数字通路、断电旁路、DAC 静态电流 |

## 第四部分：位域定义（第 175-262 行）

- **页选择**：`PAGE0_SELECT`/`PAGE1_SELECT`。
- **接口 A**：`BIT_CLK_MASTER`、`WORD_CLK_MASTER`（主从）、`DOUT_TRISTATE`。
- **数据通路**：`FSREF_44100`/`FSREF_48000`、双速率模式、DAC 到左右声道/单声道的交换与混合位。
- **PLL 位域**：`PLLP_SHIFT/MASK`、`PLLQ_SHIFT`、`PLLR_SHIFT`、`PLLJ_SHIFT`、`PLLD_MSB/LSB_SHIFT`。
- **时钟生成**：`CODEC_CLKIN_PLLDIV/CLKDIV`、`PLL_CLKIN_SHIFT`、`PLLCLK_IN_MASK`、`CLKDIV_IN_MASK`、时钟源 `CLKIN_MCLK/GPIO2/BCLK`。
- **开关位**：`SOFT_RESET`(0x80)、`PLL_ENABLE`(0x80)、路由 `ROUTE_ON`(0x80)、静音 `UNMUTE`(0x08)/`MUTE_ON`(0x80)、各路 `*_PWR_ON` 位。
- **音量约定**：`INVERT_VOL(val) (0x7f - val)`——芯片音量寄存器是反向刻度；`DEFAULT_VOL = INVERT_VOL(0x50)`（默认输出音量）、`DEFAULT_GAIN = 0x20`（默认输入增益）。
- **MICBIAS**：`MICBIAS_LEVEL_SHIFT/MASK`（电压档位）。
- **HPOUT_SC 的 OCMV**：输出共模电压四档 `HPOUT_SC_OCMV_1_35V/1_5V/1_65V/1_8V`——核心驱动里 `aic3x_configure_ocmv()` 按供电电压或设备树 `ai3x-ocmv` 属性选档时用的就是这套。

## 第五部分：耳机检测 / 按钮检测 API（第 264-299 行）

芯片支持立体声耳机（GND + 左 + 右）与手机耳机（GND + 扬声器 + 麦克风）插入检测，注释建议启用 MICBIAS 以保证功能正常，细节查数据手册。三个枚举加一组移位/掩码宏：

- `AIC3X_HEADSET_DETECT_OFF/STEREO/CELLULAR/BOTH`（检测模式 0~3）；
- `AIC3X_HEADSET_DEBOUNCE_16MS~512MS`（插入去抖六档）；
- `AIC3X_BUTTON_DEBOUNCE_0MS~32MS`（按钮去抖四档）；
- `AIC3X_HEADSET_DETECT_ENABLED`(0x80) 与 `*_SHIFT`/`*_MASK` 宏，用于把上述选项装配进 `AIC3X_HEADSET_DETECT_CTRL_A/B`(13/14) 寄存器。

本项目的机器人不使用耳机检测功能，这部分是上游驱动的完整携带。
