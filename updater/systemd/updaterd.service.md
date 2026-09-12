# updaterd.service 文件解析

## 文件位置

`d:\microduck\updater\systemd\updaterd.service`

## 核心设计

两条必须保持的性质：
1. **无 robotd 依赖**——updaterd 是恢复路径，仅在机器人健康时运行会在客户最需要时不可用
2. **updaterd 永不在自身应用的更新重启集中**——重启自身会在最危险时刻杀执行者；它通过 systemd 瞬态 unit 在回复上线 5 秒后重启自身

## [Unit]

- `After=network-online.target` + `Wants=`（仅顺序，网络不可用是正常状态）

## [Service]

| 字段 | 值 | 说明 |
|---|---|---|
| `ExecStart` | `updaterd --config ... --socket ...` | 无 --robot-socket（在配置中，单一事实） |
| `User`/`Group` | root/robot | root 必需（交换发布树/跑钩子/systemctl）；`Group=robot` 使 socket 继承 robot 组，0660 才意为 robot 组 |
| `Restart` | on-failure, 10s | 崩溃重启但不紧循环 |
| `StateDirectory` | robot/updater | 引擎状态在发布目录外 |
| `RuntimeDirectory` | updaterd | 承载 identity.json |
| 硬化 | NoNewPrivileges/PrivateTmp/ProtectKernelTunables/... | 故意不 ProtectSystem=strict（要写 /opt/robot 与 systemctl） |

## 访问控制两层

1. socket 组（0660）— 谁可对话
2. allow_uids/allow_gids — 谁可变更（root 总可以）

## 关键摘要

updaterd.service 以 root:robot 运行 updaterd：root 权限用于交换发布/钩子/systemctl，robot 组使 socket 对组可读写；无 robotd 依赖；自身不在重启集；StateDirectory 保引擎状态不被交换销毁；硬化但不 ProtectSystem=strict。
