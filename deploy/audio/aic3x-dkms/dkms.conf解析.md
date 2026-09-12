# 解析：`audio/aic3x-dkms/dkms.conf`

## 这是什么

DKMS（Dynamic Kernel Module Support）的包描述文件。`deploy/audio/aic3x-dkms/` 目录是一个内核模块源码包，本文件告诉 DKMS 这个包叫什么、构建出哪些模块、怎么构建、装到哪里。配合同目录的 `Makefile`（构建规则）和 `tlv320aic3x.c` / `tlv320aic3x-i2c.c`（驱动源码）一起工作——机器人的音频编解码器 TLV320AIC3104 需要这两个内核模块，而 Armbian 官方镜像里没有，所以用 DKMS 在目标板上随内核版本自动编译安装。

## 逐行解析

| 行 | 内容 | 含义 |
|---|---|---|
| 1 | `PACKAGE_NAME="aic3x"` | DKMS 包名，DKMS 以 `aic3x/<版本>` 管理这个包 |
| 2 | `PACKAGE_VERSION="6.1-rkr5.1"` | 包版本号，取自 Rockchip 6.1 BSP 内核的 tag（rkr5.1），表明这份源码对应哪个内核版本系的驱动 |
| 3 | `BUILT_MODULE_NAME[0]="snd-soc-tlv320aic3x"` | 第一个构建产物：编解码器核心模块（来自 `tlv320aic3x.c`） |
| 4 | `BUILT_MODULE_NAME[1]="snd-soc-tlv320aic3x-i2c"` | 第二个构建产物：I2C 总线接口模块（来自 `tlv320aic3x-i2c.c`） |
| 5-6 | `DEST_MODULE_LOCATION[0/1]="/updates/dkms"` | 两个模块都安装到内核模块树的 `/updates/dkms` 目录下 |
| 7 | `MAKE[0]="make -C ${kernel_source_dir} M=${dkms_tree}/..."` | 构建命令：进入内核源码目录、以外部模块方式（`M=` 指向 DKMS 的构建目录）执行 `make modules`——这正是内核外部模块的标准构建方式，要求板上装有内核头文件 |
| 8 | `CLEAN="make -C ... clean"` | 对应的清理命令 |
| 9 | `AUTOINSTALL="yes"` | 内核升级后 DKMS 自动为新内核重新编译并安装，无需人工干预 |

## 关键点

- **两个模块的分工**：`snd-soc-tlv320aic3x` 是 ASoC 编解码器驱动核心（寄存器、混音、DAPM、DAI），`snd-soc-tlv320aic3x-i2c` 是 I2C 探测入口。芯片通过 I2C 挂在总线上，内核先匹配到 I2C 模块，再由它调用核心模块的 `aic3x_probe()`。
- **版本号 `6.1-rkr5.1`**：这不是本仓库自己的版本号，而是标注源码来源的内核版本系（Rockchip 内核 6.1 第 5 轮发布）。DKMS 用它在内核升级时判断是否需要重编。
- **`AUTOINSTALL="yes"` 是能长期用的前提**：机器人会随更新升级内核，没有它，每次内核升级后声卡都会消失，直到有人手动 `dkms install`。
