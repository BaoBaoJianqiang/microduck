# tests/rescue.rs 文件解析（xtask）

## 文件位置

`d:\microduck\xtask\tests\rescue.rs`

## 定位

测试**启动恢复网**的决策：`scripts/robot-boot-check` 和 `scripts/robot-rescue`，在临时树上运行。这是"其他一切都不行时必须工作"的代码，也是最难测试的代码——真实板上只在 release 无法启动时运行。

## 设计：可被问而不需板子

- rescue 的决策是**两个软链 + 一个面包屑**的纯函数
- check 的决策是 `systemctl show` 回答的纯函数
- `ROBOT_*` 环境变量移动树/状态目录/rescue 路径/uptime 源
- PATH 上的 stub `systemctl` 提供 unit 状态并记录是否要求 reboot

## 未覆盖（明说）

真实 systemd 行为：timer 是否在 `OnBootSec=180` 触发、oneshot 的 `Conflicts=shutdown.target`、crash-looping unit 的 `NRestarts` 读法——这些需要真实 systemd（`systemd-nspawn` 或特权容器）。

## 为什么在 xtask

这些是*仓库脚本*而非任何 crate 的行为——正是 xtask 测试目录其余部分的目的。用 `sh` 非 `bash`：板子上解释器是 `/bin/sh`，rescue 内的标志检测（GNU `mv -T` / BSD `mv -h`）正因为同一脚本须在两处工作。

## Board fixture

- `with_releases(versions)` — 创建 `releases/<v>` 目录 + `state/`
- `link(name, version)` — 相对目标软链（如 store 所写）
- `with_breadcrumb(from, to)` — 留此前 rescue 的面包屑
- `rescue(args)` / `boot_check(args, units)` — 运行脚本，units = `(unit, ActiveState, NRestarts)`
- stub `systemctl`：`show -p Prop --value unit` 按表回答，非 show 命令记录到 log（断言 reboot/无 reboot）

## robot-rescue 测试

| 场景 | 结果 |
|------|------|
| 无 golden 发布 | code 2，不动 current |
| golden 未安装（链悬空） | code 2，不动（换成空路径更糟） |
| current 已是 golden | code 2，不 reboot（硬件故障非 release 故障） |
| 正常 swap | code 0，current→golden，写面包屑 |
| 无 current | code 0，直接放 golden |
| 未请求 reboot | code 0，不 reboot，提示 `systemctl reboot` |
| `--reboot` | reboot |
| `--dry-run` | 说 would swap，什么都不改 |
| 未知参数 | code 1（拒绝，不静默忽略） |
| 面包屑仍在（updaterd 未启动过） | code 2 拒绝（防重启循环） |
| `--force` | 绕过面包屑守卫 |

## robot-boot-check 测试

| 场景 | 结果 |
|------|------|
| 所有 daemon active | "every daemon came up"，不动 |
| 某 daemon failed | swap + reboot + 面包屑记原因 |
| active 但 NRestarts≥5（crash-loop） | 仍 rescue（仅看状态会误判健康） |
| inactive 无重启（被人停了） | 不动 |
| release 不带的 unit（systemctl 答空） | 忽略 |
| 启动 4h 后调用 | code 2 "not a boot check"（防安装中 `enable --now` 触发） |
| `--dry-run` | 说 would hand over，什么都不做 |

## 面包屑契约

`at=` / `install_dir=`（updaterd 按 install_dir 匹配组件）/ `from=` / `to=` / `because=`。覆盖而非追加——一次一个尝试，需保留的历史进 update log。

## 关键摘要

rescue.rs 用 stub systemctl + 环境变量在临时树上测试启动恢复网：rescue 决策为软链+面包屑纯函数，check 决策为 systemctl 回答纯函数。覆盖：无/坏 golden、已是 golden、正常 swap、reboot 可选、dry-run、未知参数拒绝、面包屑防重启循环守卫+force 绕过、check 区分 failed/crash-loop/stopped/缺失/非启动时机。
