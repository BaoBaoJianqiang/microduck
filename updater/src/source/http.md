# source/http.rs 文件解析

## 文件位置

`d:\microduck\updater\src\source\http.rs`

## 定位

网络源（GitHub Releases、HF Hub 等）共享的 **HTTP 底层管道**。具体源实现负责"URL 是什么、元数据怎么解析"，本文件负责所有联网请求都必须满足的两件事：**有界（Bounded）** 与 **可续传（Resumable）**。

## TLS：rustls + 操作系统信任库

TLS 走 rustls，经 `rustls-platform-verifier` 取证书根——根证书来自 **OS 信任库**而非随二进制打包的副本。对机器人的意义（文件头注释）：

- 根证书跟随 Debian 的安全更新，不必为换根证书发一次守护进程版本；
- 运维者自行安装的 CA 无需重新构建即可生效。

## 两个必须始终成立的性质

- **有界**：每个请求有连接超时；每次下载有**逐块停滞超时**而非总时限。挂死的镜像不能让一次更新永远挂着；但大 artifact 上一个慢镜像也不能被总时限杀掉——所以超时是"两块数据之间最长间隔"，不是"整次下载最长时间"。
- **可续传**：大文件连接中断时带 `Range` 头重试，而不是从头再来——机器人在家庭 wifi 上。

## 超时常量

| 常量 | 值 | 用途 |
|---|---|---|
| `CONNECT_TIMEOUT` | 15 s | 连接超时：慢上行够用，不可达主机尽快失败 |
| `METADATA_TIMEOUT` | 30 s | 小 JSON/元数据请求的整请求超时 |
| `CHUNK_STALL_TIMEOUT` | 60 s | 下载两块数据间的最长间隔，超过判停滞；刻意不是总 deadline |
| `DOWNLOAD_ATTEMPTS` | 4 | 每个 artifact 的尝试次数（含首次），重试经 `Range` 续传 |

## 重试分类：`Retry` 与 `classify()`

`Retry::Worth` / `Retry::Pointless` 两态枚举。`classify(status)` 按 HTTP 状态码分类：

- **408、429**（超时、限流）：明确"稍后再来"→ Worth；
- **其余 4xx**：请求本身错了（错误的仓库/tag/资产名 → 404），半秒后再问答案不变，重试只会白白耗掉整个退避预算 → Pointless；
- **5xx** 及其他：服务端故障/传输层问题，家庭 wifi 会掉连接、镜像也有状态差的几分钟 → Worth。

## 体积上限

| 常量 | 值 | 理由 |
|---|---|---|
| `MAX_METADATA_BYTES` | 1 MiB | 清单只有几百字节；接近这个数说明在跟错误的对象说话。下载前先查 `Content-Length`，读完再查实际长度 |
| `MAX_ARTIFACT_BYTES` | 4 GiB | 绝对天花板：错误的 URL 不能在哈希校验拒绝它之前先把 eMMC 填满。下载前按 `Content-Length` 预判，下载中逐块累计再判；超限属 Pointless（发布错误，不是状态差的一分钟） |

## 公共 API

### `user_agent()` / `client()`

UA 为 `updaterd/<CARGO_PKG_VERSION>`。client 构建器设置 UA、15 s 连接超时、**重定向上限 5 次**（跟随重定向但不无限追——重定向环是镜像配置错误）。

### `get_bytes(client, url, accept)` —— 整体拉取小资源

用于清单、签名、API 响应：30 s 整请求超时；可选 `Accept` 头；URL 含 `github.com` 时附 `GITHUB_TOKEN` 的 bearer 认证；非 2xx 返回带诊断提示的错误；`Content-Length` 与实际长度双重 1 MiB 上限检查。

### `download_to(client, url, dest, accept, progress) -> u64` —— 带续传的流式下载

**重试外层**（最多 4 次）：

1. 用 `metadata(dest).len()` 看上次写了多少字节（`already`），有残留即记日志"resuming"；
2. 调 `attempt_download()`；
3. `Pointless`：删掉具有误导性的部分文件并立即返回错误（等待不会改变答案）；
4. `Worth`：保存错误，**递增退避** `500ms × attempt` 后重试；
5. 四次尽墨则返回最后一个错误。

**完整性不在此处检查**：只负责把字节落盘，哈希与签名由调用方验证（那才是权威检查）。

## `attempt_download()` 单次尝试细节

- 续传点 > 0 时发 `Range: bytes=<n>-`；同样处理 `Accept` 与 GitHub token；
- 非成功状态按 `classify()` 分类返回；
- **206 vs 200**：只有状态为 `206 PARTIAL_CONTENT` 才是真续传；服务器忽略 `Range` 会回 200 + 完整正文——此时必须从头开始（`truncate`），追加反而会损坏文件；
- 打开文件：`create + write`，续传时不 truncate、seek 到续传点，非续传时 truncate；
- 流式循环：每个 chunk 外套 60 s `tokio::time::timeout` 实现逐块停滞检测；累计字节数超限即 Pointless 失败；写入失败为 Worth；
- **进度上报失败被忽略**（`let _ = progress.send(...)`）：没人看不是停止下载的理由；
- 收尾 `flush()` 后必须 **`sync_data()`**——哈希是从这个文件读回的，数据必须真的在盘上；
- 若服务器给过总长度但实际写字节不符（截断传输），按 Worth 再试一次。

`accept` 存在的理由（注释）：GitHub release-asset API 默认返回资产的**元数据 JSON**，除非请求 `application/octet-stream`；漏了这个头会把一个 JSON blob 以 artifact 之名写盘，然后在哈希检查处失败——一种令人困惑的发现方式。

## `github_token()` 与 `describe_failure()`

- **token 从环境变量 `GITHUB_TOKEN` 读，不从配置读**：这样它永远不会落进会被复制来复制去的文件；空白 token 视为不存在（防 unit 文件里 `GITHUB_TOKEN=` 变成一个空的 `Bearer ` 头）；只发给 `github.com`——发给重定向目标会泄漏令牌。
- `describe_failure()` 把状态码转成支持工单里可直接行动的信息：401/403 提示私有仓库或限流、429 限流、5xx 服务端。**404 的提示特别加了 token**：GitHub 对无权查看的私有资源回 404 而非 403（以免泄漏其存在），所以最常表示"你需要 token"的状态恰恰最像"名字写错了"——第一块真板上这条消息曾让人去查一个本来正确的仓库名。

## 单元测试（第 378-418 行）

- `client_builds`：客户端能构建；
- `user_agent_identifies_us`：UA 以 `updaterd/` 开头；
- `failures_carry_a_hint`：404 消息必须同时包含"wrong repo"与"GITHUB_TOKEN"，403 必须含 token 提示——测试守护的是"错误消息必须可行动"；
- `blank_token_is_treat_as_absent`：全空白 token 返回 None。

## 关键摘要

http.rs 是所有网络源共享的下载管道：rustls 经 OS 信任库验证证书（根证书随 Debian 更新、运维 CA 免重建）；有界（15 s 连接、30 s 元数据、60 s 逐块停滞而非总时限）与可续传（最多 4 次、`Range` 续传、递增退避、识别 200/206）两大性质；4xx（除 408/429）不重试且删残留，5xx/传输错误重试；元数据 1 MiB、artifact 4 GiB 双天花板防止填满 eMMC；字节落盘后 `sync_data` 再交调用方验签；`GITHUB_TOKEN` 只从环境读、只发 github.com；404 诊断刻意提示 token，因 GitHub 对私有资源回 404。
