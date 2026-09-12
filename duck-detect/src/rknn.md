# rknn.rs 文件解析

## 文件位置

`d:\microduck\duck-detect\src\rknn.rs`

## 核心设计决策

`rknn.rs` 封装 Rockchip NPU runtime 所需的**七个函数，仅此而已**。

**`dlopen` 而非链接**：`librknnrt.so` 是厂商 blob，不在任何 Debian 套件、不在笔记本上、也不需要用于*构建*——链接它的守护进程根本无法在 CI 中交叉编译。`robotd` 以同样方式接触 ONNX Runtime，理由相同。代价是这个文件；好处是 `cargo board --bins` 在没有任何 Rockchip 东西的机器上仍能工作。

C API 记录在 Rockchip 的 `rknpu2` 仓库。关键点：
- `rknn_init` 接收模型*字节*而非路径，调用方读文件
- `rknn_query` 用固定大小结构体回答，布局即 ABI。下方逐字段复刻；不匹配是静默废话而非错误，所以在启动时断言大小
- 量化模型的张量是 **int8 + scale + zero point**，`rknn_outputs_get` 可代为反量化（`want_float`）。此处请求反量化，否则要把 scale 带进解码器且迟早出错

## 常量

- `CANDIDATES` — `librknnrt.so` 的搜索路径（4 个，从 `setup-npu.sh` 放置处开始），搜索而非固定以便手工部署的板也能用
- `RKNN_QUERY_*` — query 命令：0=IN_OUT_NUM, 1=INPUT_ATTR, 2=OUTPUT_ATTR, 5=SDK_VERSION。**顺序即 ABI**：曾把输入输出搞反，症状是 `rknn_query(INPUT_ATTR)` 用*输出*张量回答——"无法理解输入形状 [1,5,2100]"
- `RKNN_TENSOR_UINT8 = 3` — 元素类型
- `RKNN_TENSOR_NHWC = 1` — 布局（NCHW=0, NHWC=1）。**搞反不会失败**：runtime 日志"不支持 NCHW src layout"，但 `rknn_inputs_set` **仍返回成功**，推理在输入缓冲里的任何东西上运行，检测器每帧报告恰好两个置信框

## 类型分析

### ABI 结构体（`#[repr(C)]`）
- `RknnInputOutputNum` — `n_input`, `n_output`
- `RknnTensorAttr` — 完整张量属性（index, n_dims, dims[16], name[256], n_elems, size, fmt, type_, qnt_type, fl, zp, scale, 跨步字段等），376 字节
- `RknnInput` — index, buf, size, pass_through, type_, fmt
- `RknnOutput` — want_float, is_prealloc, index, buf, size
- `RknnSdkVersion` — api_version[256], drv_version[256]

### 函数指针类型
`RknnInitFn`、`RknnDestroyFn`、`RknnQueryFn`、`RknnInputsSetFn`、`RknnRunFn`、`RknnOutputsGetFn`、`RknnOutputsReleaseFn`。

### `Model`
已加载的 `librknnrt.so` + 运行其上的模型：
- `_library: Library` — 最后丢弃，因为所有函数指针属于此库
- `context: *mut c_void` — rknn 上下文句柄
- 各函数指针
- `input: (height, width, channels)` — 从模型自身查询
- `output_len: usize`
- `api_version` / `driver_version`

`unsafe impl Send for Model {}` — 上下文由本类型独占，`infer` 取 `&mut self`，不能两个线程同时进入 runtime。`mediad` 把检测器移到自己线程运行，`Send` 合理；`Sync` 刻意不实现。

## 方法分析

### `why_init_failed() -> String`
`rknn_init` 失败的原因诊断，按实际发生顺序排列：
1. **设备树第一**：Armbian 在 Radxa Zero 3 上把 `npu@fde40000` 标为 `disabled`，所以没跑 `setup-npu.sh` 的机器人有硬件、内核、驱动、runtime，仍无 NPU。runtime 自己的日志是"failed to open rknpu module"，让人去找一个已内建的模块。读 `/proc/device-tree/npu@fde40000/status` 判断。
2. 无此节点：非 RK3566 或内核设备树不描述它
3. 节点已启用：模型或驱动问题（为其他平台构建的模型，或驱动旧于 runtime）

### `open(path) -> Result<Self>`
1. 读模型字节
2. 遍历 `CANDIDATES` dlopen `librknnrt.so`
3. unsafe 获取 7 个函数符号
4. `rknn_init` 传入模型字节，失败则 `why_init_failed()` 诊断
5. query SDK 版本
6. query 输入输出数量，期望 1 入 1 出
7. query 输入/输出张量属性
8. 解析输入形状：NCHW `[_,c,h,w]`（c≤4 且 h>4）或 NHWC `[_,h,w,c]`（c≤4），输出 `(h, w, c)`

### `infer(frame, out) -> Result<()>`
- 校验帧长度
- 构造 `RknnInput`：`type_=UINT8`, `fmt=NHWC`
- 构造 `RknnOutput`：`want_float=1`（**runtime 反量化**）
- `inputs_set` → `run` → `outputs_get` → 复制 f32 到 `out` → `outputs_release`

### `Drop`
若 context 非空，调用 `destroy`，然后置空。

### `c_str(raw) -> String`
从 `[c_char]` 提取 NUL 结尾字符串。

## 单元测试描述

- `the_query_structs_are_laid_out_as_the_abi_says`：ABI 结构体尺寸与字段偏移断言（`RknnTensorAttr` 376 字节、dims@8、name@72、n_elems@328、zp@352、scale@356；`RknnInput`/`RknnOutput` 的 buf@8）。不是厂商头的替代，是编辑时加错字段的绊线。
- `a_missing_runtime_names_the_setup_script`：模型不存在时先报文件错误（模型在库加载前读取）。

## 关键摘要

`rknn.rs` 通过 `libloading` dlopen `librknnrt.so`，逐字段复刻 Rockchip C ABI 结构体并断言尺寸/偏移。`rknn_init` 失败时诊断设备树（Armbian 默认禁用 NPU）。输入用 NHWC+UINT8，输出请求 runtime 反量化为 float（避免在解码器中处理 scale）。`Send` 而非 `Sync`。
