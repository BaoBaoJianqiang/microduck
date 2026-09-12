# web.rs 文件解析

**文件位置**：`d:\microduck\mediad\src\web.rs`

## 核心设计决策

由驱动它的守护进程自身提供的控制台页面。一个路由、一个文件、无构建步骤：`http://<robot>:8080/` 即可，无需其他操作（`webrtc-console.md` §1 解释为何值得一个依赖和第二个端口）。

**它消除了四个问题而非一个：**
1. 无需 `python3 -m http.server`，指令是地址而非两条命令。
2. 无需输入 URL——机器人提供的页面知道自己来自哪台机器人，从 `location.hostname` 推导信令目标。
3. Chrome 的 Private Network Access 检查不再适用（该检查针对从*公开/不透明*源到私有地址的请求；从 `192.168.x` 提供的页面是私有源）。
4. 页面与二进制一起发布，checkout 里的客户端不能再指向 release 的机器人。

### 端口与 API 版本在此填入，不由页面携带

`page(signalling_port)` 在启动时把两个 token 替换进内嵌页面一次。这使 `--port` 可安全修改：没有第二份副本。主机由浏览器填（`location.hostname`）。API 版本同理：页面无法比较版本除非知道自己说哪个，字面量会是 `API_VERSION` 的第二份副本——升级那天就错，且错在"报告一致"的方向。

从源码树直接打开的页面保留 token，读成 `NaN`，回退到 `webrtcsink` 默认的 8443；它也因此知道自己**不是**机器人提供的，可区分"主机错误"与"机器人提供了页面但信令端口没应答"。

## 常量与函数

### `PAGE: &str`
`include_str!("../webclient/index.html")`——编译时嵌入，而非请求时读文件：避免安装路径在三处漂移、避免 `ProtectSystem=strict` 下网络请求背后的文件读、避免"机器人在提供哪个页面"有两个答案。代价是改样式表需重编译。

### `PORT_TOKEN = "{{SIGNALLING_PORT}}"`、`API_TOKEN = "{{API_VERSION}}"`

### `page(signalling_port: u32) -> String`
替换两个 token。

### `serve(host, port, page) -> Result<()>`
绑定并提供直到进程结束。只在失败时返回。调用方决定代价——在 `mediad` 中只丢页面不丢视频。

### `router(page) -> Router`
一个路由 `/` 返回 `page`。用 `axum`（`default-features=false`，只 `http1`+`tokio`），因只提供一个文件，JSON/multipart/form/tracing 中间件都是不存在路由的重量。

## 单元测试

| 测试 | 意图 |
|---|---|
| `both_tokens_are_filled_in` | 两个 token 都被替换 |
| `the_page_carries_the_api_token_this_module_replaces` | 页面含 API token |
| `a_moved_port_reaches_the_page` | 非默认端口到达页面 |
| `the_route_answers_with_the_page` | 真实 socket 上 `GET /` 返回 200 + 页面 |
| `the_page_carries_the_token_this_module_replaces` | 页面含端口 token |

## 关键摘要

- 控制台页面由机器人自身提供，推导信令目标，免输 URL。
- 端口与 API 版本在提供时替换入内嵌页面，杜绝第二份副本。
- 单文件、无构建、无 npm，`include_str!` 编译时嵌入。
- `axum` 精简特性，一个路由。
