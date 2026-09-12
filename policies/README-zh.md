# 策略

`robotd` 运行的 ONNX 策略。全部都是 `obs[1,61] -> actions[1,14]`；`robotd` 在加载时检查这一点，而不是在迈步途中才发现。

## 这是一个临时的家

**它们本应在 Hugging Face Hub 上**，作为一个独立于守护进程进行版本化的 `model` 更新组件交付——一次步态重训不应该需要一次守护进程发布，一个守护进程修复也不应该重新下载 6 MB 未变动的权重。`deploy/updater.toml` 已经描述了那个组件，并刻意保持未配置，直到那些仓库存在。

它们被 vendor 在这里，是因为它们当时还不在 Hub 上，而守护进程离开它们无法运转。提交它们使发布版自包含，这正是让更新路径可以端到端测试的那个性质：一条 `robotctl update apply` 就把一台站立的机器人变成一台行走的机器人。

日后删除本目录就是整个迁移：把 `deploy/robotd.toml` 里的 `[policy]` 路径指向 model 组件安装到的位置，并从 `.github/workflows/dev.yml`、`.github/workflows/_build-release.yml` 与 `scripts/dev-push.sh` 中删掉 `--include` 行——三处都要删，且在那之前，`xtask` 的 `every_policy_in_the_repo_is_packaged` 测试就是让这三份列表保持诚实的东西。

## 来源

复制自 `apirrone/microduck_runtime` 的 commit `5f3b314`（`roulade.onnx` 取自 `7e4ab6d`，它首次出现之处），并解引用了那个仓库用来给特定训练运行赋予稳定名字的符号链接：

| 这里 | 那边 | 角色 |
| --- | --- | --- |
| `alpha_walking.onnx` | `BEST_alpha_walking_rough.onnx` | 行走 / velstand |
| `alpha_stand.onnx` | `BEST_alpha_stand_body_control.onnx` | 站立 + 身体姿态 |
| `alpha_sitstand.onnx` | `BEST_alpha_sitstand.onnx` | 坐 ↔ 站（姿态标志） |
| `alpha_ground_pick.onnx` | `alpha_ground_pick.onnx` | 地面拾取（阶段命令） |
| `ball_kick_left.onnx` | `ball_kick_left.onnx` | 左腿踢 |
| `ball_kick_right.onnx` | `ball_kick_right.onnx` | 右腿踢 |
| `roller.onnx` | `BEST_roller.onnx` | 轮式模式的运动 |
| `roller_crouch.onnx` | `BEST_roller_crounch.onnx` | 轮式模式的下蹲（ground-pick 槽位） |
| `roulade.onnx` | `roulade.onnx` | 前滚（Mjlab-Roulade-MicroDuck） |

（`roller_crouch` 同时修正了上游文件名里的拼写错误。）

这里的名字是*角色*——`deploy/robotd.toml` 所要求的——而不是训练运行。这层间接是刻意的，值得保留：更换哪次运行是"行走策略"，不应该意味着在每台机器人上改配置。

## 为什么只要 61 维家族

原型机还带有一个 51 维家族（`3 gyro + 3 gravity + 42 joints + 3 command`，遗留的 `[vx, vy, vtheta]` 命令）。61 维是相同的传感器加上本守护进程构建的统一 13 值命令（`[vel(3), head(4), body(6)]`）——它构建的唯一观测。一个 51 维文件在加载时失败，报出一行同时点名两个宽度的精确信息：

```
policy unavailable: .../walking.onnx: observation width is 51, expected 61
```

那个形状检查赢得了自己的位置：它把一个装错策略的错误变成了一次诊断，而不是一台以没人能解释的方式移动的机器人。

## 尝试你自己的

不需要发布版——`deploy/robotd.toml` 接受绝对路径：

```toml
[policy]
walk  = "/home/radxa/my_walk.onnx"
stand = "/home/radxa/my_stand.onnx"
```

然后 `sudo systemctl restart robotd`。一个加载失败的策略会通过 `robot.health` 报告为 `policy unavailable: <reason>`，同时控制环继续跳动并保持姿态，所以一个坏文件是可见的，而无需把机器人放到地板上。
