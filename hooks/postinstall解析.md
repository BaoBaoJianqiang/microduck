# 解析：`hooks/postinstall`

## 这是什么

**发布版的 systemd unit 安装钩子**（sh 脚本，153 行，约六成是注释）。它随发布版 artifact 携带，由更新器在**符号链接交换之后、`on_apply` 的重启之前**运行（见 `updater/src/hooks.rs`）——头注释第 7-9 行指出这是唯一能工作的窗口：`current` 已指向本发布，unit 的 `ExecStart` 能解析；随后的重启能找到已存在的 unit。

## 头注释确立的四条规则

### 1. 为什么存在（第 11-19 行）

unit 文件随**发布版 artifact 内部**携带，而此前只有 `scripts/install.sh` 把它们复制出去。于是一个引入新守护进程的发布装好了二进制，却把 unit 留在发布目录里——systemd 永远不看那里，`btd.service` 在一块发布完整且正确的板上以 `203/EXEC` 失败；更糟的是 `on_apply` 无法重启一个还不存在的 unit，首个携带守护进程的发布什么也重启不了。两次事故各花一个下午。教训沉淀为 [`docs/design/updater-design.md`](../docs/design/updater-design.md) §9.1 的规则（这是它的第四次实例）：**全新安装对板子做的一切，hook 也要做**——因为这是唯一在每个板上每次更新都会运行的东西。

### 2. 失败语义（第 21-25 行）

hook 非零退出**使更新失败并触发回滚**，所以本脚本的规则是：只对"意味着发布不可安装"的事情失败（exit 1），对其余一切只发警告。复制 unit 是第一种；服务拒绝启动是第二种——`btd` 在蓝牙适配器尚未出现的板上会合法地失败，"机器人因为缺无线电就无法更新"是一笔糟糕的交易。

### 3. 回滚时 unit 的残留（第 27-31 行）

回滚后，这里装的 unit 留在原地并指向 `current`——彼时它已是可能不含其对应二进制的旧发布，于是启动失败。**这是故意的**：替代方案是记录本发布新增了什么、回退时撤销；而"这个发布没装成、一个服务失败直到下一个发布成功"两种情况是同一个处境。它也是自愈的——下一个成功的更新会重装它携带的一切。

### 4. 非 template（第 5 行）

这里没有任何需要替换的东西，所以与 [`preinstall.in`](preinstall.in解析.md)（有 `@ONNX_FLOOR@`/`@ONNX_TARGET@` 替换）不同，原样发布。

## 逐段解析

### 设置（第 32-38 行）

```sh
set -eu
UNIT_DIR=/etc/systemd/system
SYSUSERS_DIR=/usr/lib/sysusers.d
SBIN_DIR=/usr/local/sbin
```

`set -eu`（未定义变量报错、命令失败即退）；hook 以**发布目录为工作目录**运行——后文所有 `scripts/`、`systemd/`、`bin/` 相对路径都基于此。

### 恢复脚本（第 40-49 行）

`robot-rescue` 与 `robot-boot-check` 复制到 `/usr/local/sbin`。与下面的 unit 同样的理由，**外加一条**：它们是本发布无法启动时运行的东西，所以绝不能经 `current` 读取。告警而非失败——无法安装它们的发布仍然值得拥有，且 `install.sh` 会在全新板上放置它们。

### 登录 shell 文件（第 51-63 行）

运行发布自带的 `scripts/setup-login.sh`：`robotctl` 补全、标注当前实际运行发布的 motd 横幅、提示符里的机器人名。`install.sh` 在全新板上运行同一脚本；**在这里运行它，是一块早于这些文件被写出的板子得到它们的唯一原因**——§9.1 的全部意义，其第四个实例正是本文件自身，曾在发布包含该功能的板子上缺席数月。永不致命，注释的原话："一个 shell 提示符不值得回滚一次更新。"

### 声音库（第 65-72 行）

```sh
bin/sounds ensure-bank --dir /var/lib/robot/sounds
```

由发布自带的生成器从 SoC 序列号渲染机器人的音库。幂等——marker 记录 seed 与音库版本，音库已是当前版本则是 no-op——所以每次安装都运行、只在第一次（或合成器音库版本升级时）实际渲染。告警而非失败："没有声音的机器人照样走路。"

### sysusers：用户与组先于 unit（第 74-90 行）

`[ -d systemd ] || exit 0`——发布没带 systemd 目录就直接退出。然后 `systemd/sysusers.d/*.conf` 安装到 `/usr/lib/sysusers.d/`（**此步失败 = exit 1**，属于第一种），再调 `systemd-sysusers`（存在才调）。为什么顺序重要（注释）：命名了不存在的 `User=` 的 unit 无法启动，且那个失败**读起来像守护进程坏了，而不是缺账号**。`systemd-sysusers` 本身失败仅告警——账号已存在是常见情况。

### unit 文件（第 92-106 行）

遍历 `systemd/*.service` 与 `systemd/*.timer`，`install -m 644` 到 `/etc/systemd/system/`，失败 = exit 1。**覆盖而非合并**，与 `install.sh` 行为一致：unit 文件属于发布版，受支持的定制方式是 `/etc/systemd/system/<unit>.d/` 下的 drop-in，本钩子不碰那里。

### 无 systemd / 无 unit 的收尾（第 106-115 行）

没有安装任何 unit 则直接退出；环境里没有 `systemctl`（容器、测试）则打印已安装的 unit 并以 0 退出——文件就位即是本钩子承诺的全部；`daemon-reload` 失败仅告警。

### enable --now 每个 unit，及两个例外（第 117-152 行）

`install.sh` 首装做同样的事，区别只在**时机**。enable 正是"新服务到达即可工作、而不是等到下一个手动步骤"的关键。每个失败都是告警：`on_apply` 稍后就会重启它自己列表里的 unit，只是慢的 unit 不是问题；真正起不来的 unit 绝不能让一次本可以成功的更新回滚。

两个例外，都关于**不要把本该在启动时运行的东西现在就跑起来**：

1. **没有 `[Install]` 段的 unit 跳过 enable**（第 137-140 行）——这样的 unit 根本无法 enable，恢复网的 oneshot（`robot-boot-check`）刻意就没有。对它执行 `enable --now` 会在正在安装它的更新中间跑一次回滚检查，而彼时守护进程正在合法地重启中——这正是回滚检查会读作"发布无法启动"的东西。
2. **timer 只 enable 不 start**（第 141-147 行）——带 `OnBootSec=` 的计时器若在超过期限后被 start 会立即触发，而本钩子运行在更新中间。它在下次启动时武装，"那正是它所关于的那次启动"。

### 结尾（第 152 行）

```sh
echo "postinstall: installed and enabled${units}"
```

## 在更新时序里的位置

`updaterd` 下载并验证 artifact → 解包 → **preinstall**（[`preinstall.in`](preinstall.in解析.md)，交换前，失败即中止）→ **符号链接交换** → **本钩子**（unit 就位、enable、恢复脚本与音库落位）→ `on_apply` 重启 unit → 30 秒健康门 → 达标则 `current` 定格，否则回滚。
