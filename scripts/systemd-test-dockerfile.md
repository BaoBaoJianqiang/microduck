# systemd-test.Dockerfile

## 文件位置

`d:\microduck\scripts\systemd-test.Dockerfile`

## 核心设计决策

该 Dockerfile 构建 `scripts/systemd-test.sh` 启动的镜像，是本仓库中唯一让 systemd 作为 pid 1 运行的地方。

- **用 Trixie 而非 Bookworm**：与 `dev-build.Dockerfile` 相反。此镜像运行板端所运行的，而非构建针对它的二进制，因此应使用我们发布的用户态（Trixie）而非构建时针对的更旧版本。
- **只装 systemd 和 dbus**：目的是一个有真实 cgroups 和真实瞬时定时器的真实 init，更新所需的其他一切——二进制、unit、hook——都由测试套件铸造的签名发布内部带入，与板端完全一致。
- **STOPSIGNAL SIGRTMIN+3**：systemd 自己的"干净停止"约定。没有它的话 `docker stop` 发送 SIGTERM，pid 1 systemd 会将其解读为 re-exec 请求而非关机。
- **CMD /lib/systemd/systemd**：Debian slim 镜像没有 `/sbin/init` 符号链接，测试套件显式传入此命令，此处仅记录原因。

## 指令分析

| 指令 | 内容 | 说明 |
|---|---|---|
| `FROM` | `debian:trixie-slim` | 基于 Debian Trixie slim，与板端用户态一致 |
| `RUN` | `apt-get update && apt-get install -y --no-install-recommends systemd dbus && rm -rf /var/lib/apt/lists/*` | 只安装 systemd 和 dbus，提供真实 init 环境 |
| `STOPSIGNAL` | `SIGRTMIN+3` | 告诉 Docker 用 SIGRTMIN+3 停止容器，systemd 将其解读为干净关机 |
| `CMD` | `["/lib/systemd/systemd"]` | 以 systemd 作为 pid 1 启动 |

## 关键要点总结

1. 这是本仓库唯一以 systemd 为 pid 1 的容器环境。
2. 极简设计：只有 systemd + dbus，其他测试内容通过签名发布注入。
3. `STOPSIGNAL SIGRTMIN+3` 是容器中运行 systemd 的关键配置，否则无法正常停止。
4. 与 `dev-build.Dockerfile` 的 glibc 策略相反：一个用 Trixie（运行时），一个用 Bookworm（构建时旧 glibc 下限）。
5. 此镜像配合 `scripts/systemd-test.sh` 运行真实的 systemd 更新测试（需要 `--privileged`，不在 CI 中运行）。
