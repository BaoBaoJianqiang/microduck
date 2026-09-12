# rkaiq-modinfo-shim.c

## 文件位置

`d:\microduck\scripts\rkaiq-modinfo-shim.c`

## 核心设计决策

这是一个 LD_PRELOAD shim，用于 Radxa Zero 3W（RK3566）上的 `rkaiq_3A_server`。

- **问题根源**：librkaiq 通过 `RKMODULE_GET_MODULE_INFO` 私有 ioctl 查询传感器驱动的模块信息（sensor/module/lens 名称，用于在 `/etc/iqfiles` 中选择 IQ 调优文件）。ioctl 号编码了 `sizeof(struct rkmodule_inf)`，该大小在 BSP 内核版本间漂移——Radxa 的 camera-engine-rkaiq deb 是针对与 Armbian vendor 内核不同的头文件构建的（6.1.115 上为 5203 vs 5207 字节），导致 ioctl 以 ENOTTY 失败，librkaiq 静默得到空 IQ 文件名（`/etc/iqfiles//`）并段错误。
- **shim 策略**：拦截不匹配的 ioctl（type 'V'，nr 0xc0 = private 0），通过暴力探测一次内核期望的结构体大小，用内核自己的 ioctl 号执行调用，然后只复制基础信息（前 96 字节：sensor[32] + module[32] + lens[32]）——这是 librkaiq 选择 IQ 文件所需的全部。
- **运行时探测而非编译时**：结构体大小是运行内核的属性，但此 shim 在首次拦截的 ioctl 时运行时探测，因此目标文件不依赖编译时的内核。`setup-rkaiq.sh` 只在源码变化时重建。
- **只拦截 rkaiq_3A 服务**：systemd drop-in 只为该服务设置 LD_PRELOAD，板上其他任何东西都不拦截此 ioctl。

## 常量/类型/函数分析

### 常量

| 常量 | 值 | 说明 |
|---|---|---|
| `MODINFO_TYPE` | `0x56` ('V') | ioctl type 字段 |
| `MODINFO_NR` | `0xc0` | ioctl nr 字段（BASE_VIDIOC_PRIVATE + 0） |
| 缓冲区大小 | `16384` | `probe_kernel_req` 和 `ioctl` 中 `kbuf` 的大小 |

### 函数

| 函数 | 功能 |
|---|---|
| `probe_kernel_req(int fd)` | 扫描候选结构体大小（96 到 16384 字节），用 `_IOC(_IOC_READ, MODINFO_TYPE, MODINFO_NR, sz)` 构造 ioctl 号，调用 `real_ioctl` 直到返回 0（成功）。结果缓存在静态变量中，只运行一次（约 1 万次失败 syscall ≈ 10ms） |
| `ioctl(int fd, unsigned long req, ...)` | 拦截函数。首次调用时用 `dlsym(RTLD_NEXT, "ioctl")` 获取真实 ioctl。若请求匹配 GET_MODULE_INFO 且探测到的内核 ioctl 号与请求不同，则用内核号调用并复制前 96 字节（或 `sz` 取较小者）回调用者缓冲区；否则透传给真实 ioctl |

## 关键要点总结

1. 编译命令：`gcc -shared -fPIC -O2 -o rkaiq_modinfo_shim.so rkaiq-modinfo-shim.c -ldl`
2. 必须在板端编译为 aarch64（仅此一个原因），由 `scripts/setup-rkaiq.sh` 构建。
3. 只复制前 96 字节，因为 librkaiq 只需 sensor/module/lens 名称来选 IQ 文件。
4. 暴力探测范围 96–16384 字节，缓存结果避免重复探测。
5. 这是让 Radxa engine deb 在 Armbian vendor 内核上运行的唯一方案，从原型（microduck_runtime/radxa_setup）原样移植。
