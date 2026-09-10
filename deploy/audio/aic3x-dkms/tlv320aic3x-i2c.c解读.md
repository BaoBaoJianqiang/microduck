# `tlv320aic3x-i2c.c` 解读

## 一、文件概览

| 属性 | 内容 |
|---|---|
| 文件类型 | Linux 内核模块源码（C） |
| 职责 | TLV320AIC3x codec 的 **I2C 总线适配层**——负责设备匹配、regmap 初始化，然后委托给核心驱动 |
| 作者 | Arun KS, Mistral Solutions（2008） |
| 基于 | `sound/soc/codecs/wm8731.c`（Richard Purdie） |
| 许可证 | GPL-2.0-only |
| 对应模块 | `snd-soc-tlv320aic3x-i2c.ko` |
| 依赖 | 核心模块 `snd-soc-tlv320aic3x.ko`（提供 `aic3x_probe` / `aic3x_remove` / `aic3x_regmap`） |

---

## 二、整体结构

```
头文件包含（linux/i2c.h, module.h, of.h, regmap.h, sound/soc.h）
  │
  ├─ aic3x_i2c_id[]        I2C 设备 ID 表（5 型号 + 哨兵）
  ├─ MODULE_DEVICE_TABLE(i2c, ...)   导出 I2C 匹配表
  │
  ├─ aic3x_i2c_probe()     探测：初始化 regmap → 调用核心 aic3x_probe()
  ├─ aic3x_i2c_remove()    移除：调用核心 aic3x_remove()
  │
  ├─ aic3x_of_id[]         设备树兼容字符串表（5 型号 + 哨兵）
  ├─ MODULE_DEVICE_TABLE(of, ...)    导出 OF 匹配表
  │
  ├─ aic3x_i2c_driver      i2c_driver 结构体（绑定上述函数与表）
  ├─ module_i2c_driver()   注册/注销 I2C 驱动的宏
  │
  └─ MODULE_DESCRIPTION / AUTHOR / LICENSE
```

---

## 三、关键代码解读

### 1. I2C 设备 ID 表

```c
static const struct i2c_device_id aic3x_i2c_id[] = {
    { "tlv320aic3x",   AIC3X_MODEL_3X },
    { "tlv320aic33",   AIC3X_MODEL_33 },
    { "tlv320aic3007", AIC3X_MODEL_3007 },
    { "tlv320aic3104", AIC3X_MODEL_3104 },
    { "tlv320aic3106", AIC3X_MODEL_3106 },
    { }
};
```

- 每个条目将 I2C 设备名映射到型号常量（定义在 `tlv320aic3x.h`）。
- `driver_data` 字段携带型号 ID，probe 时通过 `i2c_match_id()` 取回，传给核心驱动的 `aic3x_probe()`。
- 末尾 `{ }` 是哨兵，标记数组结束。
- **本项目使用的是 `tlv320aic3104`**（`AIC3X_MODEL_3104 = 3`）。

### 2. probe 函数

```c
static int aic3x_i2c_probe(struct i2c_client *i2c)
{
    struct regmap *regmap;
    struct regmap_config config;
    const struct i2c_device_id *id = i2c_match_id(aic3x_i2c_id, i2c);

    config = aic3x_regmap;       // 从核心模块复制 regmap 配置
    config.reg_bits = 8;         // I2C 寄存器地址：8 位
    config.val_bits = 8;         // I2C 寄存器值：8 位

    regmap = devm_regmap_init_i2c(i2c, &config);
    return aic3x_probe(&i2c->dev, regmap, id->driver_data);
}
```

**设计要点**：
- **regmap 配置从核心模块继承**（`aic3x_regmap`），只覆盖总线特有的 `reg_bits` / `val_bits`（均为 8）。这意味着寄存器缓存策略、默认值表、volatile 判定都由核心驱动统一维护，总线层不重复定义。
- `devm_regmap_init_i2c`：设备管理的 regmap 初始化，自动在设备释放时清理。
- **probe 本身极薄**：所有真正的 codec 初始化（电源、DAPM、控件、PLL）都在 `aic3x_probe()` 中完成。这是"核心与总线分离"架构的直接体现。
- 使用 `.probe_new`（无 `id` 参数的新版 probe），通过 `i2c_match_id()` 手动查表获取型号——兼容设备树匹配和传统 I2C 匹配两种路径。

### 3. remove 函数

```c
static void aic3x_i2c_remove(struct i2c_client *i2c)
{
    aic3x_remove(&i2c->dev);
}
```

- 同样极薄，直接委托核心驱动的 `aic3x_remove()`。
- 核心 remove 会从 `reset_list` 移除实例、释放 reset GPIO。

### 4. 设备树兼容表

```c
static const struct of_device_id aic3x_of_id[] = {
    { .compatible = "ti,tlv320aic3x", },
    { .compatible = "ti,tlv320aic33" },
    { .compatible = "ti,tlv320aic3007" },
    { .compatible = "ti,tlv320aic3104" },
    { .compatible = "ti,tlv320aic3106" },
    {},
};
```

- 设备树中 codec 节点的 `compatible` 属性必须匹配这些字符串之一。
- 本项目的设备树 overlay 使用 `ti,tlv320aic3104`。
- `MODULE_DEVICE_TABLE(of, ...)` 把该表导出到模块元数据，使 `modprobe` / udev 能根据设备树自动加载模块。

### 5. I2C 驱动注册

```c
static struct i2c_driver aic3x_i2c_driver = {
    .driver = {
        .name = "tlv320aic3x",
        .of_match_table = aic3x_of_id,
    },
    .probe_new = aic3x_i2c_probe,
    .remove    = aic3x_i2c_remove,
    .id_table  = aic3x_i2c_id,
};
module_i2c_driver(aic3x_i2c_driver);
```

- `.probe_new`：新版 I2C probe 签名（不带 `id` 参数），与 `i2c_match_id()` 配合。
- 同时提供 `.id_table`（传统 I2C 匹配）和 `.of_match_table`（设备树匹配），两条路径都支持。
- `module_i2c_driver()` 宏自动生成 `module_init` / `module_exit`，调用 `i2c_add_driver` / `i2c_del_driver`。

---

## 四、关键设计点

### 1. 核心与总线的严格分离

| 层 | 文件 | 职责 |
|---|---|---|
| 总线适配层 | `tlv320aic3x-i2c.c` | 设备匹配、regmap 初始化（8x8 bit）、委托核心 |
| 核心驱动层 | `tlv320aic3x.c` | 寄存器、DAPM、控件、PLL、电源、型号差异 |

- 核心驱动通过 `EXPORT_SYMBOL(aic3x_probe)` / `EXPORT_SYMBOL(aic3x_remove)` / `EXPORT_SYMBOL_GPL(aic3x_regmap)` 导出接口。
- 这种分离让核心驱动可复用于 SPI 或其他总线（只需写另一个适配层），也让总线层保持极简、不易出错。

### 2. reg_bits / val_bits 的覆盖

- 核心模块的 `aic3x_regmap` 定义了缓存策略、默认值、volatile 判定，但**不定义** `reg_bits` / `val_bits`（因为这些是总线相关的）。
- I2C 层在 probe 时复制配置并设置为 8/8——TLV320AIC3x 通过 I2C 访问时，寄存器地址和数据都是 8 位。

### 3. 双匹配表（I2C ID + OF）

- 传统板级文件用 `i2c_board_info` + `id_table` 匹配。
- 设备树用 `compatible` + `of_match_table` 匹配。
- 两者并存保证驱动在新旧两种平台上都能自动加载。

---

## 五、与系统其他部分的关联

| 关联点 | 说明 |
|---|---|
| `tlv320aic3x.h` | 提供型号常量（`AIC3X_MODEL_3104` 等）和 `aic3x_probe` / `aic3x_regmap` 声明 |
| `tlv320aic3x.c` | 核心驱动，导出 `aic3x_probe` / `aic3x_remove` / `aic3x_regmap` 供本文件调用 |
| 设备树 overlay | 节点 `compatible = "ti,tlv320aic3104"` 触发本驱动 probe |
| `Makefile` | 本文件编译为 `tlv320aic3x-i2c.o`，链接成 `snd-soc-tlv320aic3x-i2c.ko` |
| `dkms.conf` | 模块名 `snd-soc-tlv320aic3x-i2c` 与本驱动注册名一致 |
| `aic3104-init.sh` | 依赖本驱动 probe 成功后注册的 ALSA 声卡 `aic3104` |

---

## 六、速查表

| 维度 | 内容 |
|---|---|
| 总线 | I2C |
| 支持型号 | aic3x / aic33 / aic3007 / aic3104 / aic3106 |
| 本项目型号 | `tlv320aic3104`（AIC3X_MODEL_3104 = 3） |
| regmap 位宽 | reg=8, val=8 |
| probe 逻辑 | 初始化 regmap → 调用核心 `aic3x_probe()` |
| remove 逻辑 | 调用核心 `aic3x_remove()` |
| 匹配方式 | I2C id_table + 设备树 of_match_table 双路径 |
| 模块导出符号依赖 | `aic3x_probe`、`aic3x_remove`、`aic3x_regmap`（来自核心模块） |
| 作者 | Arun KS (Mistral Solutions, 2008) |
#（注：内容由AI生成）
