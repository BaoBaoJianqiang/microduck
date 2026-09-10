# `rknn.rs`（duck-detector）解读

## 概述

Rockchip NPU runtime 这个 crate 需要的七个函数，以及仅此而已。

### 核心设计：dlopen 而非链接

`librknnrt.so` 是厂商 blob：它不在任何 Debian 套件中，不在笔记本上，也不需要用来*构建*——一个链接它的守护进程根本无法在 CI 中交叉编译。`robotd` 以同样的方式、同样的理由到达 ONNX Runtime。代价是这个文件；好处是 `cargo board --bins` 在没有任何 Rockchip 东西的机器上继续工作。

C API 记录在 Rockchip 的 `rknpu2` 仓库中。这里重要的部分：

- `rknn_init` 接受模型*字节*，不是路径，所以调用者读取文件
- `rknn_query` 用固定大小的结构体回答，其布局是 ABI。它们在下面逐字段复刻；不匹配是静默的胡说而非错误，这就是为什么大小在启动时断言而非信任
- 量化模型的张量是 **int8 带 scale 和 zero point**，如果要求（`want_float`）`rknn_outputs_get` 会为你反量化。这里要求了：替代方案是把 scale 带进解码器并弄错一次

---

## 关键常量

### 运行时候选路径

```rust
const CANDIDATES: [&str; 4] = [
    "librknnrt.so",
    "/usr/lib/librknnrt.so",
    "/usr/lib/aarch64-linux-gnu/librknnrt.so",
    "/usr/local/lib/librknnrt.so",
];
```

运行时通常落地的位置，按值得尝试的顺序。第一个是 `scripts/setup-npu.sh` 放的位置；其余是从别处获得它的板子倾向于有的位置。搜索而非固定，以便手工配置的板子仍然工作。

### `rknn_query` 命令

```rust
const RKNN_QUERY_IN_OUT_NUM: c_uint = 0;
const RKNN_QUERY_INPUT_ATTR: c_uint = 1;
const RKNN_QUERY_OUTPUT_ATTR: c_uint = 2;
const RKNN_QUERY_SDK_VERSION: c_uint = 5;
```

**顺序是 ABI。** 这些曾经被猜错过，输入和输出交换了，症状是 `rknn_query(INPUT_ATTR)` 用*输出*张量回答——"无法理解输入形状 [1, 5, 2100]"，这是关于错误问题的完全合理的抱怨。除了在板子上运行没有任何东西能抓到那个，所以写下来而非推导。

### 张量类型和布局

```rust
const RKNN_TENSOR_UINT8: c_uint = 3;   // 张量元素类型：float32, float16, int8, uint8, …
const RKNN_TENSOR_NHWC: c_uint = 1;    // 张量布局：NCHW 是 0，NHWC 是 1
```

#### NHWC 值得强调

> 因为弄反它不会失败。运行时日志 "Meet unsupported src layout for normalize: NCHW, only support NHWC src layout"——然后 `rknn_inputs_set` **仍然返回成功**，所以推理在输入缓冲区里的任何东西上运行，检测器在每一帧上报告恰好两个置信框，永远。从外面看，每帧两个相同的检测就是那个样子。

### 其他

```rust
const RKNN_MAX_DIMS: usize = 16;
const RKNN_MAX_NAME_LEN: usize = 256;
```

---

## C ABI 结构体

### `RknnInputOutputNum`

```rust
#[repr(C)]
struct RknnInputOutputNum {
    n_input: c_uint,
    n_output: c_uint,
}
```

输入输出数量。大小 8 字节。

### `RknnTensorAttr` — 最关键的 ABI 结构体

```rust
#[repr(C)]
struct RknnTensorAttr {
    index: c_uint,
    n_dims: c_uint,
    dims: [c_uint; RKNN_MAX_DIMS],   // 16
    name: [c_char; RKNN_MAX_NAME_LEN], // 256
    n_elems: c_uint,
    size: c_uint,
    fmt: c_uint,
    type_: c_uint,
    qnt_type: c_uint,
    fl: i8,
    zp: i32,
    scale: f32,
    w_stride: c_uint,
    size_with_stride: c_uint,
    pass_through: u8,
    h_stride: c_uint,
}
```

总大小 376 字节。关键字段偏移：
- `dims` 在偏移 8
- `name` 在偏移 72
- `n_elems` 在偏移 328
- `zp`（zero point）在偏移 352
- `scale` 在偏移 356

**这些是 ABI，弄错一个是静默的。** 不是厂商头的替代品——它是添加字段位置错误的编辑的绊线，这里没有编译器能抓到因为这个边界的另一边是没人链接的 blob。

### `RknnInput`

```rust
#[repr(C)]
struct RknnInput {
    index: c_uint,
    buf: *mut c_void,
    size: c_uint,
    pass_through: u8,
    type_: c_uint,
    fmt: c_uint,
}
```

每次推理按值传递。`buf` 在偏移 8。

### `RknnOutput`

```rust
#[repr(C)]
struct RknnOutput {
    want_float: u8,
    is_prealloc: u8,
    index: c_uint,
    buf: *mut c_void,
    size: c_uint,
}
```

`want_float` 和 `is_prealloc` 是字节，`index` 是 u32，后面的指针是 8 对齐——所以 `buf` 在偏移 8，不是 16。

### `RknnSdkVersion`

```rust
#[repr(C)]
struct RknnSdkVersion {
    api_version: [c_char; 256],
    drv_version: [c_char; 256],
}
```

API 和驱动版本字符串，各 256 字节，总 512 字节。

---

## 函数指针类型

```rust
type RknnInitFn = unsafe extern "C" fn(*mut *mut c_void, *const c_void, c_uint, c_uint, *const c_void) -> c_int;
type RknnDestroyFn = unsafe extern "C" fn(*mut c_void) -> c_int;
type RknnQueryFn = unsafe extern "C" fn(*mut c_void, c_uint, *mut c_void, c_uint) -> c_int;
type RknnInputsSetFn = unsafe extern "C" fn(*mut c_void, c_uint, *mut RknnInput) -> c_int;
type RknnRunFn = unsafe extern "C" fn(*mut c_void, *const c_void) -> c_int;
type RknnOutputsGetFn = unsafe extern "C" fn(*mut c_void, c_uint, *mut RknnOutput, *const c_void) -> c_int;
type RknnOutputsReleaseFn = unsafe extern "C" fn(*mut c_void, c_uint, *mut RknnOutput) -> c_int;
```

七个函数：init、destroy、query、inputs_set、run、outputs_get、outputs_release。

---

## 核心数据结构

### `Model` — 加载的 `librknnrt.so` 和运行在其上的模型

```rust
pub struct Model {
    _library: libloading::Library,  // 最后丢弃：下面每个函数指针都属于这个库
    context: *mut c_void,
    destroy: RknnDestroyFn,
    inputs_set: RknnInputsSetFn,
    run: RknnRunFn,
    outputs_get: RknnOutputsGetFn,
    outputs_release: RknnOutputsReleaseFn,
    pub input: (usize, usize, usize),       // [height, width, channels]
    pub output_len: usize,                   // 返回多少 float
    pub api_version: String,
    pub driver_version: String,
}
```

#### `_library` 最后丢弃

下面每个函数指针都属于这个库，在 context 还活着时卸载它是段错误而非错误。Rust 的字段丢弃顺序是声明顺序，所以 `_library` 声明在最前意味着它最后丢弃。

#### `unsafe impl Send for Model {}`

```rust
unsafe impl Send for Model {}
```

context 是这个类型独占拥有的句柄——什么都不把它交出去，`infer` 接受 `&mut self`，所以两个线程不能同时在 runtime 内。把整个东西移到另一个线程是 `mediad` 做的（检测器在自己的线程上运行），这是健全的；`Sync` 故意*不*声明，因为两个线程共享一个 context 正是 runtime 不支持的。

---

## 核心函数

### `why_init_failed()` — 为什么 `rknn_init` 失败

```rust
fn why_init_failed() -> String
```

按原因实际发生的顺序。

#### 设备树第一，因为这是每个板子开始的状态

> Armbian 在 Radxa Zero 3 上把 `npu@fde40000`  shipped 为 `status = "disabled"`，所以没人运行过 `setup-npu.sh` 的机器人有硬件、内核、驱动和运行时，仍然没有 NPU——运行时自己的日志行是 "failed to open rknpu module, need to insmod rknpu dirver!"，这让人去找一个内置的模块。

三种情况：
1. `/proc/device-tree/npu@fde40000/status` 以 `disabled` 开头 → NPU 在设备树中被禁用，告诉运行 `setup-npu.sh` 并重启
2. 节点不存在 → 不是 RK3566，或设备树不描述它的内核（主线没有 rknpu 驱动）
3. 节点已启用 → 这是模型或驱动：为其他平台构建的模型，或比运行时旧的 NPU 驱动

这曾经先命名另外两个原因，但它们是有设备可对话之后剩下的。先命名它们花了一次 bring-up 会话。

### `Model::open()` — 加载运行时和模型

```rust
pub fn open(path: &Path) -> Result<Self>
```

#### 1. 读取模型字节

```rust
let bytes = std::fs::read(path)?;
```

`rknn_init` 接受模型字节，不是路径。

#### 2. 加载 `librknnrt.so`

```rust
let library = CANDIDATES.iter()
    .find_map(|candidate| unsafe { libloading::Library::new(*candidate) }.ok())
    .ok_or_else(|| anyhow!("cannot load librknnrt.so ({}). ... sudo /usr/local/sbin/robot-setup-npu", ...))?;
```

按顺序尝试四个候选路径，记录最后一个错误。找不到时告诉运行 `robot-setup-npu`。

#### 3. 查找七个符号

```rust
unsafe {
    let init = *library.get::<RknnInitFn>(b"rknn_init\0")?;
    // ... 其余六个
}
```

每个符号按厂商头声明的名称查找，签名从头转录。缺失的符号在这里是错误而非后来跳进虚无。

#### 4. 初始化 context

```rust
let mut context: *mut c_void = std::ptr::null_mut();
let code = init(&mut context, bytes.as_ptr() as *const c_void, bytes.len() as c_uint, 0, std::ptr::null());
if code != 0 || context.is_null() {
    bail!("rknn_init failed ({code}) on {}. {}", path.display(), why_init_failed());
}
```

失败时调用 `why_init_failed()` 给出可操作的诊断。

#### 5. 查询 SDK 版本

```rust
let mut version = RknnSdkVersion { api_version: [0; 256], drv_version: [0; 256] };
query(context, RKNN_QUERY_SDK_VERSION, &mut version as *mut _ as *mut c_void, size_of::<RknnSdkVersion>() as c_uint);
```

#### 6. 查询输入输出数量

```rust
let mut counts = RknnInputOutputNum { n_input: 0, n_output: 0 };
let code = query(context, RKNN_QUERY_IN_OUT_NUM, ...);
if counts.n_input != 1 || counts.n_output != 1 {
    bail!("expected one input and one output, got {} and {} — this decoder only knows the single-tensor YOLO head", ...);
}
```

这个解码器只懂单张量 YOLO 头，多输入多输出的模型明确拒绝。

#### 7. 查询输入输出张量属性

```rust
let input = attr(RKNN_QUERY_INPUT_ATTR, 0)?;
let output = attr(RKNN_QUERY_OUTPUT_ATTR, 0)?;
```

#### 8. 解析输入形状（NCHW 或 NHWC）

```rust
let dims = &input.dims[..input.n_dims as usize];
let shape = match dims {
    // NCHW 先，且仅当第二维小到是通道：
    [_, c, h, w] if *c <= 4 && *h > 4 => (*h as usize, *w as usize, *c as usize),
    [_, h, w, c] if *c <= 4 => (*h as usize, *w as usize, *c as usize),
    other => { bail!("cannot make sense of the input shape {other:?}"); }
};
```

NHWC 如模型声明的，这也是 RGA 和 JPEG 解码器产生的——路上不需要转置的唯一布局。

模型从 NCHW ONNX 转换，运行时报告构建选定的任一布局。320×320×3 张量两种布局数字相同，但 3×320×320 读作 HWC 会要求三行 320 像素。用 `c <= 4 && h > 4` 区分。

#### 9. 构建 Model

```rust
Ok(Self {
    context, destroy, inputs_set, run, outputs_get, outputs_release,
    input: shape,
    output_len: output.n_elems as usize,
    api_version: c_str(&version.api_version),
    driver_version: c_str(&version.drv_version),
    _library: library,
})
```

### `Model::infer()` — 一次推理

```rust
pub fn infer(&mut self, frame: &[u8], out: &mut Vec<f32>) -> Result<()>
```

一个帧进，原始头出为 float。

`frame` 是模型自己布局的 `height × width × channels` uint8——正是 `Model::input` 描述的。运行时做转换烘焙进去的归一化（mean 0, std 255），所以这交给它字节并拿回分数。

#### 帧大小验证

```rust
let wanted = height * width * channels;
if frame.len() != wanted {
    bail!("frame is {} bytes, the model wants {wanted}", frame.len());
}
```

#### 设置输入

```rust
let mut input = RknnInput {
    index: 0,
    buf: frame.as_ptr() as *mut c_void,
    size: frame.len() as c_uint,
    pass_through: 0,
    type_: RKNN_TENSOR_UINT8,
    fmt: RKNN_TENSOR_NHWC,
};
```

uint8，NHWC。**弄反布局不会失败**——运行时记录不支持但 `rknn_inputs_set` 返回成功，推理在垃圾上运行。

#### 运行

```rust
let code = (self.inputs_set)(self.context, 1, &mut input);
let code = (self.run)(self.context, std::ptr::null());
```

#### 获取输出（运行时反量化）

```rust
let mut output = RknnOutput {
    want_float: 1,  // 运行时反量化
    is_prealloc: 0,
    index: 0,
    buf: std::ptr::null_mut(),
    size: 0,
};
let code = (self.outputs_get)(self.context, 1, &mut output, std::ptr::null());
```

**由运行时反量化。** 量化模型的输出是 int8 带 scale 和 zero point；在这里要求 float 把那个算术保持在一个地方——厂商的——而非在会弄错一次并被相信的解码器中。

#### 复制输出并释放

```rust
let count = output.size as usize / size_of::<f32>();
out.clear();
out.extend_from_slice(std::slice::from_raw_parts(output.buf as *const f32, count));
(self.outputs_release)(self.context, 1, &mut output);
```

输出缓冲区是运行时自己的，直到 `outputs_release`。必须复制然后释放，否则泄漏。

### `Drop for Model`

```rust
impl Drop for Model {
    fn drop(&mut self) {
        if !self.context.is_null() {
            unsafe { (self.destroy)(self.context) };
            self.context = std::ptr::null_mut();
        }
    }
}
```

销毁 context，在库卸载之前（字段丢弃顺序保证）。

### `c_str()` — NUL 终止的厂商字符串

```rust
fn c_str(raw: &[c_char]) -> String
```

取到第一个 NUL，转 String。没有 NUL 时返回空。

---

## 测试要点

文件包含 2 个测试：

1. **`the_query_structs_are_laid_out_as_the_abi_says`** — 查询结构体按 ABI 布局。
   - `RknnInputOutputNum` 大小 8
   - `RknnSdkVersion` 大小 512
   - `RknnTensorAttr` 关键字段偏移：dims=8, name=72, n_elems=328, zp=352, scale=356，总大小 376
   - `RknnInput.buf` 偏移 8
   - `RknnOutput.buf` 偏移 8（不是 16，因为 want_float/is_prealloc 是字节，index 是 u32，指针 8 对齐）
   
   **偏移，不只是总数。** 两个字段交换保持大小但改变意义，这个边界的另一边是这里没有编译器能检查的 blob。

2. **`a_missing_runtime_names_the_setup_script`** — 缺失的运行时必须说运行什么，而非 "cannot open shared object file"。模型在库加载之前读取，所以不存在的模型先报告文件错误。

---

## 与其他模块的关系

- **`crate::decode`**：后处理解码，`infer` 返回 float 原始头（平面 `[1,5,N]`），与 onnx 后端完全一致
- **`crate::onnx`**：CPU 后端，`Model` 接口对称（`open`/`infer`/`input`）
- **`libloading`**：运行时 dlopen 库
- **`librknnrt.so`**：Rockchip NPU 运行时（厂商 blob）
- **`scripts/setup-npu.sh`**：安装运行时并启用设备树中的 NPU
- **`mediad`**：在自己的线程上运行检测器（`Model: Send` 但不 `Sync`）

---

## 关键踩坑点总结

1. **dlopen 而非链接**：`librknnrt.so` 是厂商 blob，不在 Debian 中，不在笔记本上，链接它会破坏 CI 交叉编译。dlopen 让 `cargo board --bins` 在没有 Rockchip 东西的机器上继续工作。

2. **ABI 结构体布局是静默错误**：`RknnTensorAttr` 等 `#[repr(C)]` 结构体的字段偏移是 ABI，弄错一个是静默的胡说而非错误。启动时断言大小和偏移（测试中），而非信任。

3. **`rknn_query` 命令顺序是 ABI**：INPUT_ATTR=1, OUTPUT_ATTR=2。曾经猜错过（交换了），症状是用输出张量回答输入查询——"无法理解输入形状 [1,5,2100]"。除了在板子上运行没有东西能抓到。

4. **NHWC 弄反不会失败**：运行时记录 "unsupported src layout" 但 `rknn_inputs_set` 返回成功，推理在垃圾上运行，检测器每帧报告恰好两个置信框。从外面看这就是那个样子。

5. **运行时反量化（`want_float=1`）**：量化模型输出是 int8 带 scale/zero_point，让运行时反量化保持算术在厂商代码中，而非在会弄错一次并被相信的解码器中。

6. **设备树禁用是最常见的 init 失败原因**：Armbian 把 `npu@fde40000` shipped 为 disabled，机器人有硬件/内核/驱动/运行时但仍然没有 NPU。运行时自己的日志误导人去找内置的模块。`why_init_failed()` 先检查设备树。

7. **`Model: Send` 但不 `Sync`**：context 独占拥有，`infer` 取 `&mut self`，移到另一个线程健全（mediad 这样做），但两个线程共享一个 context 是 runtime 不支持的。

8. **`_library` 最后丢弃**：函数指针属于库，在 context 还活着时卸载是段错误。Rust 字段丢弃顺序是声明顺序，所以 `_library` 声明在最前意味着最后丢弃。

9. **输入形状 NCHW/NHWC 自动检测**：模型从 NCHW ONNX 转换，运行时报告构建选定的布局。用 `c <= 4 && h > 4` 区分，320×320×3 两种布局数字相同但 3×320×320 读作 HWC 会出错。

10. **输出必须复制然后释放**：`rknn_outputs_get` 返回的缓冲区是运行时自己的，必须复制到调用者缓冲区然后 `rknn_outputs_release`，否则泄漏。
#（注：内容由AI生成）
