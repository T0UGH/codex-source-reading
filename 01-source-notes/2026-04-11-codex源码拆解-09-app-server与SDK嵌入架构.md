# Codex 源码拆解 09：app-server 与 SDK 嵌入架构

## 这篇看什么

回答：**Codex 如何被别的系统调用？为什么有 app-server，又为什么 SDK 并不完全统一地走 app-server？**

## 先给结论

Codex 的嵌入式能力至少有两条线：

1. **app-server 线**：更正式的控制平面 / RPC 服务层
2. **CLI SDK 线**：直接把 `codex` CLI 当 runtime 去 spawn

目前能看到一个明显事实：
- **Python SDK** 更偏 app-server 路线
- **TypeScript SDK** 更偏 CLI exec + JSONL event 路线

这说明 Codex 的嵌入式能力还不是完全单轨，而是处在多条 surface 并存的状态。

## 关键代码路径

- `codex-rs/app-server/src/lib.rs`
- `codex-rs/app-server/src/in_process.rs`
- `codex-rs/app-server-client/src/lib.rs`
- `codex-rs/app-server-client/src/remote.rs`
- `codex-rs/app-server-protocol/src/lib.rs`
- `codex-rs/app-server-protocol/src/export.rs`
- `sdk/typescript/README.md`
- `sdk/typescript/src/exec.ts`
- `sdk/python/README.md`
- `sdk/python/src/codex_app_server/client.py`
- `sdk/python/src/codex_app_server/async_client.py`

## app-server 线判断

### 1. app-server 是正式控制平面，不只是内部 hack
它支持：
- stdio
- websocket
- remote-control
- in-process

说明它不是给单一 UI 用的，而是一个明确的服务面。

### 2. `in_process` 仍保留 app-server 语义
这非常关键。

说明 OpenAI 的思路不是：
> in-process 就绕开协议

而是：
> 即使在进程内，也尽量维持 app-server contract

### 3. `app-server-client` 是面向调用方的 facade
它试图屏蔽：
- remote vs in-process
- transport 差异
- queue / backpressure

也就是说，这层在承担 **嵌入式调用方友好接口** 的角色。

### 4. `app-server-protocol` 是真正的共享契约层
它负责：
- v1/v2 类型
- schema export
- Rust/TS/JSON 之间的契约桥接

这说明 app-server 线是有完整 contract 意识的。

## SDK 线判断

### TypeScript SDK
TypeScript README 明确说：
- 它 wrap 的是 `@openai/codex` CLI
- 通过 stdin/stdout 交换 JSONL event

也就是说，TS SDK 更像：
> “把 CLI 程序化”

### Python SDK
Python README 和 client 代码说明：
- 它通过 `codex app-server` 走 JSON-RPC v2 over stdio

也就是说，Python SDK 更像：
> “把 app-server 程序化”

## 设计判断

### 1. app-server 才是更像长期稳定嵌入面的方向
因为它有：
- protocol
- client facade
- in-process/remote 双态
- schema export

### 2. TypeScript SDK 走 CLI exec 说明实际产品还保留务实路径
这不一定是坏事，反而说明 OpenAI 在兼顾：
- 产品快速可用
- 逐步收敛正式嵌入接口

### 3. 当前存在“嵌入面分叉”现象
这非常值得后续继续观察。

因为它意味着：
- 还没有一个绝对统一的 embed surface
- 不同生态可能还在吃不同层

## 当前边界判断

- `app-server`：控制平面 / RPC 服务层
- `app-server-client`：调用方 facade
- `app-server-protocol`：契约层
- TS SDK：CLI embedding 路线
- Python SDK：app-server embedding 路线

## 还没确认的点

1. OpenAI 长期是否会把 TS SDK 也逐步收敛到 app-server 线，还没法从当前源码直接拍板。
2. `mcp-server` 与 `app-server` 的定位差异还值得后续专门写一篇。
3. app-server 与 TUI 的耦合程度还可继续向下追更细的调用链。

## 下一步建议

继续写：
- `state / rollout / session 恢复链` 详细版
- `MCP 与 app-server 的关系`
- `interactive TUI 主链草图`
