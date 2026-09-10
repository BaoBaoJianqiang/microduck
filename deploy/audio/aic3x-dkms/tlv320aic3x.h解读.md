# `tlv320aic3x.h` 头文件解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | C 头文件（内核驱动公共接口） |
| 作用 | 定义 TLV320AIC3x codec 驱动的**寄存器地址、位域、型号常量、默认值、枚举和外部接口声明** |
| 作者 | Vladimir Barinov, MontaVista Software（2007） |
| 许可证 | GPL-2.0-only |
| 被引用 | `tlv320aic3x.c`（核心驱动）、`tlv320aic3x-i2c.c`（I2C 适配层） |
| 对应头文件 | 内核 UAPI 侧 `<sound/tlv320aic3x.h>`（定义平台数据结构 `aic3x_pdata` 等） |

---

## 二、外部接口声明

```c
extern const struct regmap_config aic3x_regmap;
int  aic3x_probe(struct device *dev, struct regmap *regmap, kernel_ulong_t driver_data);
void aic3x_remove(struct device *dev);
```

| 符号 | 类型 | 作用 |
|---|---|---|
| `aic3x_regmap` | `const struct regmap_config` | regmap 配置（寄存器缓存、默认值、volatile 判定），由总线层复制并覆盖位宽后使用 |
| `aic3x_probe` | 函数 | 核心探测入口：分配私有数据、解析平台数据/设备树、请求电源、注册 ASoC component |
| `aic3x_remove` | 函数 | 核心移除：从 reset_list 摘除、释放 GPIO |

- 这三个符号是核心驱动对总线适配层（I2C/SPI）的**全部导出接口**，体现了"核心与总线分离"的架构。

---

## 三、型号常量

```c
#define AIC3X_MODEL_3X    0
#define AIC3X_MODEL_33    1
#define AIC3X_MODEL_3007  2
#define AIC3X_MODEL_3104  3
#define AIC3X_MODEL_3106  4
```

| 常量 | 值 | 对应芯片 | 本项目 |
|---|---|---|---|
| `AIC3X_MODEL_3X` | 0 | aic31/aic32 通用 | |
| `AIC3X_MODEL_33` | 1 | TLV320AIC33 | |
| `AIC3X_MODEL_3007` | 2 | TLV320AIC3007（带 Class-D 功放） | |
| `AIC3X_MODEL_3104` | 3 | **TLV320AIC3104** | ✅ 本项目使用 |
| `AIC3X_MODEL_3106` | 4 | TLV320AIC3106 | |

- 型号 ID 通过 I2C 设备表的 `driver_data` 传入 `aic3x_probe`，驱动据此启用/禁用特定功能（如 3007 的 Class-D、3104 的保留寄存器规避）。

---

## 四、寄存器地址定义（共 110 个，0x00–0x6D）

`#define AIC3X_CACHEREGNUM 110` —— 寄存器缓存覆盖 0–109。

按功能分组：

### 1. 系统与时钟（0–12）

| 寄存器 | 地址 | 功能 |
|---|---|---|
| `AIC3X_PAGE_SELECT` | 0 | 页选择（PAGE0/PAGE1） |
| `AIC3X_RESET` | 1 | 软件复位（写 `SOFT_RESET=0x80`） |
| `AIC3X_SAMPLE_RATE_SEL_REG` | 2 | 采样率选择 |
| `AIC3X_PLL_PROGA/B/C/D_REG` | 3–6 | PLL 编程 A/B/C/D（P/Q/J/D 等） |
| `AIC3X_CODEC_DATAPATH_REG` | 7 | 编解码器数据路径（DAC 路由、Fsref、双速模式） |
| `AIC3X_ASD_INTF_CTRLA/B/C` | 8–10 | 音频串口控制（主从、位时钟、字时钟、TDM） |
| `AIC3X_OVRF_STATUS_AND_PLLR_REG` | 11 | 溢出状态 + PLL R 值 |
| `AIC3X_CODEC_DFILT_CTRL` | 12 | 数字滤波器控制（ADC HPF） |

### 2. 耳机检测（13–14）

`AIC3X_HEADSET_DETECT_CTRL_A/B` —— 耳机插入/按钮检测控制。

### 3. ADC 与输入路由（15–31）

| 寄存器 | 地址 | 功能 |
|---|---|---|
| `LADC_VOL` / `RADC_VOL` | 15/16 | 左右 ADC PGA 增益 |
| `MIC3LR_2_LADC_CTRL` / `MIC3LR_2_RADC_CTRL` | 17/18 | MIC3 到左右 ADC 路由 |
| `LINE1L/R_2_LADC/RADC_CTRL` | 19/21/22/24 | Line1 到 ADC 路由 |
| `LINE2L/R_2_LADC/RADC_CTRL` | 20/23 | Line2 到 ADC 路由 |
| `MICBIAS_CTRL` | 25 | MICBIAS 电压控制 |
| `LAGC_CTRL_A/B/C` / `RAGC_CTRL_A/B/C` | 26–31 | 左右 AGC（自动增益控制） |

### 4. DAC 与输出路由（37–93）

- `DAC_PWR` / `HPLCOM_CFG`（37）、`HPRCOM_CFG`（38）—— DAC 电源与高功率输出配置。
- `HPOUT_SC`（40）—— 高功率输出级控制（含输出共模电压 OCMV）。
- `DAC_LINE_MUX`（41）—— DAC 输出切换。
- `HPOUT_POP_REDUCTION`（42）—— 上电爆音抑制（上电时间、斜坡步长）。
- `LDAC_VOL` / `RDAC_VOL`（43/44）—— DAC 数字音量。
- 大量 `*_2_HPLOUT_VOL` / `*_2_HPROUT_VOL` / `*_2_HPLCOM_VOL` / `*_2_HPRCOM_VOL` / `*_2_LLOPM_VOL` / `*_2_RLOPM_VOL` / `*_2_MONOLOPM_VOL`（45–79）—— 各输入源（LINE2/PGA/DAC）到各输出端的路由与音量。
- `CLASSD_CTRL`（73，仅 3007）—— Class-D 功放控制（与 `MONOLOPM_CTRL` 地址重叠，因 3007 无 MONO_LOUT）。
- `LLOPM_CTRL` / `RLOPM_CTRL`（86/93）—— 左右线路输出控制（含 `Line Playback Switch` 和 `Line Playback Volume`）。

### 5. GPIO/IRQ 与时钟生成（96–102）

- `AIC3X_STICKY_IRQ_FLAGS_REG`（96）/ `AIC3X_RT_IRQ_FLAGS_REG`（97）—— 中断标志。
- `AIC3X_GPIO1_REG` / `AIC3X_GPIO2_REG`（98/99）—— GPIO 功能配置。
- `AIC3X_GPIOA_REG` / `AIC3X_GPIOB_REG`（100/101）—— GPIO 数据/时钟源选择。
- `AIC3X_CLKGEN_CTRL_REG`（102）—— 时钟生成控制（CODEC_CLKIN 来源）。

### 6. 新增 AGC 与数字路径（103–109）

- `LAGCN_ATTACK/DECAY` / `RAGCN_ATTACK/DECAY`（103–106）—— 新型 AGC 攻击/衰减时间。
- `NEW_ADC_DIGITALPATH`（107）—— 可编程 ADC 数字路径 + I2C 总线条件。
- `PASSIVE_BYPASS`（108）—— 掉电期间模拟信号旁路选择。
- `DAC_ICC_ADJ`（109）—— DAC 静态电流调整（`aic3x_regmap.max_register` 以此为界）。

---

## 五、寄存器位域定义

### 1. 音频串口（ASD_INTF_CTRLA）

| 宏 | 值 | 含义 |
|---|---|---|
| `BIT_CLK_MASTER` | `0x80` | 位时钟主模式 |
| `WORD_CLK_MASTER` | `0x40` | 字时钟主模式 |
| `DOUT_TRISTATE` | `0x20` | DOUT 三态 |

### 2. 数据路径（CODEC_DATAPATH_REG）

| 宏 | 值 | 含义 |
|---|---|---|
| `FSREF_44100` | `(1<<7)` | 参考采样率 44.1kHz |
| `FSREF_48000` | `(0<<7)` | 参考采样率 48kHz |
| `DUAL_RATE_MODE` | `((1<<5)\|(1<<6))` | 双速模式（≥64kHz） |
| `LDAC2LCH` / `RDAC2RCH` | `0x08` / `0x02` | 左 DAC→左声道、右 DAC→右声道（默认） |
| `LDAC2RCH` / `RDAC2LCH` | `0x10` / `0x04` | 交叉路由 |
| `LDAC2MONOMIX` / `RDAC2MONOMIX` | `0x18` / `0x06` | 单声道混合 |

### 3. PLL 位域

| 宏 | 含义 |
|---|---|
| `PLLP_SHIFT/MASK` | P 分频（shift=0, mask=7） |
| `PLLQ_SHIFT` | Q 分频（shift=3） |
| `PLLR_SHIFT` | R 分频（shift=0，在寄存器 11） |
| `PLLJ_SHIFT` | J 值（shift=2） |
| `PLLD_MSB_SHIFT` / `PLLD_LSB_SHIFT` | D 值高/低位 |

### 4. 时钟生成（CLKGEN_CTRL_REG）

| 宏 | 含义 |
|---|---|
| `CODEC_CLKIN_PLLDIV` / `CODEC_CLKIN_CLKDIV` | CODEC_CLKIN 来源：PLL 分频 / 直接分频 |
| `CLKIN_MCLK` / `CLKIN_GPIO2` / `CLKIN_BCLK` | PLL 输入时钟源：MCLK / GPIO2 / BCLK |

### 5. 通用控制位

| 宏 | 值 | 含义 |
|---|---|---|
| `SOFT_RESET` | `0x80` | 软件复位 |
| `PLL_ENABLE` | `0x80` | PLL 使能 |
| `ROUTE_ON` | `0x80` | 路由接通 |
| `UNMUTE` | `0x08` | 取消静音 |
| `MUTE_ON` | `0x80` | 静音 |
| `LADC_PWR_ON` / `RADC_PWR_ON` | `0x04` | ADC 上电 |
| `LDAC_PWR_ON` / `RDAC_PWR_ON` | `0x80`/`0x40` | DAC 上电 |
| `HPLOUT_PWR_ON` 等 | `0x01` | 各输出端上电 |

### 6. 默认值与音量反转

```c
#define INVERT_VOL(val)   (0x7f - val)
#define DEFAULT_VOL     INVERT_VOL(0x50)   // 输出默认音量（反转编码）
#define DEFAULT_GAIN    0x20               // 输入默认增益
```

- AIC3x 的输出音量寄存器采用**反转编码**：寄存器值越大，实际音量越小（0x7f 为静音）。`INVERT_VOL` 宏把"期望的线性值"转换为寄存器值。
- `DEFAULT_VOL = 0x7f - 0x50 = 0x2f`。

### 7. MICBIAS 与 OCMV

| 宏 | 含义 |
|---|---|
| `MICBIAS_LEVEL_SHIFT/MASK` | MICBIAS 电压位域（shift=6, mask=0xC0） |
| `HPOUT_SC_OCMV_MASK/SHIFT` | 输出共模电压位域 |
| `HPOUT_SC_OCMV_1_35V/1_5V/1_65V/1_8V` | OCMV 四档：1.35V / 1.5V / 1.65V / 1.8V |

---

## 六、耳机检测枚举

### 检测模式

```c
enum {
    AIC3X_HEADSET_DETECT_OFF      = 0,  // 关闭
    AIC3X_HEADSET_DETECT_STEREO   = 1,  // 立体声耳机（GND+L+R）
    AIC3X_HEADSET_DETECT_CELLULAR = 2,  // 手机耳机（GND+扬声器+麦克风）
    AIC3X_HEADSET_DETECT_BOTH     = 3,  // 两者都检测
};
```

### 去抖时间

| 枚举 | 时间 |
|---|---|
| `AIC3X_HEADSET_DEBOUNCE_16MS` … `_512MS` | 16 / 32 / 64 / 128 / 256 / 512 ms |
| `AIC3X_BUTTON_DEBOUNCE_0MS` … `_32MS` | 0 / 8 / 16 / 32 ms |

- 注释明确建议：启用耳机检测时应同时开启 MICBIAS，否则检测不可靠。

---

## 七、关键设计点

### 1. 寄存器地址与位域的集中管理

- 所有 110 个寄存器地址和关键位域都在头文件中以 `#define` 集中定义，核心驱动 `.c` 文件只引用符号名。
- 这使得寄存器级别的修改（如发现数据手册勘误）只需改头文件一处。

### 2. 型号差异通过运行时判断而非编译时

- 5 个型号共享同一套寄存器定义，差异（如 3007 的 Class-D、3104 的保留寄存器）在 `.c` 文件中通过 `aic3x->model` 运行时判断。
- 这意味着一个驱动二进制支持所有型号，通过设备树/I2C 匹配选择行为。

### 3. 音量反转宏

- `INVERT_VOL` 封装了 AIC3x 输出音量寄存器的反转编码特性，避免驱动代码中到处出现 `0x7f - x` 的魔法表达式。

---

## 八、与其他文件的关联

| 关联点 | 说明 |
|---|---|
| `tlv320aic3x.c` | 核心驱动 `#include "tlv320aic3x.h"`，使用所有寄存器/位域/型号宏 |
| `tlv320aic3x-i2c.c` | I2C 层 `#include "tlv320aic3x.h"`，使用型号常量和 `aic3x_probe` 声明 |
| `<sound/tlv320aic3x.h>` | 内核 UAPI 头，定义平台数据 `aic3x_pdata`、`aic3x_setup_data`、`aic3x_micbias_voltage` 等，本文件不重复定义 |
| 设备树 overlay | `compatible = "ti,tlv320aic3104"` 对应 `AIC3X_MODEL_3104` |

---

## 九、速查表

| 维度 | 内容 |
|---|---|
| 寄存器总数 | 110（0x00–0x6D） |
| 支持型号 | 5 个（3X/33/3007/3104/3106） |
| 本项目型号 | AIC3X_MODEL_3104 (=3) |
| 外部接口 | `aic3x_regmap`、`aic3x_probe()`、`aic3x_remove()` |
| 页选择 | PAGE0=0, PAGE1=1 |
| 软件复位 | 写 0x80 到寄存器 1 |
| 输出音量编码 | 反转（`INVERT_VOL`），0x7f=静音 |
| 默认输出音量 | `INVERT_VOL(0x50)` = 0x2f |
| 默认输入增益 | 0x20 |
| MICBIAS 电压 | 2.0V / 2.5V / AVDD（由平台数据选择） |
| OCMV 档位 | 1.35 / 1.5 / 1.65 / 1.8 V |
| 采样率参考 | 44.1kHz 或 48kHz（双速模式支持 ≥64kHz） |
| 耳机检测 | 4 种模式 + 6 档去抖 + 4 档按钮去抖 |
#（注：内容由AI生成）
