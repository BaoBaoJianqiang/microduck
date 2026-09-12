# build.rs 文件解析

## 文件位置

`d:\microduck\tof\build.rs`

## 核心设计决策

1. **用 `cc` crate 编译 vendored ULD，无 autotools/系统库/Python。** `cc` 从环境取目标编译器，因此支持交叉编译——`cargo board`（cargo-zigbuild）为 aarch64 导出 `zig cc`，与 updater 编译 `zstd` 同一路径。

2. **六个符号重命名解决两代 ULD 的平台钩子冲突。** 每个 ULD 都以裸名 `RdByte`/`WrByte`/`RdMulti`/`WrMulti`/`SwapBuffer`/`WaitMs` 调用平台钩子，而我们只有一份实现（`vendor/platform.c`）。两代同二进制会定义相同六符号两次。因 ULD 源码是上游未编辑的，重命名在预处理器完成：每代以 `vl5_`/`vl8_` 前缀编译自己的 `platform.c` 副本，其 ULD 也以相同 `define` 编译使调用点跟随。`TOF_PLATFORM` 命名该副本构建所针对的平台结构体类型。

3. **警告不当错误。** `vl53l?cx_api.c` 是上游代码不编辑，新编译器在其中发现东西不能阻止机器人版本构建。

4. **仅 Linux，且安静跳过。** `vendor/platform.c` 通过 `linux/i2c.h` 的 `I2C_RDWR` ioctl 访问总线，其他平台不存在。注意用 `CARGO_CFG_TARGET_OS`（目标）而非 `cfg!(target_os)`（主机），否则交叉编译到板子时会在笔记本上判定非 Linux 而什么都不编译。

## 数据结构

```rust
struct Generation {
    dir: &'static str,       // vendor/ 下目录，也是静态库名
    prefix: &'static str,    // 该代平台钩子的符号前缀
    platform: &'static str,  // 其头文件定义的平台结构体类型
}

const GENERATIONS: [Generation; 2] = [
    { dir: "vl53l8cx", prefix: "vl8", platform: "VL53L8CX_Platform" },
    { dir: "vl53l5cx", prefix: "vl5", platform: "VL53L5CX_Platform" },
];

const HOOKS: [&str; 6] = ["RdByte", "WrByte", "RdMulti", "WrMulti", "SwapBuffer", "WaitMs"];
```

## `main()` 流程

1. 检查 `CARGO_CFG_TARGET_OS` 是否为 `linux`，否则只 `rerun-if-changed=build.rs` 后返回
2. 对每代：
   - `include` 该代目录（其 `platform.h` 与 `*_buffers.h` 优先，避免看到另一代头）
   - `define("TOF_PLATFORM", generation.platform)`
   - 编译 `<dir>_api.c` + `shim.c` + `platform.c`
   - 对每个钩子 `define(hook, "<prefix>_<hook>")`
   - `warnings(false)`、`opt_level(2)`、`compile(dir)`
3. 编译代无关的 `probe.c` 为 `tof_probe`（决定上面哪代与传感器对话）
4. 注册 `rerun-if-changed`：`build.rs`、`vendor/platform.c`、`vendor/probe.c`、每代的 `api.c/api.h/buffers.h`、`platform.h`、`shim.c`（固件 blob 是 550 KB 头文件，无源文件引用，必须显式命名，否则驱动更新不触发重编译）

## 关键摘要

build.rs 用 `cc` 编译两代 ST ULD：通过预处理器符号重命名让两代共享的 `platform.c` 钩子不冲突；`TOF_PLATFORM` 宏选择正确的平台结构体类型；警告降级、opt 2；非 Linux 目标安静跳过（用 `CARGO_CFG_TARGET_OS` 而非 `cfg!`）；显式追踪固件 blob 头文件的修改以触发重编译。
