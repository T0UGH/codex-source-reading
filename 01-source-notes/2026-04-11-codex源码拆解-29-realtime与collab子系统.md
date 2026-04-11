# Codex 源码拆解 29：realtime 与 collab 子系统

## 这篇看什么

回答：**realtime conversation 和 collab/multi-agent 子系统到底怎么分层？它们和 app-server/TUI/core 的边界在哪？**

## 先给结论

这其实是两套不同的实时能力：

1. realtime conversation
   - 面向语音/实时对话 transport
2. collab / multi-agent
   - 面向 agent 间协作、spawn、message、wait、resume、close

它们都经过 app-server/TUI，但真正 runtime 都在 core。

一句话：

> app-server/TUI 负责控制面与展示面，realtime/collab 的真实状态机都在 core。

## 关键代码路径

- `codex-rs/tui/src/app_server_session.rs`
- `codex-rs/app-server/src/codex_message_processor.rs`
- `codex-rs/core/src/realtime_conversation.rs`
- `codex-rs/core/src/tools/handlers/multi_agents_v2/*`
- `codex-rs/app-server/src/bespoke_event_handling.rs`
- `codex-rs/tui/src/chatwidget.rs`
- `codex-rs/tui/src/multi_agents.rs`

## realtime 主链

### TUI 侧
TUI 会发：
- `RealtimeConversationStart`
- `RealtimeConversationAudio`
- `RealtimeConversationText`
- `RealtimeConversationClose`

通过 `AppServerSession` 统一走 `thread_realtime_*` RPC。

### app-server 侧
`thread_realtime_*` 会：
- load thread
- ensure listener
- gate `Feature::RealtimeConversation`
- 转成 core 的：
  - `Op::RealtimeConversationStart`
  - `Op::RealtimeConversationAudio`
  - `Op::RealtimeConversationText`
  - `Op::RealtimeConversationClose`

### core 侧
真正 runtime 在 `realtime_conversation.rs`：
- start transport
- emit started / sdp / realtime / closed 事件
- route text input
- 做 fanout

这说明 realtime conversation 不是 app-server 自己实现的实时协议，而是：

> core 管实时对话 runtime，app-server 只负责协议映射。

## realtime 的一个细节：本地 WebRTC 在 TUI
这个点很重要。

TUI 里本地 WebRTC offer/session 的创建，走的是：
- `realtime-webrtc`
- 本地 app event
- 再把 SDP 交给正常 thread realtime start 路径

所以 WebRTC 这块不是 server 全包，而是：

- 本地会话准备在 TUI
- 真正 thread realtime state 在 core/app-server

这是一个很典型的前后端分工。

## collab / multi-agent 主链
collab 是另一套体系，不要和 realtime 混。

core 里有一整套事件：
- spawn begin/end
- interaction begin/end
- waiting begin/end
- close begin/end
- resume begin/end

这些主要来自：
- `multi_agents_v2/spawn.rs`
- `message_tool.rs`
- `wait.rs`
- `close_agent.rs`
- `resume_agent.rs`

也就是说 collab 不是“几条状态通知”，而是已经有完整 event model。

## app-server 怎么投影 collab
`bespoke_event_handling.rs` 会把 collab 相关 event 统一转成：

- `ThreadItem::CollabAgentToolCall`

里面会带：
- tool kind
- sender thread id
- receiver thread ids
- prompt/model/reasoning effort
- per-agent states

所以客户端看到的 collab UI，不是直接吃 core event，而是吃 app-server 做过协议归一化之后的 thread item。

## TUI 的 collab 状态不是纯 live，还是 hybrid backfill
这里很值。

TUI 虽然会实时吃 collab 通知，
但在 resume/backfill 时还会：
- `thread/loaded/list`
- `thread/read`
- 再根据 `SessionSource::SubAgent(ThreadSpawn { parent_thread_id, .. })` 重构子树

这说明 multi-agent UI 当前不是纯实时源驱动，而是：

> live notifications + loaded thread metadata backfill

这个设计很稳，因为断线/恢复时更抗丢状态。

## collaboration mode ≠ collab runtime
还有一个很容易混的概念：

- `collaboration mode`
- `collab/multi-agent runtime`

`collaboration mode` 更像：
- plan/default 之类的 preset
- turn-start 的配置/instructions 注入

而 `collab runtime` 才是：
- spawn agent
- 发消息
- wait
- close
- resume

所以这两个概念不能混在一起讲。

## 当前边界判断

- realtime conversation：实时语音/文本对话 runtime
- local WebRTC prep：TUI 侧
- realtime thread state：core 侧
- collab runtime：core 多 agent 事件系统
- collab presentation：app-server 投影成 `ThreadItem::CollabAgentToolCall`
- collab UI：TUI + loaded-thread backfill
- collaboration mode：turn-level preset，不等于 collab runtime

## 设计判断

### 1. realtime 和 collab 都是 core-first
这说明 app-server/TUI 没有偷偷演化成真正 runtime owner。

### 2. TUI 在 realtime 上承担了部分本地媒体职责
这是合理的，因为 WebRTC 本来就更偏端侧。

### 3. collab 状态做成 live + backfill hybrid 是成熟设计
不然多 agent UI 很容易在恢复时乱掉。

## 还值得继续观察的点

1. realtime transport provider 还能继续深挖更底层
2. `multi_agents` 与 `multi_agents_v2` 的差异还可补一篇
3. collab thread item 的历史回放/持久化细节后续可再拆
