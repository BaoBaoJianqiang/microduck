# 解析：`audio/aic3x-dkms/tlv320aic3x-i2c.c`

## 这是什么

TLV320AIC3x 编解码器驱动的 **I2C 总线接口层**（73 行，含许可证头注释）。核心驱动逻辑全部在 `tlv320aic3x.c`（约 1888 行）里，本文件只负责一件事：作为内核 I2C 设备驱动的入口，把 I2C 总线上探测到的芯片接到核心驱动上。上游出处：作者 Arun KS（Mistral Solutions, 2008），基于 `sound/soc/codecs/wm8731.c`，许可证 GPL-2.0-only。

## 逐段解析

### 设备 ID 表（第 20-28 行）

```c
static const struct i2c_device_id aic3x_i2c_id[] = {
	{ "tlv320aic3x", AIC3X_MODEL_3X },
	{ "tlv320aic33", AIC3X_MODEL_33 },
	{ "tlv320aic3007", AIC3X_MODEL_3007 },
	{ "tlv320aic3104", AIC3X_MODEL_3104 },
	{ "tlv320aic3106", AIC3X_MODEL_3106 },
	{ }
};
MODULE_DEVICE_TABLE(i2c, aic3x_i2c_id);
```

一张 I2C 设备 ID 表覆盖 AIC3x 家族五个型号。每行第二列是传给核心驱动的 `driver_data`（型号枚举，定义在 `tlv320aic3x.h`），核心驱动据此启用/裁剪对应型号的功能。本项目实际用到的是 `tlv320aic3104`（`AIC3X_MODEL_3104`）。`MODULE_DEVICE_TABLE` 让模块加载时自动向 udev 暴露可支持设备。

### I2C probe / remove（第 30-47 行）

```c
static int aic3x_i2c_probe(struct i2c_client *i2c)
{
	struct regmap *regmap;
	struct regmap_config config;
	const struct i2c_device_id *id = i2c_match_id(aic3x_i2c_id, i2c);

	config = aic3x_regmap;
	config.reg_bits = 8;
	config.val_bits = 8;

	regmap = devm_regmap_init_i2c(i2c, &config);
	return aic3x_probe(&i2c->dev, regmap, id->driver_data);
}

static void aic3x_i2c_remove(struct i2c_client *i2c)
{
	aic3x_remove(&i2c->dev);
}
```

- 取核心驱动导出的通用 `aic3x_regmap` 配置模板，补上 I2C 特有的**寄存器 8 位、值 8 位**寻址参数；
- `devm_regmap_init_i2c` 创建挂在这个 I2C 客户端上的 regmap 实例（后续所有寄存器读写都经它走 I2C）；
- 调用核心驱动的 `aic3x_probe()`（由 `tlv320aic3x.c` 导出），把型号作为 `driver_data` 传入；
- `remove` 对称地只调 `aic3x_remove()`。

这种"总线层极薄、核心层共享"的结构意味着将来若需要 SPI 接口，只需再写一个类似的胶水文件。

### 设备树匹配表（第 49-57 行）

```c
static const struct of_device_id aic3x_of_id[] = {
	{ .compatible = "ti,tlv320aic3x", },
	{ .compatible = "ti,tlv320aic33" },
	{ .compatible = "ti,tlv320aic3007" },
	{ .compatible = "ti,tlv320aic3104" },
	{ .compatible = "ti,tlv320aic3106" },
	{},
};
MODULE_DEVICE_TABLE(of, aic3x_of_id);
```

五个 `compatible` 字符串与设备树 overlay 里声明的编解码器节点对应。本项目 Radxa Zero 3W 的 overlay 中节点 compatible 即 `ti,tlv320aic3104`。

### 驱动注册（第 59-73 行）

```c
static struct i2c_driver aic3x_i2c_driver = {
	.driver = {
		.name = "tlv320aic3x",
		.of_match_table = aic3x_of_id,
	},
	.probe_new = aic3x_i2c_probe,
	.remove = aic3x_i2c_remove,
	.id_table = aic3x_i2c_id,
};

module_i2c_driver(aic3x_i2c_driver);

MODULE_DESCRIPTION("ASoC TLV320AIC3x codec driver I2C");
MODULE_AUTHOR("Arun KS <arunks@mistralsolutions.com>");
MODULE_LICENSE("GPL");
```

`module_i2c_driver` 宏展开为标准的模块 init/exit（注册 + 注销 `i2c_driver`）。`probe_new` 是新式 probe 回调（不再接收旧的 `i2c_device_id` 参数，需要时在函数内用 `i2c_match_id` 自取）。

## 在整个部署里的位置

`setup-board.sh` 的音频节通过 DKMS 安装本目录两个模块 → 板子启动时 I2C 核心（`rk3x-i2c`）枚举到编解码器 → 本文件匹配 `ti,tlv320aic3104` → 调核心模块 `aic3x_probe()` 完成注册 → ALSA 出现 `aic3104` 声卡 → [`aic3104-init.sh`](../aic3104-init.sh解析.md) 在其上设置混音电平与麦克风路由。
