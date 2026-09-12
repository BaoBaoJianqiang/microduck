# 解析：`audio/aic3x-dkms/tlv320aic3x.c`

## 这是什么

TI TLV320AIC3x 系列音频编解码器的 **ALSA SoC（ASoC）驱动核心**（1888 行）。上游出处：作者 Vladimir Barinov（MontaVista Software, 2007），基于 Liam Girdwood 的 `wm8753.c`，许可证 GPL-2.0-only。本目录把它从内核树 `sound/soc/codecs/` 拿出来，经 DKMS 在目标板上随内核编译成 `snd-soc-tlv320aic3x.ko`（配合同目录的 `Makefile` 与 `dkms.conf`）。

头注释说明了芯片家族兼容性：完整支持 aic33；aic32/aic3007 缺 MONO_LOUT，aic31 输入管脚命名不同（IN1L→LINE1L 等），机器层（machine driver）应通过 `snd_soc_dapm_disable_pin()` 关掉不支持的路由。本项目使用的是 AIC3104 型号。

它在本部署里的角色：Radxa Zero 3W 的音频编解码器是挂在 I2C 总线上的 TLV320AIC3104，`snd-soc-tlv320aic3x-i2c.ko` 探测到芯片后调用本文件导出的 `aic3x_probe()` 完成全部注册；之后 ALSA 出现 `aic3104` 声卡，混音器由 [`aic3104-init.sh`](../aic3104-init.sh解析.md) 设置，`robotd` 的 `[audio] device = "plughw:aic3104"` 读写它。

## 文件结构总览（按行号）

| 行号 | 内容 |
|---|---|
| 52-58 | 四路供电名：`IOVDD`（I/O）、`DVDD`（数字核）、`AVDD`（模拟 DAC）、`DRVDD`（ADC 模拟与输出驱动） |
| 60 | `static LIST_HEAD(reset_list)`——跨实例的共享复位跟踪链表 |
| 64-90 | `struct aic3x_disable_nb`（稳压器关闭通知块）与 `struct aic3x_priv`（驱动私有数据：component、regmap、4 路稳压器及通知块、setup 数据、sysclk、dai_fmt、tdm_delay、slot_width、master、gpio_reset、power、model、micbias_vg、ocmv） |
| 92-121 | `aic3x_reg[]`——110 个寄存器的复位默认值表 |
| 123-131 | `aic3x_volatile_reg()`——仅 `AIC3X_RESET` 是易失寄存器 |
| 133-142 | `aic3x_regmap`——通用 regmap 配置（max_register 到 `DAC_ICC_ADJ`、默认值表、RBTREE 缓存），`EXPORT_SYMBOL_GPL` 供 I2C 层取用 |
| 144-194 | 自定义 DAPM put 回调 `snd_soc_dapm_put_volsw_aic3x()`——见下"特殊处理" |
| 205-225 | `mic_bias_event()`——MICBIAS 上电时把寄存器 reclaim 为用户设定电压、断电前置 0（MICBIAS 电源与电压档共用同一组位） |
| 227-298 | 枚举控件声明：DAC 输出 mux（DAC_L1/3/2）、HPCOM 模式（差分/恒定 VCM/单端等）、Line 输入单端/差分模式、ADC HPF 四档、AGC 目标电平/攻击/衰减、POP 抑制的上电时间与斜坡步进 |
| 303-318 | dB 刻度表：DAC 数字音量 -63.5~0 dB（0.5 dB 步）、ADC PGA 0~59.5 dB、输出级 -78.3~0 dB（近似线性段 -55~0，注释解释低段差异）、输出级 0~9 dB |
| 320-630 | 控件表：`aic3x_snd_controls`（PCM Playback Volume、各混音旁路音量、Line/HP/HPCOM 的 Volume+Switch、AGC Switch 等）、`aic3x_extra_snd_controls`、`aic3x_mono_controls`、六个输出级 mixer 控件表、左右 PGA 输入 mixer 控件表、以及 **aic3104 专用的 PGA mixer 变体**（598-606 行 `aic3104_left/right_pga_mixer_controls`）、`classd_amp_tlv`(482) |
| 633-810 | DAPM widget 表：`aic3x_dapm_widgets`、`aic3x_extra_dapm_widgets`、**`aic3104_extra_dapm_widgets`**(757)、单声道变体、aic3007 的 Class-D widget |
| 812-996 | DAPM 路由表：`intercon`（主音频图）、`intercon_extra`、**`intercon_extra_3104`**(968)、`intercon_mono`、`intercon_3007` |
| 998-1035 | `aic3x_add_widgets()`——按型号拼装上面的 widget/路由子集 |
| 1037-1189 | `aic3x_hw_params()`——采样率与 PLL/分频器计算的核心 |
| 1190-1212 | `aic3x_prepare()`、`aic3x_mute()` |
| 1230-1309 | `aic3x_set_dai_sysclk()`、`aic3x_set_dai_fmt()`（I2S/左对齐/右对齐/DSP 格式与主从协商） |
| 1310-1356 | `aic3x_set_dai_tdm_slot()`——TDM 时隙 |
| 1357-1473 | 电源管理：`aic3x_regulator_event()`（稳压器关闭通知）、`aic3x_set_power()`（断电前 regcache 置 cache-only、上电后软复位并同步缓存）、`aic3x_set_bias_level()` |
| 1474-1500 | `aic3x_dai_ops`（hw_params/prepare/mute_stream/set_sysclk/set_fmt/set_tdm_slot，`no_capture_mute = 1`）与 `aic3x_dai`——名为 `tlv320aic3x-hifi`，播放/采集各 2 声道、`AIC3X_RATES`/`AIC3X_FORMATS`（定义于内核 UAPI 头 `<sound/tlv320aic3x.h>`）、`symmetric_rate = 1` |
| 1502-1597 | `aic3x_mono_init()`（MONO_LOP 默认音量与路由）与 `aic3x_init()`——切页 0、软复位、DAC 默认音量并静音等出厂化 |
| 1598-1610 | `aic3x_is_shared_reset()`——同一 reset GPIO 挂多个 codec 时的共享判断 |
| 1611-1688 | `aic3x_component_probe()`——申请 reset GPIO 并拉低、设置 micbias/GPIO 功能寄存器、注册 DAI 与 widget、使能稳压器并挂关闭通知块 |
| 1689-1700 | `soc_component_dev_aic3x`——component 驱动：`set_bias_level`、控件/widget/路由表、`use_pmdown_time`、`endianness` |
| 1702-1736 | `aic3x_configure_ocmv()`——输出共模电压选档：设备树 `ai3x-ocmv` 优先；否则按 AVDD/DVDD 实测电压分档（1.8V/1.65V/1.5V/1.35V 四档，越界告警） |
| 1739-1747 | `aic3007_class_d[]`——aic3007 专属 Class-D 初始化寄存器序列（数据手册 p.46，切到页 0x0D 写三次 0x8） |
| 1749-1869 | `aic3x_probe()`——总线无关的公共 probe，见下 |
| 1871-1883 | `aic3x_remove()`——移出 reset_list、复位 GPIO 拉低并释放 |
| 1885-1887 | 模块元信息：`MODULE_DESCRIPTION/AUTHOR/LICENSE("GPL")` |

## 两个值得展开的机制

### 特殊的 DAPM put 回调（144-194 行）

芯片输入混音的连接位约定非常规：**任何非 `0xf` 位型都表示连接、`0xf` 才是断开**，与 ALSA 常规的 0/1 开关不同。因此输入 mixer 控件不能用标准 `snd_soc_dapm_put_volsw`，本文件写了 `snd_soc_dapm_put_volsw_aic3x()`：把用户值折叠成 `0xf`（断）或 `mask`（通），再调 `snd_soc_dapm_mixer_update_power()` 重算 DAPM 电源树。`SOC_DAPM_SINGLE_AIC3X` 宏（144 行）把这个回调封装给所有输入 mixer 控件使用。

### `aic3x_probe()` 的流程（1749-1869 行）

总线层（I2C/SPI）只负责递进 regmap，其余全在这里：

1. 分配 `aic3x_priv`，挂入 regmap，**`regcache_cache_only(true)`**——上电前只写缓存不碰总线；
2. 平台数据或设备树：读 `reset-gpios`（兼容废弃的 `gpio-reset` 并告警）、`ai3x-gpio-func`（GPIO1/2 功能）、`ai3x-micbias-vg`（1/2/3 → 2.0V/2.5V/AVDD，非法值报错并落到 OFF）；
3. `driver_data` 作为型号存入 `aic3x->model`；
4. reset GPIO 有效且非共享时 `gpio_request` 并拉低；
5. `devm_regulator_bulk_get()` 取四路供电；`aic3x_configure_ocmv()` 定输出共模电压档；
6. 型号为 aic3007 时注册 Class-D 寄存器补丁序列；
7. `devm_snd_soc_register_component()` 向 ASoC 注册 component + 一个 DAI；
8. 挂入 `reset_list` 支持共享复位。

`aic3x_remove()` 对称：摘链、拉低并释放复位 GPIO（稳压器与 regmap 由 devm 自动处理）。

## 与本项目相关的两个事实

- **本文件按原样携带上游驱动**：文件中未见针对本板卡的改写痕迹（probe、控件、DAPM 与上游结构一致）；对 3104 的特有支持走的是上游自带的 `aic3104_extra_dapm_widgets` / `intercon_extra_3104` / `aic3104_*_pga_mixer_controls` 分支。板级差异（引脚、混音电平、路由开关）全部交给设备树 overlay 与 `aic3104-init.sh`，这正是 DKMS 打包"不改驱动、只改板级配置"的用意。
- **构建依赖内核头文件**：`dkms.conf` 的 `MAKE[0]` 用 `-C ${kernel_source_dir}` 在内核源码树内构建，且本文件 include 的 `<sound/tlv320aic3x.h>` 等头来自内核自身——因此 `PACKAGE_VERSION="6.1-rkr5.1"` 标注的内核版本系决定了这份源码能对哪些内核头编译。
