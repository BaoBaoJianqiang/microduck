# board-test.sh

## 文件位置

`d:\microduck\scripts\board-test.sh`

## 核心设计决策

交叉编译板端二进制并在真实 ARM64 Linux 容器中运行验证。目标板端：Radxa Zero 3（RK3566, Cortex-A55 → aarch64）跑 Armbian 26.2.x，目标用户态 Debian 13 (Trixie)。

- **不是硬件的替代**，但能捕获所有只在开发机外出现的问题：交叉链接（特别是 zstd 的 C 代码）、glibc 底线、unix socket 和文件权限语义、任何静默依赖 macOS 的东西。
- **arm64 主机原生运行**容器（Apple Silicon Mac 或 CI runner 上很快）；x86_64 主机则每个进程都被模拟，shell 主导的检查占主导。
- **构建 glibc 底线极低（2.31）**：风险在构建主机而非目标，未 pin 的构建链接到 CI runner 恰好有的 glibc，某天超过板端的就会在板端加载失败且构建无提示。
- **只测目标用户态（Trixie）**：Armbian 提供其他镜像，但只发货 Debian 13，测无人运行的配置花 ~2x 时间防御不需要的主张。可通过 `BOARD_IMAGES` 覆盖。

## 常量/路径

| 名称 | 值 | 说明 |
|---|---|---|
| `TARGET_DIR` | `target/aarch64-unknown-linux-gnu/release` | 交叉编译输出 |
| `GLIBC_FLOOR` | `2.31` | glibc 底线 |
| `IMAGES` | `debian:trixie-slim` | 目标镜像（可 `BOARD_IMAGES` 覆盖） |
| `FIXTURE` | `target/board-fixture` | 假 release fixtures |
| `INSTALL_STAGED` | `target/board-install/staged` | 打包 staged 二进制 |
| `INSTALL_DIST` | `target/board-install/dist` | 打包产物 |
| `INSTALL_RELEASE` | `target/board-install/release` | 解包后的真实 release |

## 前置检查

- Docker daemon 可达（否则交叉构建成功但二进制无法运行，Docker 自己的错误像代码问题）。
- `cargo-zigbuild`、`zig`、`docker`；x86_64 主机还需 binfmt 注册 arm64。

## 构建与打包

### 交叉编译

```
cargo zigbuild --release --target aarch64-unknown-linux-gnu.2.31 --bins
```

设置 `PKG_CONFIG_ALLOW_CROSS=1` 和 `PKG_CONFIG_PATH=/usr/lib/aarch64-linux-gnu/pkgconfig`（否则 libudev-sys 拒绝或静默用主机库）。

### glibc 底线断言

`strings updaterd | grep GLIBC_2.x | sort -V | tail -1` 取最高 GLIBC 符号版本，断言不超过 `GLIBC_2.31`。

### 假 release fixtures

`test-support/examples/fake-release` 在宿主机上 mint（release 是签名清单+tarball，与目标架构无关，交叉编译产相同字节）。一次运行一个签名密钥：`1.0.0 1.1.0 1.2.0:tamper 2.0.0 3.0.0`。

### 真实 artifact 打包

`xtask package` 从刚交叉编译的二进制打包。两个列表从 `.github/workflows/_build-release.yml` 解析（不复制，避免第三份手维护列表漂移）：
- staged binaries：`grep 'release/[a-z]* staged/'`
- `--include` pairs：`grep -- '--include "..."'`

空列表会在此处显式失败（命名自己的原因），而非 200 行后报 "installed release has no systemd/updaterd.service"。

workspace version 从 `[workspace.package]` 段中解析（不用 `grep -m1 '^version'`，因为 `[workspace.metadata.gst-plugins]` 可能有 `version` 键）。

zstd 用 level 1（产物只用一次，默认 19 单线程太慢）。

## CHECKS：容器内验证

整个 CHECKS 是一个大字符串，对每个 IMAGES 运行。核心场景：

### updaterd 引擎路径

- **首次安装**：stage 1.0.0，`updaterd install --from published`，断言 `version=1.0.0`。
- **不健康 release 回滚**：`--inject-fault fail_health`，apply 1.1.0 后断言 rolled_back + 回到 1.0.0。
- **篡改 artifact 拒绝**：1.2.0:tamper，apply 退出码 5（REFUSED）。
- **socket 权限**：`srw-rw----` (0660)。
- **peer 凭据**：日志有 "mutating request"。
- **SIGPIPE**：`update log | head -1` 不 panic。
- **不可达 daemon**：exit 3。
- **掉电回滚**：`--inject-fault abort_after_swap` 后 swap 已发生，两次 `--check-only`（第一次记录 trial，第二次耗尽 MAX_BOOT_ATTEMPTS=2 回滚）。
- **正常 apply**：3.0.0 成功。

### 分层访问控制（updaterd）

- unit 的 `Group=robot` 使 socket 继承进程主组 → `root:robot`。
- Layer 1：组成员可读；非成员被 socket 模式阻止（exit 3）。
- Layer 2：组成员不能变更（exit 6 denied）。

### configd 授权

`--fake-net --fake-pads --allow-user member`：
- Layer 1：组成员可读 net status，非成员 exit 3。
- Layer 2 allow：named user 可 `system set-name`。
- Layer 2 deny：另一组成员 bystander 不能改（exit 6），且拒绝不生效。
- wifi 结果：错误 PSK exit 5（REFUSED），正确 PSK "connected to Pollen"。
- PIN 前导零保留（字符串存储，不转 int）。

### pad.* 表面

- fresh robot "none paired"。
- 未命名组成员配对被拒（exit 6）。
- named user 配对 Xbox 手柄。
- paired pad trusted（重启后自动重连，需 agent 批准）。
- forget 可重复（与 `net forget` 同契约）。
- **passphrase 不入日志**（NetConnectParams 手写 Debug 脱敏，这是检查其诚实性的地方）。

### btd 链接验证

`btd --version` 能跑即可（bluer 链接 libdbus，vendored source 由 zig cc 构建，这是本脚本存在的交叉链接失败类）。

### setup-board.sh 行为测试

用 stub（curl/find/systemctl 记录参数）和 fixture 断言脚本做了什么，而非 grep：
- 修正 `overlay_prefix=rk3568` + 启用 `uart2-m0`（RK3566 与 RK3568 共享 overlays，错误前缀可启动但无 /dev/ttyS2）。
- mask `serial-getty@ttyS2.service`（getty 读取端口吃掉舵机回复，所有电机看似缺失）。
- `console=display`（`console=both` 让 printk 走舵机线，间歇破坏回复）。
- **幂等性**：二次运行不追加 overlay。
- 蓝牙：
  - 无 flag 不动 Privacy 和 marker。
  - `--pause-btd-on-pair`：写 marker，不动 Privacy。
  - `--weird-ble`：设 `Privacy = device` + marker（644），隐含 pause。
  - 幂等：Privacy 只设一次。
  - 升级：已有 `Privacy = off` 被纠正为 `device`。
  - 重跑无 flag 不丢失已配置的 workaround。

### preinstall hook

从 `hooks/preinstall.in` 渲染（替换 `@ONNX_FLOOR@`/`@ONNX_TARGET@`），断言无残留占位符：
- runtime 在 floor 之上 → satisfies。
- runtime 太旧且 curl 失败（无法下载）→ 非零退出，命名版本和 fix（防止无法加载 policy 的 release 安装后 panic robotd 控制线程）。

### install.sh + hooks/postinstall（真实 artifact）

用 stub curl（服务 releases API 和 raw.githubusercontent，未知路径失败而非空文件）、stub systemctl（记录参数）。

**install.sh 断言**：
- 每个 release 的 unit 安装到 `/etc/systemd/system/`，字节一致，mode 644。
- 每个 unit 的 ExecStart 二进制（在 `/opt/robot/daemon/current/` 内的）可执行。
- accounts 在 units 之前（sysusers drop-ins + robot 组 + btd 用户）。
- login banner `/etc/update-motd.d/40-robot` 可执行且报告回滚。
- prompt snippet + bash completion 安装；实际驱动 PROMPT_COMMAND 验证 PS1 注入机器人名。
- `/usr/local/bin/robotctl` 符号链接通过 `current`（不钉到版本目录）。
- journald drop-in 安装 + 重启 journald。
- 顺序：`daemon-reload` 在所有 `enable --now` 之前；configd 在 btd 之前。
- boot recovery timer 用 `enable` 不用 `enable --now`（`OnBootSec=` 过截止立即触发，会在配置中途运行）。
- `DUCK_NO_START`：不 enable/start 任何 unit，反而 disable --now 已有运行的 unit，但仍安装 unit 到磁盘。
- operator config 文件（`/etc/robot/updater.toml`、`robotd.toml`）保留不覆盖。
- 幂等性：二次运行不改变文件集和 config 内容。
- 未知 unit：安装但不启用，且警告（release 是内容权威，但 install.sh 不知道启动顺序）。

**hooks/postinstall 断言**（在无 unit 的盒子上，cwd 为 release 目录）：
- 无 `[Install]` 的 unit：只安装不碰。
- timer：enable 不 start（下次启动武装）。
- 其他：`enable --now`。
- sysusers + daemon-reload。
- 恢复脚本到 `/usr/local/sbin`。
- login-shell 文件（prompt、completion、motd）。
- **不**放置：journald drop-in 和 robotctl symlink（只有 install.sh 做，配置时运行一次）。

## 关键要点总结

1. 检查通过行为（stub 记录 systemctl 参数）而非 grep，能捕获装错目录等问题。
2. release fixtures 无 systemd units（设计如此），真实 artifact 检查用 xtask package 的真实产物。
3. 容器内脚本整体单引号传参，内部不能有单引号（会提前结束脚本在主机运行）。
4. 每个 CHECKS 失败都有明确 `[FAIL]` 行和 exit 1。
5. 多镜像循环：`for image in $IMAGES; do docker run ... sh -c "$CHECKS"; done`。
