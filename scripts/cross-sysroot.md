# cross-sysroot.sh

## 文件位置

`d:\microduck\scripts\cross-sysroot.sh`

## 核心设计决策

该脚本构建 `mediad` 交叉编译所针对的 aarch64 sysroot。

- **为什么需要**：`cargo board` 链接一个 C 依赖（libudev，供 `padd` 中的 gilrs 使用），GStreamer 是第二个且大得多——`gstreamer-rs` crate 是 pkg-config crate，交叉编译需要目标机的头文件、`.pc` 文件和共享库。
- **用机器人自己的包，而非 Ubuntu multiarch**：CI 对 libudev 走 Ubuntu multiarch + ports.ubuntu.com，但这里不能走那条路——那会链接 Ubuntu 的 GStreamer，而机器人运行的是 Debian trixie 的。此脚本解包机器人自己的包，编译器看到的就是板上有的，版本一致。
- **显式包列表，不解析 Depends 闭包**：从这些根包遍历 Debian Depends 会拉入 543 个包——`libgstreamer-plugins-bad1.0-dev` 声明了每个可选后端的 dev 包，闭包会到达 Qt、Vulkan 和 OpenEXR。编译一个 `.pc` 文件不需要这些。
- **`PKG_CONFIG_LIBDIR` 替换而非追加**：`PKG_CONFIG_PATH` 会追加到主机的，pkg-config 会用主机库回答，产生无法在机器人上运行的二进制。因此替换是正确选择，但 sysroot 必须服务整个工作区而非单个 crate——所以 libudev 也在此。
- **只下载解包，不安装任何东西**：所有内容在一个可删除的目录下。

## 常量/类型/函数分析

### 关键变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `SYSROOT` | `${DUCK_SYSROOT:-${TMPDIR:-/tmp}/duck-aarch64-sysroot}` | sysroot 目录，可通过 `DUCK_SYSROOT` 覆盖 |
| `MIRROR` | `http://deb.debian.org/debian` | Debian 镜像源 |
| `SUITE` | `trixie` | Debian 13，与板端 Armbian 用户态一致 |
| `ARCH` | `arm64` | 目标架构 |
| `TRIPLE` | `aarch64-linux-gnu` | 目标三元组 |

### MODULES（pkg-config 模块）

```
gstreamer-1.0 gstreamer-app-1.0 gstreamer-video-1.0 gstreamer-audio-1.0
gstreamer-webrtc-1.0 gstreamer-sdp-1.0 libudev
```

`--check` 精确验证这些模块，过期的包列表会在此失败而非构建内部。`gstreamer-webrtc-1.0` 和 `gstreamer-sdp-1.0` 来自 plugins-bad。

### PACKAGES（显式包列表）

包含 GStreamer、GLib、libudev 的 dev 和 runtime 包。注意：
- `-dev` 包只提供 `libfoo.so` 符号链接，实际链接需要 runtime 包的 `libfoo.so.N`，所以 GStreamer/GLib/udev 的 runtime 包都在此。
- `libglib2.0-0t64`（不是 `libglib2.0-0`）：trixie 的 64 位 `time_t` 迁移重命名了它。
- `libgio-2.0-dev`（不是 `libglib2.0-dev`）：后者是 55 KiB 元包，GLib 头文件和 `.pc` 已移出。

### 函数

| 函数 | 功能 |
|---|---|
| `check_tools` | 检查 curl、ar、tar、awk、pkg-config 是否存在 |
| `print_env` | 打印交叉构建所需的环境变量（PKG_CONFIG_SYSROOT_DIR、PKG_CONFIG_LIBDIR、PKG_CONFIG_ALLOW_CROSS、RUSTFLAGS） |
| `verify` | 验证所有 MODULES 可通过 sysroot 解析，打印版本号或 MISSING 原因 |
| `fetch` | 下载 Packages 索引，对每个包从索引查 Filename、下载 .deb、用 ar 解包、tar 提取到 SYSROOT |
| `main` | 入口：--check 模式仅验证；否则构建 sysroot 并验证，最后打印环境变量 |

## 关键要点总结

1. 运行于 macOS 和 Linux，需要 curl、ar、tar、awk、pkg-config，不需要 dpkg（.deb 是 ar 归档内含 tarball）。
2. 使用方式：`sh scripts/cross-sysroot.sh` 构建并打印要 export 的内容；`--check` 验证现有 sysroot。
3. `main "$@"` 在最后一行调用，防止截断下载只定义函数而不执行半构建。
4. 环境变量打印而非 export，因为脚本无法向调用者 shell 导出；调用者可读后 `eval`。
5. 缓存的 .deb 在 SYSROOT 下，重建无成本；删除 SYSROOT 即可重新开始。
6. 若 `--check` 失败，MISSING 行指出 pkg-config 找不到的模块，查 `Contents-arm64.gz` 索引找到提供它的包加入 PACKAGES。
