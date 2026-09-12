# hooks.rs 文件解析

## 文件位置

`d:\microduck\updater\src\hooks.rs`

## 定位

pre/post 安装钩子。钩子**随签名工件发布**，所以无未签名代码运行。类似 dpkg 的 `postinst`。

## 顺序

```
extract → [pre_install] → symlink swap → [post_install] → apply → health gate
```

非零退出=更新失败=触发回滚，与健康探针失败相同。钩子是门的一部分，不是 fire-and-forget。

## 钩子文件

- `PRE_INSTALL = "hooks/preinstall"` — 交换**前**，旧版本仍活（可装 ONNX/GStreamer 等发布需要但不能含的东西）
- `POST_INSTALL = "hooks/postinstall"` — 交换后

## `HookContext`

通过环境变量传给钩子：`UPDATE_COMPONENT`/`UPDATE_CHANNEL`、`UPDATE_OLD_VERSION`、`UPDATE_NEW_VERSION`、`UPDATE_INSTALL_DIR`、`UPDATE_RELEASE_DIR`、`UPDATE_OLD_SCHEMA_VERSION`、`UPDATE_NEW_SCHEMA_VERSION`。缺失值省略而非设空，使钩子能区分"首次安装"与"未知"。

## `run(release_dir, kind, ctx, timeout) -> Result<HookOutcome, Error>`

- 钩子不存在→成功（`ran: false`）
- 存在但失败/超时→错误（挂起的钩子不能无限卡住 updater）
- 输出截断到 8 KB，防啰嗦钩子撑爆更新日志

## 关键摘要

hooks.rs 运行随工件发布的 pre/post 安装钩子：pre 在交换前（可装系统依赖，旧版本仍活），post 在交换后；非零退出或超时触发回滚；上下文通过 UPDATE_* 环境变量传递；输出截断防日志爆炸。
