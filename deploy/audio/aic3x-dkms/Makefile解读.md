# `Makefile` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Linux 内核外部模块构建 Makefile（Kbuild 格式） |
| 作用 | 定义两个内核模块的目标名与源文件映射，供 `dkms.conf` 中的 `MAKE[0]` 调用 |
| 关联文件 | `dkms.conf`（调用方）、`tlv320aic3x.c` / `tlv320aic3x-i2c.c`（源码） |
| 命名约定 | 与主线内核 `sound/soc/codecs/Makefile` 一致，确保模块名匹配 |

---

## 二、逐行解读

```makefile
# Out-of-tree build of the TI TLV320AIC3x ASoC codec driver (core +
# I2C interface), mirroring sound/soc/codecs/Makefile in-tree naming so
# the modules land as snd-soc-tlv320aic3x[-i2c].ko. Built via DKMS —
# see dkms.conf and README.md in this directory.
```

- **Out-of-tree build**：明确这是**外部构建**，不参与内核整体编译。
- **mirroring in-tree naming**：刻意镜像主线 `sound/soc/codecs/Makefile` 的命名，使产物为 `snd-soc-tlv320aic3x.ko` / `snd-soc-tlv320aic3x-i2c.ko`——与主线模块同名，udev/modprobe 行为一致。

```makefile
snd-soc-tlv320aic3x-objs := tlv320aic3x.o
snd-soc-tlv320aic3x-i2c-objs := tlv320aic3x-i2c.o
```

- Kbuild 语法：`<module-name>-objs` 列出构成该模块的所有目标文件。
- 核心模块由 `tlv320aic3x.o`（编译自 `tlv320aic3x.c`）构成。
- I2C 模块由 `tlv320aic3x-i2c.o`（编译自 `tlv320aic3x-i2c.c`）构成。
- 每个模块目前只有一个源文件，但若未来拆分（如把 PLL 逻辑独立），只需在此追加对象名。

```makefile
obj-m += snd-soc-tlv320aic3x.o
obj-m += snd-soc-tlv320aic3x-i2c.o
```

- `obj-m`：Kbuild 标准变量，声明要构建为**可加载模块**（`=m`）的目标。
- `+=`：追加而非覆盖，因为外部构建时 Kbuild 可能已预置 `obj-m`。
- 注意这里写的是 `.o` 而非 `.ko`——Kbuild 会自动把 `obj-m` 中的 `.o` 链接成对应的 `.ko`。

---

## 三、构建流程

```
dkms.conf 的 MAKE[0]
  │
  └─ make -C ${kernel_source_dir} M=${dkms_tree}/aic3x/6.1-rkr5.1/build modules
       │
       ├─ 内核 Kbuild 系统进入 M= 目录
       │    │
       │    └─ 读取本 Makefile
       │         │
       │         ├─ obj-m += snd-soc-tlv320aic3x.o
       │         │    └─ 依赖 snd-soc-tlv320aic3x-objs = tlv320aic3x.o
       │         │         └─ 编译 tlv320aic3x.c → tlv320aic3x.o
       │         │              └─ 链接 → snd-soc-tlv320aic3x.ko
       │         │
       │         └─ obj-m += snd-soc-tlv320aic3x-i2c.o
       │              └─ 依赖 snd-soc-tlv320aic3x-i2c-objs = tlv320aic3x-i2c.o
       │                   └─ 编译 tlv320aic3x-i2c.c → tlv320aic3x-i2c.o
       │                        └─ 链接 → snd-soc-tlv320aic3x-i2c.ko
       │
       └─ 产物复制到 /updates/dkms/（由 dkms.conf 的 DEST_MODULE_LOCATION 指定）
```

---

## 四、关键设计点

### 1. 为什么不直接写 `obj-m += tlv320aic3x.o`

- Kbuild 要求 `obj-m` 中的目标名与最终模块名一致。如果写 `obj-m += tlv320aic3x.o`，产物会是 `tlv320aic3x.ko`，与主线命名 `snd-soc-tlv320aic3x.ko` 不符。
- 通过 `<module>-objs` 桥接：`obj-m` 用模块名，`-objs` 用真实源文件对象名，实现"源文件名 ≠ 模块名"的解耦。

### 2. 无 `ccflags-y` / `EXTRA_CFLAGS`

- 本 Makefile 没有添加任何额外编译标志，说明驱动代码不依赖特殊的 include 路径或宏定义。
- 这降低了与不同内核版本构建的耦合风险。

### 3. 无 `all` / `clean` 目标

- 因为是通过 `make -C $kernel M=$dir modules` 调用，`all` 和 `clean` 由内核 Kbuild 系统提供，本文件无需定义。
- `dkms.conf` 的 `CLEAN` 变量同样调用 `make ... clean`，走内核的清理规则。

---

## 五、与其他文件的关联

| 关联点 | 说明 |
|---|---|
| `dkms.conf` | `MAKE[0]` 和 `CLEAN` 都通过 `make -C $kernel M=$dir` 间接执行本文件 |
| `tlv320aic3x.c` | 核心模块的唯一源文件，包含 `EXPORT_SYMBOL(aic3x_probe)` 供 I2C 模块调用 |
| `tlv320aic3x-i2c.c` | I2C 模块源文件，`#include "tlv320aic3x.h"` 并调用核心模块的 `aic3x_probe` |
| 模块加载顺序 | I2C 模块依赖核心模块导出的符号，`modprobe snd-soc-tlv320aic3x-i2c` 会自动加载核心模块 |

---

## 六、速查表

| 维度 | 内容 |
|---|---|
| 构建系统 | Linux Kbuild（外部模块） |
| 模块目标 1 | `snd-soc-tlv320aic3x.o` ← `tlv320aic3x.o` |
| 模块目标 2 | `snd-soc-tlv320aic3x-i2c.o` ← `tlv320aic3x-i2c.o` |
| 声明方式 | `obj-m +=`（追加） |
| 额外编译标志 | 无 |
| 自定义目标 | 无（全部走内核 Kbuild） |
| 命名策略 | 镜像主线 `sound/soc/codecs/Makefile` |
#（注：内容由AI生成）
