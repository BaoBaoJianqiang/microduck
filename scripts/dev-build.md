# dev-build.Dockerfile

## 文件位置

`d:\microduck\scripts\dev-build.Dockerfile`

## 核心设计决策

该 Dockerfile 为 `scripts/dev-push.sh --docker` 构建镜像：板端的用户态环境，并以 CI 相同的方式安装唯一的 C 依赖。

- **为什么存在**：默认路径用 `cargo zigbuild` 交叉编译，需要 `zig`、`cargo-zigbuild`，以及——因为 `padd` 链接 libudev 而 Mac 无法安装 aarch64 Linux 库——从板上拷贝的 `libudev.so.1`。每个都是"我想在机器人上试试"变成一下午工具链工作的途径。在此镜像中目标即主机：`apt-get install libudev-dev` 即可，无需交叉。
- **选择 Bookworm 而非 Trixie**：与 `scripts/board-test.sh` 相同的理由——针对比我们可能发布的任何用户态更旧的 glibc 构建。Bookworm 的 2.36 低于 Trixie 的 2.41，因此在此构建的二进制可加载到我们发布的板端和更旧镜像上，而在 Trixie 构建的会要求 2.41 并在更旧版本上以无原因的消息失败。这是 `.cargo/config.toml` 中 `.2.31` glibc 钉版本的容器等价物。
- **arm64 主机上为原生构建**：产出的守护进程是 aarch64 ELF，因为容器本身就是。`dev-push.sh` 传 `--platform linux/arm64` 以在 x86 笔记本上也保持此特性（代价是 qemu，脚本会提示）。
- **rust:1-bookworm 浮动到最新 1.x**：工作区声明自己的下限（`rust-version = "1.89"`），低于它的工具链会以该消息失败而非缺少方法，在此钉版本会是第二个需要同步 bump 的地方。

## 指令分析

| 指令 | 内容 | 说明 |
|---|---|---|
| `FROM` | `rust:1-bookworm` | 基于 Debian Bookworm 的 Rust 镜像，glibc 2.36，提供旧 glibc 下限 |
| `RUN` | `apt-get update && apt-get install -y --no-install-recommends libudev-dev pkg-config && rm -rf /var/lib/apt/lists/*` | 安装 libudev（供 `padd` 通过 gilrs 使用，是"到达板端的一切都是纯 Rust"的唯一例外）和 pkg-config（供 libudev-sys 查找）。`zstd` 的 C 代码由 crate 从源码编译，rust 镜像已带 C 编译器 |

## 关键要点总结

1. 镜像只安装 `libudev-dev` 和 `pkg-config`，无其他依赖。
2. 使用 `--no-install-recommends` 并清理 apt 列表，保持镜像精简。
3. 该镜像的核心价值是消除交叉编译的工具链复杂性，让本地构建+推送到板端的流程更简单。
4. glibc 版本选择是关键设计决策：构建环境 glibc 必须低于或等于目标板端 glibc。
5. 与 `systemd-test.Dockerfile`（用 Trixie）形成对比：后者运行板端所运行的，前者针对更旧 glibc 构建。
