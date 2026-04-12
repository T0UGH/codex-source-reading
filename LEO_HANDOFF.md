# LEO HANDOFF

## Goal

接手 `codex-source-reading` 仓库，**不要再把它当成“继续扫源码”的仓库**，而要把它当成：

> **一套已经完成主骨架收敛、现在进入轻收尾与少量 future-looking 补料阶段的 guidebook 仓库。**

你接手后的默认目标不是继续扩面，而是：

1. 维持现有 guidebook 主线稳定
2. 只在真正高价值的 future-looking 问题上补料
3. 避免破坏当前已经形成的导航 / 正文 / 专题 / 附录 / 微缺口结构

---

## Current repo shape

当前仓库已经不是笔记堆，而是分成了明确 6 层：

1. **navigation**
   - `index.md`
   - `STATUS.md`
2. **guidebook 主干正文**
   - `00-guidebook/00` ~ `06`
3. **topic deep-dives**
   - `00-guidebook/07` ~ `10`
4. **appendices**
   - `00-guidebook/11` ~ `13`
5. **micro-gap docs**
   - `00-guidebook/14`
   - `00-guidebook/15`
   - `03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`
   - `03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`
6. **evidence layer**
   - `01-source-notes/`

当前状态不是“还缺大骨架”，而是：

> **主骨架已闭合，专题已补到位，附录和微缺口已足够支撑后续 selective continuation。**

---

## What is already stable

### Repo / layering level
- `codex-cli/` 只是 distribution shell
- `codex-rs/cli` 是统一入口层
- `core` 是 runtime aggregation center
- `ThreadManager` 比 app-server 更接近 runtime owner
- `app-server` 是 control-plane facade，不是另一套平行 runtime
- `mcp-server` 是 toolified exposure layer，不是 app-server 同类物

### Persistence / recovery level
- rollout JSONL 是正文真相源 / durable truth source
- SQLite 更偏 metadata / index / sidecar
- 恢复本质上是 replay + semantic rebuild，不是 ORM 式查表重建

### app-server / turn-history level
- listener task 是 thread-event pump
- `bespoke_event_handling` 是协议投影层
- `thread_state_manager` / `thread_watch_manager` 是有意拆分：内部协调 vs outward status projection
- turn-history 是 client-facing semantic projection，不是 event log 镜像
- `ThreadHistoryBuilder` 是当前最清晰的 turn 语义权威层
- pending server request replay / resolve / abort 已经可以作为 thread 语义的一部分来理解

### unified-exec level
- `exec.rs` 是 primitive layer
- `unified_exec` 是 sessionful, agent-facing execution subsystem
- unified-exec 主线已基本闭环：
  - request assembly
  - runtime adaptation
  - spawn/sessionization
  - store
  - output watcher
  - transcript authority
  - success/failure end packaging
  - process-store reconciliation
- transcript 是 final aggregated output 的 primary truth
- live delta streaming 是 budget-limited 的 secondary channel

### Request semantics level
这条线现在非常关键，而且已经基本收口：

- callback-map resolution ≠ `ServerRequestResolved`
- `ServerRequestResolved` 只覆盖一批 **V2 thread-scoped interactive requests**
- 当前 `ServerRequest` 枚举已可稳定归类为：
  - **5 resolved**
  - **1 semantic split**：`DynamicToolCall`
  - **1 global bridge**：`ChatgptAuthTokensRefresh`
  - **2 legacy holdouts**：`ApplyPatchApproval` / `ExecCommandApproval`

也就是说：

> **这条线当前不是“分类不清”，而是“未来是否还会变”。**

### review / guardian / advanced systems level
- `/review` 和 guardian 是两套系统：user review workflow vs approval reviewer infrastructure
- guardian protocol / UI 观察面已存在
- guardian analytics schema / reducer / client API 已存在
- 但 guardian analytics 的 **runtime emission 仍未接上**
- realtime 和 collab 是两套 live control plane
- `AgentControl` 是 rooted thread-tree multi-agent control plane
- memories 是 startup pipeline，不是普通 turn-time helper
- external-agent-config 当前更像 migration-oriented infra

---

## What has already been done recently

### 1. Request gap cleanup 已做完一轮
已经落盘：
- `00-guidebook/14-ServerRequestResolved覆盖面与未迁移疑点.md`
- `03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`
- `03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`

当前结论：
- `DynamicToolCall` 不该再写成“未迁移洞”
- 更准确是：**transport 上复用 request machinery，语义上走 item lifecycle**

### 2. guardian analytics gap 已单独成文
已落盘：
- `00-guidebook/15-guardian-analytics为何还没真正接上.md`

当前结论：
- protocol/UI observability 在
- analytics schema/reducer/client 在
- runtime emission 不在

### 3. cross-link / terminology 已做过一轮
已完成：
- `00-guidebook/10` ~ `15` 以及 `11` / `12` / `13` 的高价值 `相关阅读` / `延伸阅读`
- 术语漂移清理过一轮（如 `façade` → `facade`）
- open questions wording 已与 request-shape 收口结果对齐

当前判断：

> **后面不是“大面积补链”阶段，而是轻 polish 阶段。**

---

## What Leo should NOT do

### 不要再做这些
- 不要再 breadth-first 横向扫描仓库
- 不要为了“显得继续推进”而机械增加 source notes
- 不要把已经收口的问题重新打回 open question（尤其 `DynamicToolCall`）
- 不要把 guidebook 写回“文件清单”或“百科堆砌”
- 不要大面积改章节骨架、目录顺序、命名体系，除非发现明显结构错误
- 不要再把 cross-link 当成主工作流，只能做局部修补

换句话说：

> **别把仓库重新拖回扩面阶段。**

---

## What Leo SHOULD do next

### 默认优先级 1：future-looking 小专题，而不是旧问题返工
当前最值继续写的，不是再补骨架，而是这种问题：

1. **deprecated V1 request path 会不会被真正移除**
   - `ApplyPatchApproval`
   - `ExecCommandApproval`
   - 看它们是长期 compatibility seam，还是明确走向移除

2. **future/new request shape 会不会打破当前 5+1+1+2 分类**
   - 这不是回头查旧结论，而是盯未来演化风险

3. **guardian analytics runtime emission 的真正发射边界**
   - 如果还补一个最小高价值缺口，这仍然是值的

### 默认优先级 2：轻 polish
如果不补新专题，就做很轻的事情：
- 局部 cross-link 修补
- 局部术语统一
- 明显重复段落收紧
- `STATUS.md` / `LEO_HANDOFF.md` 小同步

### 默认优先级 3：只在必要时补证据
只有出现下面情况才值得回 evidence layer：
- 正文结论不稳
- 新专题需要补最小证据
- 有明确 future drift 信号

---

## Recommended immediate reading order for Leo

如果 Leo 现在起手，建议顺序：

1. `index.md`
2. `STATUS.md`
3. `LEO_HANDOFF.md`（这份）
4. `00-guidebook/13-open-questions与后续深挖方向.md`
5. `03-boundary-judgments/2026-04-12-app-server-request-shape分类与收口.md`
6. `00-guidebook/15-guardian-analytics为何还没真正接上.md`

如果要补 request / app-server future-looking 问题，再回查：
- `00-guidebook/14-ServerRequestResolved覆盖面与未迁移疑点.md`
- `03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`
- `00-guidebook/03-app-server与thread-turn主线.md`

如果要补 guardian analytics，再回查：
- `00-guidebook/09-review工作流与guardian审查基础设施.md`
- `00-guidebook/15-guardian-analytics为何还没真正接上.md`
- 对应 core / analytics 源码

---

## Open risks / edges Leo should keep in mind

这些点正文里仍然要谨慎：

- `ServerRequest` 当前分类已经清晰，但未来新增 request shape 可能改写这套分类
- deprecated V1 request path 是否真的会被移除，当前还没有定论
- guardian analytics 在生产里到底有没有别处发射 runtime event，当前仍未完全打死
- SQLite search/indexing 会不会继续增强
- `interface.capabilities` 会不会从 presentation layer 演化成 runtime-significant layer
- app-server 会不会继续吞并更多 embedding surface
- external-agent-config 会不会从 Claude-first migration infra 演化成正式通用层

---

## Collaboration discipline

### Git discipline
这个仓库默认多人 / 多 agent 并发。

每次收尾都要遵守：
1. `git status --short`
2. `git pull --rebase --autostash`
3. `git add ...`
4. `git commit -m "..."`
5. `git push`

如果第一次 pull/push 因 SSL / 网络抖动失败：
- 先重试一次
- 不要立刻误判成权限问题

### State discipline
如果 Leo 再推进一轮，结束时必须至少更新：
- `STATUS.md`
- `LEO_HANDOFF.md`

并且要把这三件事写清楚：
1. **目标**
2. **这轮新稳定下来的判断**
3. **下一轮最值的一刀**

---

## Acceptance bar for Leo takeover

Leo 接手后，算“接稳了”，至少满足：

- 没有把仓库重新拖回 broad scan
- 没有重新打开已经收口的问题
- 能基于当前结构继续做 selective continuation
- 新增内容要么提供了新的稳定判断，要么明确改善了未来接手成本
- 收尾时 repo 状态、下一刀、风险边界都写回仓库，而不是只留在对话里

---

## One-sentence directive

> **把这个仓库当成“已完成主架构收敛的 guidebook 仓”，不是“继续扫源码仓”；默认做 selective continuation，不做重新扩面。**
