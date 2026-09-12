# source/local.rs 文件解析

## 文件位置

`d:\microduck\updater\src\source\local.rs`

## 定位

本地目录源。两个用途：
1. CI 测试用真实引擎代码路径无网络（防测试漂移）
2. 开发旁加载流（本地构建+dev 密钥签名的工件不经 prod 签名）

## 签名验证不放松

本地工件与下载的完全一样验证；旁加载能工作是因为 dev 密钥在信任集合中，**非**因为跳过检查。

## 目录布局

```
<version>.manifest.json
<version>.manifest.json.minisig
<manifest.url 命名的文件>
<该文件>.minisig
```

## 版本枚举

`versions()` 扫描目录，取 `.manifest.json` 后缀前能解析为 semver 的名称，新→旧排序。非版本名称（工件、签名）忽略而非错误。

## 关键摘要

local.rs 是测试与旁加载源：目录含 `<version>.manifest.json` + 工件 + 各自签名；签名验证完全不放松（dev 密钥在信任集）；版本从 manifest 文件名枚举。
