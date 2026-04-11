---
title: Codex guidebook 章节与架构设计
date: 2026-04-11
status: draft
purpose: guidebook-design
---

# Codex guidebook 章节与架构设计

## 先给结论

这份 guidebook 不该按“目录树”来写，
也不该按“我看过哪些文件”来写。

更合适的写法是：

> **先讲系统分层与运行主线，再讲关键子系统，再讲函数级边界与证据。**

也就是说，guidebook 应该是一个 **三层结构**：

1. **主叙事层**：让读者先搞清楚系统怎么分层、谁拥有 runtime、主链路怎么跑
2. **子系统层**：分别拆 app-server / persistence / unified-exec / capability system / collaboration
3. **证据层**：把函数级源码共读稿放到附录或章节内“关键边界函数”小节，不让正文被细节淹没

我建议最终不要做成“一篇超长文”，而是做成 **1 个总览页 + 6 个主章节 + 1 个附录层**。

---

# 一、guidebook 的目标与读者

## 目标

这份 guidebook 要解决的不是“把仓库介绍一遍”，而是这几个问题：

1. Codex 这套系统的**真正 runtime 主体**在哪
2. TUI / CLI / app-server / core / exec / plugin / MCP 之间的**边界**是什么
3. thread / turn / rollout / unified-exec 这些关键语义对象是怎么连起来的
4. 为什么它的很多正确性不是来自一个中心大对象，而是来自很多小型 reducer / projection / reconciliation helper

## 目标读者

建议默认对准两类人：

### 读者 A：工程师 / 架构师
关心：
- 整体设计是否清楚
- 模块边界是否合理
- 关键状态机怎么运作

### 读者 B：后续源码继续阅读者
关心：
- 应该先看哪些文件
- 哪些函数是关键 pivot
- 哪里容易误读

所以 guidebook 的风格应该是：
- 正文先给系统判断
- 每章最后给“关键文件 / 关键函数 / 建议继续读哪里”

---

# 二、整体叙事结构

我建议 guidebook 的正文结构是下面这 8 个部分。

## Chapter 0. 如何阅读这份 guidebook

### 目标
先校准读者预期：
- 这不是 API 文档
- 也不是按目录的百科全书
- 而是一份围绕 runtime 主线组织的源码导读

### 该章内容
- 仓库主体在哪
- 哪些目录是壳，哪些是 runtime 主体
- 这份 guidebook 的阅读方法
- 术语表（至少先定义：thread / turn / rollout / app-server / unified-exec / plugin）

### 作用
这是入口缓冲层，避免读者一上来陷进细节。

---

## Chapter 1. 系统总图：Codex 到底是怎么分层的

### 要回答的问题
- `codex-cli/` 为什么不是主体
- `codex-rs/cli`、TUI、core、app-server、mcp-server 分别是什么
- 为什么 `ThreadManager` 才更接近 runtime owner

### 建议小节
1. 分发壳 vs 真正主体
2. CLI / TUI / core 的关系
3. app-server 的定位：控制面 facade，不是 runtime owner
4. mcp-server 的定位：toolified exposure layer
5. remote app-server：同一 contract 的 transport variation

### 这一章的核心产出
读者读完后要得到一张稳定总图：

- 壳层：`codex-cli/`、npm 打包
- 入口层：`codex-rs/cli`
- 交互层：TUI / exec / app-server client surfaces
- runtime 层：`core` + `ThreadManager`
- 暴露层：app-server / mcp-server / remote app-server

### 这一章引用的现有材料
- 01 / 02 / 03 / 04 / 05 / 09 / 10 / 13 / 20 / 23 / 31

---

## Chapter 2. 状态与恢复：Codex 为什么能把会话找回来

### 要回答的问题
- rollout、SQLite、config、checkpoint 各管什么
- 为什么 rollout 是正文真相源而不是 SQLite
- 历史恢复为什么是 replay，不是“查表拼对象”

### 建议小节
1. session state 的几种存储介质
2. rollout JSONL：durable event/body truth
3. SQLite：metadata / index / sidecar
4. 恢复链：reconstruct → replay → turn history rebuild
5. 为什么 `state_db_bridge` 不是恢复核心

### 这一章的核心产出
读者要明确：
- Codex 的持久化不是 ORM 模式
- 它更像 event/body replay + side metadata

### 这一章引用的现有材料
- 06 / 11 / 18 / 34 / 47

---

## Chapter 3. app-server 主线：listener、thread、turn 是怎么串起来的

### 要回答的问题
- listener 为什么是线程事件泵
- `thread_state_manager` 和 `thread_watch_manager` 为什么分离
- turn history 是怎么从 event 流 materialize 出来的
- app-server 为什么需要那么多小修正器

### 建议小节
1. app-server 与 `ThreadManager` 的连接关系
2. listener task：事件泵 + 协议投影入口
3. thread state / watch state 的职责拆分
4. turn materialization：active current turn + persisted rollout replay
5. pending request：store / replay / resolve / abort
6. 状态修正与竞态窗口：为什么需要很多小 helper

### 这一章的核心产出
让读者真正理解：

> app-server 的关键能力不是“重新实现 runtime”，而是把 core runtime 事件稳定投影成 thread/turn 协议视图。

### 这一章引用的现有材料
- 15 / 16 / 17 / 32 / 33 / 36 / 39 / 40 / 43 / 44 / 45 / 48 / 50 / 51 / 58

---

## Chapter 4. turn-history 语义层：从 event stream 到可展示 turn view

### 要回答的问题
- turn 是怎样被切分、恢复、修补、对外暴露的
- 显式 turn 边界和 implicit heuristics 怎么共存
- 为什么 `ThreadHistoryBuilder` 是关键语义权威

### 建议小节
1. `ThreadHistoryBuilder` 的角色
2. `build_turns_from_rollout_items(...)`：replay reducer 入口
3. `handle_event(...)`：统一语义归约总线
4. `active_turn_snapshot(...)`：当前 turn 对外投影
5. `handle_user_message(...)`：implicit turn 划分与新输入归属
6. compaction-only turn、late completion、rollback 这些兼容点

### 这一章的核心产出
让读者知道：
- turn-history 不是 event log 镜像
- 它是 client-facing semantic projection
- 它的正确性来自 reducer + 小边界函数

### 这一章引用的现有材料
- 43 / 47 / 50 / 51 / 58，必要时带 37 / 41 / 44 / 45 / 48

> 注：Chapter 3 和 Chapter 4 相关，但不建议合并。
> Chapter 3 讲 app-server 线程与协议主线，Chapter 4 专讲 turn-history 语义层。

---

## Chapter 5. unified-exec 主线：为什么它不是普通 exec 封装

### 要回答的问题
- `exec.rs` 和 `unified_exec` 为什么不是一回事
- unified-exec 比普通 exec 多出的产品语义是什么
- process session、output transcript、end event、process store 是怎么闭环的

### 建议小节
1. `exec.rs`：执行 primitive
2. `unified_exec`：sessionful agent-facing execution subsystem
3. handler / runtime / spawn / store 的主链
4. output lifecycle：receiver bytes → UTF-8-safe chunk → transcript
5. end lifecycle：success/failure packaging
6. process store 与真实进程生命周期的 reconciliation
7. exec-server / environment / sandbox 在这条链里的位置

### 这一章的核心产出
让读者清楚：

> unified-exec 的本质不是“再包一层 shell 执行”，而是把执行过程做成一个可持续交互、可恢复、可观察、可协议化的 session 子系统。

### 这一章引用的现有材料
- 08 / 12 / 25 / 27 / 35 / 38 / 42 / 46 / 49 / 52 / 53 / 54 / 55 / 56 / 57

---

## Chapter 6. capability system：skills / plugins / MCP / apps / connectors

### 要回答的问题
- plugin 为什么是更上层能力打包单元
- skills / MCP / apps / connectors 之间到底怎么关联
- mcp-client / mcp-server 为什么是两条不同链

### 建议小节
1. skills / hooks / MCP 接入层
2. plugin：shared capability packaging unit
3. plugin 安装 / 市场 / 生效链
4. rmcp-client：MCP client transport/auth/recovery wrapper
5. apps / connectors：目录态 + 运行态 + plugin 声明态融合

### 这一章的核心产出
让读者建立 capability system 的清晰抽象，而不是把 skills / MCP / plugin / apps 混成一团。

### 这一章引用的现有材料
- 07 / 14 / 19 / 21 / 26 / 30

---

## Chapter 7. 协作与高级子系统：review / guardian / realtime / collab / memories / agents

### 要回答的问题
- `/review` 和 guardian 为什么不是一套系统
- realtime 和 collab 为什么不是一回事
- memories / agents / external-agent-config 在系统里分别在哪一层

### 建议小节
1. review vs guardian
2. realtime vs collab
3. memories：startup pipeline
4. agents：session-scoped control plane
5. external-agent-config：当前更像 Claude-oriented migration infra

### 这一章的核心产出
把“高级功能名词堆”拆成几条清晰的系统线。

### 这一章引用的现有材料
- 22 / 24 / 29

---

## Chapter 8. 可观测性、传输与横切关注点

### 要回答的问题
- analytics 为什么说 reducer-centered
- model transport 为什么要分 `codex-client -> codex-api -> ModelClient`
- `backend-client` 为什么是另一条线

### 建议小节
1. analytics / telemetry 归约链
2. model transport stack
3. backend-client 的不同定位
4. 这些横切层怎么服务前面几章的 runtime 主线

### 这一章的核心产出
补齐那些不属于单一产品功能，但决定系统可用性与边界感的横切能力。

### 这一章引用的现有材料
- 28 / 31

---

# 三、附录层怎么设计

正文不要塞太多函数级细节。

函数级共读稿更适合变成两类附录：

## 附录 A. 关键函数索引
按主题列：
- app-server / turn-history 关键函数
- unified-exec 关键函数
- persistence / restore 关键函数

每个函数只保留：
- 角色一句话
- 所属主线
- 跳转到对应共读稿

## 附录 B. call-chain 草图
- interactive / TUI 主链
- exec / unified-exec 主链
- 后面如果有必要，再补 recovery / thread-resume 主链

## 附录 C. 开放问题与未确认判断
把现在 `STATUS.md` 里的 open questions 收拢进来，作为后续研究入口。

---

# 四、建议的文件组织方式

我不建议未来 guidebook 继续全堆在 `index.md` 或一个巨文件里。

更合理的是：

```text
00-guidebook/
  00-如何阅读这份导读.md
  01-系统总图与分层.md
  02-状态持久化与恢复.md
  03-app-server与thread-turn主线.md
  04-turn-history语义层.md
  05-unified-exec执行子系统.md
  06-capability-system.md
  07-协作与高级子系统.md
  08-可观测性与传输横切层.md
  appendix-A-关键函数索引.md
  appendix-B-call-chain草图索引.md
  appendix-C-开放问题.md
```

然后根目录：
- `index.md` 继续只当导航页
- guidebook 正文放 `00-guidebook/`
- `01-source-notes/` 保留为证据仓

这套结构的好处：
- 导航层、正文层、证据层分开
- 后续改正文不会破坏原始研究材料
- 可以渐进重写，不需要一次性重构全仓库

---

# 五、章节之间的依赖关系

建议按下面依赖写，不要乱跳：

```text
Chapter 1 系统总图
  -> Chapter 2 状态与恢复
  -> Chapter 3 app-server 主线
      -> Chapter 4 turn-history 语义层
  -> Chapter 5 unified-exec 主线
  -> Chapter 6 capability system
  -> Chapter 7 协作与高级子系统
  -> Chapter 8 横切层
```

关键原因：
- 没有 Chapter 1，后面边界全会漂
- 没有 Chapter 2，Chapter 3/4 的 replay 与恢复不好讲
- 没有 Chapter 3，Chapter 4 会失去 app-server 上下文
- Chapter 5 相对独立，可以和 3/4 并行阅读
- Chapter 6/7/8 更适合后置

---

# 六、每章的写作模板

为了避免后面写散，我建议每章都用统一模板：

## 1. 这一章回答什么问题
先给读者阅读目标。

## 2. 先给结论
3~8 条，直接拍板。

## 3. 系统图 / 分层图 / 主链图
优先用简单文字图，不追求花哨。

## 4. 主叙事
讲清主线，不急着铺细节。

## 5. 关键边界与 trade-off
这是最值钱的部分。

## 6. 关键文件 / 关键函数
给继续深读的人入口。

## 7. 常见误读
专门列“不要怎么理解”。

## 8. 小结
用 3~5 句收口。

---

# 七、哪些东西不该进正文主线

有几类东西不建议放进正文大段展开：

## 1. 过细的函数实现过程
比如某个 helper 的每行 match 细节。
这些应该放附录或“关键边界函数”小节。

## 2. 目录遍历式介绍
这是最容易把 guidebook 写烂的方式。

## 3. 未证实的推测性判断
可以进“开放问题”，不要直接写成正文结论。

## 4. 太多“我怎么读到这里”的过程叙述
读者不关心你的扫描轨迹，只关心系统本身。

---

# 八、当前最推荐的落地顺序

如果现在开始真正写 guidebook，我建议顺序是：

1. 先写 `00-guidebook/00-如何阅读这份导读.md`
2. 再写 `01-系统总图与分层.md`
3. 接着写 `05-unified-exec执行子系统.md`
4. 再写 `03-app-server与thread-turn主线.md`
5. 然后写 `02-状态持久化与恢复.md`
6. 再补 `04-turn-history语义层.md`
7. 最后收 capability / collaboration / 横切层

为什么不是严格按编号写：
- Chapter 1 最适合作为总图定调
- Chapter 5 和 Chapter 3/4 是目前证据最充分、最容易写出质量的两章
- Chapter 2 虽然重要，但更适合在 reader 已经有 runtime 主线后再讲

---

# 九、最终判断

如果只压一句话，我对这份 guidebook 的架构建议是：

> **把 Codex 讲成一个“runtime 主线 + 关键子系统 + 边界函数证据层”的系统，而不是一个按目录展开的源码百科。**

具体落地就是：
- `index.md` 只负责导航
- `00-guidebook/` 承担连续可读正文
- `01-source-notes/` 保留为证据仓
- 函数级材料进入附录或章节内的“关键边界函数”小节

这套设计的好处是：
- 对工程读者友好
- 对后续继续扩写友好
- 对多 session / 多 agent 接力也友好
