# updater.example.toml 文件解析

## 文件位置

`d:\microduck\updater\updater.example.toml`

## 定位

`/etc/robot/updater.toml` 的注释参考。**非发货文件**——`deploy/updater.toml` 才是 install.sh 安装的；此文件文档化每个选项，包括发货配置中刻意省略的。

## 顶层配置

| 字段 | 示例值 | 说明 |
|---|---|---|
| `trusted_keys_dir` | `/etc/robot/trusted_keys` | minisign 公钥目录，空目录致命 |
| `hw_rev` | 1 | 单一前向兼容守卫 |
| `state_dir` | `/var/lib/robot/updater` | 必须在 install_dir 外 |
| `robot_socket` | `/run/robotd.sock` | robotd 监听处 |
| `allow_dev_keys` | false | 永不客户端机器人 |
| `allow_fault_injection` | false | 永不客户端机器人 |
| `check_interval` | `"6h"` | 轮询间隔 |
| `auto_apply` | `"mandatory"` | 无人值守策略 |

## `auto_apply` 三值

- `off` — 从不
- `mandatory`（默认）— 仅 min_supported 地板以下
- `all` — 所有版本（canary/bench）

## 组件示例

### daemon 组件

- `install_dir = /opt/robot/daemon`
- `keep_previous = 1`
- `golden = "1.0.0"`
- `source = github_releases`，`tag_prefix = "daemon-v"`
- `on_apply = restart ["robotd"]` — **不含 updaterd/btd**（前者会杀执行者，后者会丢 app 连接）
- `health = socket 30s`

### 模型组件（model-walk, model-jump）

- `source = hf_hub`
- `on_apply = reload`（原位信号，不中断电机控制）
- `health = none`

### 本地源（dev/CI）

注释示例，非生产。

## 关键摘要

updater.example.toml 是配置的注释参考：文档化所有选项；daemon 用 github_releases+restart robotd+socket 健康门；模型用 hf_hub+reload；updaterd/btd 不在重启集；auto_apply 默认 mandatory。
