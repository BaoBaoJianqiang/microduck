# 解析：`audio/aic3x-dkms/Makefile`

## 这是什么

内核外部模块（out-of-tree）的构建规则文件。DKMS 执行 `dkms.conf` 里的 `MAKE[0]` 命令（`make -C <内核源码> M=<本目录> modules`）时，内核构建系统会读这个 Makefile，决定把哪些 `.c` 编译成哪些 `.ko` 模块。

## 全文逐段解析

```makefile
# Out-of-tree build of the TI TLV320AIC3x ASoC codec driver (core +
# I2C interface), mirroring sound/soc/codecs/Makefile in-tree naming so
# the modules land as snd-soc-tlv320aic3x[-i2c].ko. Built via DKMS —
# see dkms.conf and README.md in this directory.
```

头注释说明三件事：这是 TI TLV320AIC3x ASoC 编解码器驱动（核心 + I2C 接口）的树外构建；命名刻意对齐内核树内 `sound/soc/codecs/Makefile` 的写法，保证产物模块名与树内版本一致（`snd-soc-tlv320aic3x.ko` / `snd-soc-tlv320aic3x-i2c.ko`）；通过 DKMS 构建，细节见本目录 `dkms.conf` 与 README。

```makefile
snd-soc-tlv320aic3x-objs := tlv320aic3x.o
snd-soc-tlv320aic3x-i2c-objs := tlv320aic3x-i2c.o

obj-m += snd-soc-tlv320aic3x.o
obj-m += snd-soc-tlv320aic3x-i2c.o
```

四行构建逻辑，每两行一个模块：

- `snd-soc-tlv320aic3x-objs := tlv320aic3x.o`——模块 `snd-soc-tlv320aic3x` 由目标文件 `tlv320aic3x.o`（编译自 `tlv320aic3x.c`，驱动核心）组成。
- `snd-soc-tlv320aic3x-i2c-objs := tlv320aic3x-i2c.o`——模块 `snd-soc-tlv320aic3x-i2c` 由 `tlv320aic3x-i2c.o`（编译自 `tlv320aic3x-i2c.c`，I2C 探测胶水层）组成。
- 两条 `obj-m +=` 把两个模块声明为外部模块构建目标（`m` = 可加载模块）。

## 关键点

- **文件名 = 模块名**：`tlv320aic3x.c` → `snd-soc-tlv320aic3x.ko` 的映射完全靠这几行 `-objs` 赋值。内核模块名必须与设备树/驱动匹配逻辑里引用的名字一致，所以注释里强调"mirroring in-tree naming"。
- **同目录还有 `tlv320aic3x.h`**（驱动私有头文件，两个 `.c` 共用），它不产生模块，只参与编译。
- 这个 Makefile 不需要写 `ccflags` / `Kbuild` 之外的东西——`dkms.conf` 的 `MAKE[0]` 已经带着 `-C ${kernel_source_dir} M=...` 进来，剩下的事全由内核构建系统接管。
