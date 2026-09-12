# tests/install.rs 文件解析

## 文件位置

`d:\microduck\updater\tests\install.rs`

## 定位

`updaterd install` 引导路径测试，**运行真实 updaterd 二进制**（非调用 engine）。

为何在 `updater/tests/`：此包定义 binary，cargo 设置 `CARGO_BIN_EXE_updaterd` 并保证测试前重建。用其他方式推导路径曾导致 robotd 门控测试断言过期二进制却看似通过。

## 验证内容

子命令添加的接线：
- 覆盖 source（`--from`）
- 覆盖 `on_apply=none`
- 覆盖 `health=none`
- **且仅这些**

## 夹具 `FreshRobot`

从未更新的机器人：已发布的发布、可信密钥、空安装树、**生产** on_apply/health 配置。生产配置是故意的——仅能对抗 inert 配置的引导证明不了什么。

## 关键摘要

install.rs 测试 updaterd install 子命令：运行真实二进制验证引导路径正确覆盖 source/on_apply/health 且仅这些；在 updater/tests/ 保证 CARGO_BIN_EXE 重建二进制；用生产配置测试而非 inert 配置。
