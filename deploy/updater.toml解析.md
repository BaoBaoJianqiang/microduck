# 解析：`updater.toml`

## 这是什么

**出厂客户机器人随附的更新器配置**（170 行，大半为注释）。由 `scripts/install.sh` 装到 `/etc/robot/updater.toml`。文件头注释强调它和 `updater/updater.example.toml` 的分工：example 是逐项注解的参考（含刻意不设的项），**本文件展示的是"一台出货机器人上真实成立的东西"**——每一行要么是一个决定、要么是一个事实，example 的注释解释选项是什么，本文件的注释解释为什么这么选。

## 顶层配置逐项解析

| 配置 | 值 | 作用与理由 |
|---|---|---|
| `trusted_keys_dir` | `/etc/robot/trusted_keys` | 信任锚点。三把发布公钥在安装时全部放入，尽管如今只有 `release-1` 在签名——机器人只能对烧进它里的密钥集做验证，预先放入备用密钥是让密钥轮换不必重刷的唯一机会 |
| `hw_rev` | `1` | 硬件版本号 |
| `state_dir` | `/var/lib/robot/updater` | 刻意放在所有 `install_dir` 之外：更新状态的记录要能幸存于它所记录的那次替换/回滚（§5.7） |
| `robot_socket` | `/run/robotd.sock` | 更新器向 robotd 询问健康/安全性的套接字 |
| `allow_dev_keys` | `false` | 拒绝团队开发密钥签名的构建。开发板在本地覆盖这两项；更新不会碰 `/etc`，所以覆盖不会被更新冲掉 |
| `allow_fault_injection` | `false` | 拒绝"故意让更新器失败"的注入指令 |
| `check_interval` | `6h` | 周期性检查更新的间隔。没有它，机器人只有打开 app 才知道有版本下限，"我们发了个坏版本"就救不了任何主人不在场的机器人（§8.1） |
| `auto_apply` | `"mandatory"` | 周期检查允许无人值守安装什么，取值 `off`/`mandatory`/`all`，见下 |
| `allow_users` | `["btd"]` | 谁可以 apply/rollback/select/pin，见下 |

### `auto_apply = "mandatory"`

- **`mandatory`（本文件的选择）**：只有清单里 `min_supported` 高于当前运行版本的发布可以无人值守安装——否则整个舰队会卡在一个我们已撤回的版本上。普通发布仍等客户端发起：机器人何时重启是主人的决定，app 驱动的更新流程就是为给出这个决定而存在。
- **`all`** 是 canary 与台架机器人（§16.2 Tier 2）用的；写在这里等于让所有客户机器人接受无人值守重启。
- 无论设哪个，无人值守 apply 就是一次普通 apply：同样的预检（正在行走/推流的机器人拒绝并等下个周期）、同样的健康门（起不来的发布被回滚）。

### `allow_users = ["btd"]` 的三条纪律（文件注释着墨最多的一节）

1. **只读调用永不受此门控**（status/log/listInstalled/check/subscribe）：到达套接字本身已要求属组（mode 0660），支持人员必须能查看一台自己无权改动的机器人。
2. **按名字不按 uid**：systemd-sysusers 动态分配 uid，写数字在一块板上对、另一块板上错。`allow_uids`/`allow_gids` 仍存在但仅作台架覆盖，且有测试断言出厂配置不用它们。
3. **绝不写 `allow_groups = ["robot"]`**：`robot` 组的成员身份只该让进程"够得着" updaterd（读状态），再授予变更权就把两层并成一层——能读状态就能换固件。这同样有测试把关。

## `[component.daemon]` 段逐项解析

| 配置 | 值 | 作用与理由 |
|---|---|---|
| `install_dir` | `/opt/robot/daemon` | 发布版安装根目录 |
| `keep_previous` | `1` | 保留上一个版本用于回滚 |
| `golden` | （注释掉） | **刻意不设，直到 1.0.0 出现**。它指向一个永不修剪的已知良好版本，是 `robotctl update reset-to-golden` 防变砖链的最后一环（§8.2）；指向一个机器人从未装过的版本会让这条命令在最需要的时刻失败。随 1.0.0 的同一变更设置 |
| `source.type` | `github_releases` | 稳定通道的来源 |
| `source.repo` | `ORG/duck-daemon` | 仓库 |
| `source.tag_prefix` | `daemon-v` | 只认 `daemon-v*` tag；staging 发的是 `daemon-staging-v*` 且标记 prerelease，双重排除，客户机器人不会漂到候选版上 |
| `ref_tag_prefix` | （注释，默认 `daemon-dev-`） | 分支 dev 构建 tag 前缀，`--ref my-branch` 解析为 `daemon-dev-my-branch`。与 `tag_prefix` 分开是因为两条流绝不能混淆：dev tag 会*移动*，发布 tag 不可变，且装 dev 构建还需 `allow_dev_keys` + dev 公钥双条件 |
| `staging_tag_prefix` | （注释，默认 `daemon-staging-v`） | `--staging` 找的 tag。到达候选版仍需有人带 root 手敲 `--staging`；候选版标记为 prerelease，`latest` 跳过它们 |

### `[component.daemon.on_apply]`

```toml
action = "restart"
units  = ["robotd", "configd"]
```

- `updaterd` 既不重启自己也不重启 `btd`：重启自己会杀掉更新执行者，重启 `btd` 会掐断正在向 app 汇报进度的连接。两者在下一次启动时拾起新二进制（更新应答 5 秒后有补偿机制，见 `restart-order.md`）。
- `mediad` 无需条目：重启集从发布版实际携带的 unit 推导，`mediad.service` 有 `[Install]` 段，会像其他守护进程一样被重启。
- **这份列表是增量的、不是权威的**：既然重启集由发布版携带的 unit 推导，这里只需点名发布版*不*携带的 unit。因此下面两条其实冗余，保留它们是因为"写明预期"的配置比一个默默表示"自己琢磨"的空列表好读。它曾经是权威的——那正是被修复的 bug：`install.sh` 保留运维者的文件，`configd` 出现之前部署的板子永远只写 `units = ["robotd"]`，之后每个发布都替换了 configd 的二进制却留着旧进程（见 `docs/project/install-path-gap.md` §4）。

### `[component.daemon.health]`

```toml
probe   = "socket"
timeout = "30s"
```

- 让自动回滚成立的门。30s 是占位值，M4 要换成实测启动时间加余量；太短会把只是启动慢的健康发布回滚掉。
- **门对 unhealthy 回滚、对 degraded 放行**：看不见舵机的机器人报 degraded 且通过——它换版本前也是这么说的，回滚修不好它；真坏了控制环路的发布仍会回滚。

## `[component.daemon]` 里刻意缺席的 `models` 段

不是疏漏，也不是"照抄 example 就能修"的问题：example 里的 `model-walk`/`model-jump` 指向尚不存在的 HF 仓库。一个源 404 的组件会让每次 `check`（包括上面的周期检查）都为一个没人发布过的组件报失败，训练读状态的人忽略失败。每个模型组件随首次发布其 bundle 的那个变更一起加入。
