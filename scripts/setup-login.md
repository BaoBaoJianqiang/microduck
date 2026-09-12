# setup-login.sh

## 文件位置

`d:\microduck\scripts\setup-login.sh`

## 核心设计决策

该脚本安装登录 shell 相关文件：`robotctl` 补全、显示当前运行版本的 banner、以及提示符中的机器人名称。

- **为什么是独立脚本而非 install.sh 中的函数**：`install.sh` 只运行一次，用于初始化板端后不再出现，更新路径无法触及它（不在制品中）。此前这三项只加在 `install.sh` 中，所以只做过更新的板端都没有这些功能。现在 `install.sh` 和 `hooks/postinstall` 都运行此脚本，与 `setup-gstreamer.sh` 的模式一致。
- **幂等，每次更新都运行**：每步重写自己的文件，使两个月前初始化的板端最终拥有今天初始化的板端所拥有的。
- **失败被视为装饰性**：`hooks/postinstall` 将失败视为 cosmetic，因为 shell 提示符不值得回滚更新。机器人运行不需要这些。

## 函数分析

### install_login_banner

写入 `/etc/update-motd.d/40-robot`，在每次 ssh 登录时显示：
- 当前运行的版本（`readlink /opt/robot/daemon/current`）
- `robotctl health` 的整体判定（第一行）
- 若上次更新被回滚，提示 `ROLLED BACK`

设计要点：
- 用 motd drop-in 而非 `/etc/profile.d`：每次 ssh 登录运行一次而非每个 shell，且慢或坏的 robotctl 不会卡住交互 shell。
- 所有路径 exit 0，banner 失败绝不能导致登录嘈杂或缓慢。
- 背景：一个 dev 板静默回滚到稳定版（updaterd 的跨启动健康门控，因为无舵机电源的实验板永远无法报告健康），后续命令都运行在非预期代码上。

### install_name_prompt

写入 `/etc/profile.d/robot-name-prompt.sh`，在提示符中显示机器人名称，如 `microduck@radxa-zero3 (coincoin):~$`。

设计要点：
- 从同一镜像刷出的板端提示符都一样，三个 ssh 窗口无法区分。
- 用 profile.d 片段而非编辑 `.bashrc`：无需逐用户配置，更新时干净替换。
- 排序问题：`~/.bashrc` 在 `/etc/profile.d` 之后设置 PS1，所以片段不能直接改 PS1，而是 hook `PROMPT_COMMAND` 在第一次提示符时注入名称。
- 名称来源：configd 的 `config.json` 中的 `name` 字段，或从 SoC 序列号 SHA-256 前 4 位十六进制派生的 `duck-xxxx`（与 configd 派生方式一致）。
- 安全：名称会进入提示符展开（`$(...)` 和反引号会执行），configd 只剥离控制字符，此处额外剥离 `\$` 和反引号。

### install_completions

写入 `/etc/bash_completion.d/robotctl`，为 `robotctl` 提供 tab 补全。

设计要点：
- 用 loader 而非生成的快照：在 shell 启动时向活的二进制请求补全，更新添加子命令后补全自动描述当前版本，无需重装，回滚也不会留下残留。
- 代价是每个交互 bash 一个进程，与 `bash-completion` 自身对动态补全器的做法相同。
- 用 `eval` 而非 `source <(robotctl ...)`：进程替换在 bash 3.2 不可靠；丢弃 stderr 使早于 `robotctl completions` 的回滚版本不会导致每次登录报错。

## 关键要点总结

1. 脚本按顺序调用 `install_completions`、`install_login_banner`、`install_name_prompt`。
2. 每个函数在对应目录不存在时直接 return 0，不创建登录机制。
3. 文件权限：banner 755（可执行），prompt 和补全 644。
4. 核心设计原则：所有内容都是装饰性的，失败不影响更新；每次更新重写以保证一致性。
