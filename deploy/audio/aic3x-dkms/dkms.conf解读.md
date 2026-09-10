# `dkms.conf` 配置解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | DKMS（Dynamic Kernel Module Support）配置文件 |
| 作用 | 告诉 DKMS 如何构建、安装和管理 TLV320AIC3x 音频 codec 的两个内核模块 |
| 关联文件 | 同目录 `Makefile`（构建规则）、`tlv320aic3x.c` / `tlv320aic3x-i2c.c`（源码） |
| 安装位置 | 随 DKMS 源码树放入 `/usr/src/aic3x-6.1-rkr5.1/` |

---

## 二、配置项逐行解读

| 配置项 | 值 | 含义 |
|---|---|---|
| `PACKAGE_NAME` | `"aic3x"` | DKMS 包名，模块源码目录名为 `aic3x-<version>` |
| `PACKAGE_VERSION` | `"6.1-rkr5.1"` | 包版本。`6.1` 对应内核版本基线，`rkr5.1` 是 Rockchip（rkr）内核补丁版本后缀——表明该驱动是从 Rockchip 6.1 内核 r5.1 树中提取出来做外部构建的 |
| `BUILT_MODULE_NAME[0]` | `"snd-soc-tlv320aic3x"` | 第一个构建产物：codec 核心驱动模块（`.ko`） |
| `BUILT_MODULE_NAME[1]` | `"snd-soc-tlv320aic3x-i2c"` | 第二个构建产物：I2C 接口层模块（`.ko`） |
| `DEST_MODULE_LOCATION[0]` | `"/updates/dkms"` | 核心模块安装到 `/lib/modules/<kernel>/updates/dkms/` |
| `DEST_MODULE_LOCATION[1]` | `"/updates/dkms"` | I2C 模块安装到同一目录 |
| `MAKE[0]` | `make -C ${kernel_source_dir} M=${dkms_tree}/.../build modules` | 构建命令：进入内核源码目录，以本包 build 目录为外部模块目录执行 `modules` 目标——标准的内核外部构建（Kbuild）方式 |
| `CLEAN` | `make -C ${kernel_source_dir} M=... clean` | 清理命令，与构建对称 |
| `AUTOINSTALL` | `"yes"` | 内核升级时自动重新构建并安装这两个模块——这是 DKMS 的核心价值：换内核不用手动重装驱动 |

---

## 三、关键设计点

### 1. 为什么需要 DKMS 外部构建

- 该 codec 驱动虽然存在于主线内核 `sound/soc/codecs/`，但 Radxa/Armbian 使用的 Rockchip 内核可能版本不对、或缺少对 `tlv320aic3104` 的特定支持。
- 通过 DKMS 把驱动**独立于内核发行版**维护，可以在不重编整个内核的情况下部署/更新驱动。
- `AUTOINSTALL=yes` 确保每次 `apt upgrade` 升级内核后，模块自动针对新内核重编译——这与 `README.md` 中"内核升级可能撤销设备树 overlay 补丁"的风险形成互补。

### 2. 两个模块的分离

| 模块 | 职责 | 依赖 |
|---|---|---|
| `snd-soc-tlv320aic3x.ko` | codec 核心逻辑：寄存器、DAPM、控件、PLL、电源管理 | regmap、ASoC 核心 |
| `snd-soc-tlv320aic3x-i2c.ko` | I2C 总线适配层：设备匹配、regmap 初始化 | 核心模块 + I2C 子系统 |

- 分离的好处：核心驱动可复用于 SPI 等其他总线接口（虽然本包只提供 I2C），符合 Linux 内核"核心与总线接口分离"的惯例。
- 模块名与主线内核命名一致（`snd-soc-tlv320aic3x[-i2c]`），确保 `modprobe` / udev 自动加载行为与主线一致。

### 3. 构建路径的变量

- `${kernel_source_dir}`：DKMS 注入的当前内核源码/头文件目录（通常 `/lib/modules/$(uname -r)/build`）。
- `${dkms_tree}`：DKMS 工作树（默认 `/var/lib/dkms`）。
- `M=.../build`：Kbuild 的外部模块目录参数，告诉内核构建系统"到这个目录找 Makefile 和源码"。

---

## 四、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `Makefile` | `MAKE[0]` 调用的 `make` 实际执行同目录 `Makefile` 中的 Kbuild 规则 |
| `setup-board.sh` | 负责触发 DKMS 安装（`dkms install`），是本配置生效的前提 |
| `aic3104-init.sh` | 依赖本驱动加载后创建的 ALSA 声卡 `aic3104`，再设置混音器电平 |
| `robotd.toml` `[audio] device` | `plughw:aic3104` 引用的正是本驱动注册的声卡 |
| 内核升级 | `AUTOINSTALL=yes` 保证新内核自动重编；但设备树 overlay 补丁仍可能被内核升级撤销（见 README ⚠） |

---

## 五、速查表

| 维度 | 值 |
|---|---|
| 包名 | `aic3x` |
| 版本 | `6.1-rkr5.1` |
| 模块数 | 2（核心 + I2C） |
| 模块名 | `snd-soc-tlv320aic3x`、`snd-soc-tlv320aic3x-i2c` |
| 安装位置 | `/lib/modules/<kernel>/updates/dkms/` |
| 构建方式 | 内核外部 Kbuild（`make -C $kernel M=$build modules`） |
| 自动重装 | 是（`AUTOINSTALL=yes`） |
| 源码目录 | `/usr/src/aic3x-6.1-rkr5.1/` |
#（注：内容由AI生成）
