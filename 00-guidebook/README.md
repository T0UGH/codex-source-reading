---
title: Codex Guidebook 写作方案
date: 2026-04-11
status: draft
---

# Codex Guidebook 写作方案

## 1. 这份文档解决什么问题

这不是正文，
而是 **Codex guidebook 的整体设计文档**。

它主要回答 4 个问题：

1. 这份 guidebook 应该怎么组织，而不是写成目录百科
2. 应该分成哪些章节，每章回答什么问题
3. 现有 `01-source-notes/` 应该怎样映射到 guidebook 正文
4. 后续写作时，什么内容该进正文，什么内容该留在证据层

一句话说：

> 先把 guidebook 的叙事结构设计对，再写正文。

---

## 2. 总体设计原则

### 2.1 不按目录树写

这份 guidebook 不应该按仓库目录展开，
否则很容易写成：
- 文件清单
- 模块百科
- 资料堆砌

这类文档的典型问题是：
- 信息很多，但没有主线
- 读者知道“有哪些目录”，却不知道“系统怎么运作”

更合理的方式是：

> 按系统主线、状态流和关键边界来写。

---

### 2.2 不按调研时间顺序写

`01-source-notes/` 是研究过程产物，
它的价值是：
- 证明某个判断
- 提供局部源码证据
- 保留继续深挖的抓手

但它不适合作为正文顺序。

guidebook 正文应该是：
- 先给系统判断
- 再讲主链
- 最后给证据入口

而不是把研究过程原样平铺。

---

### 2.3 正文与证据层分离

这套材料现在已经明显分成三层：

#### 导航层
- `index.md`
- 作用：告诉读者先看什么、按哪条线看

#### 正文层
- `00-guidebook/`
- 作用：连续叙事、讲清系统

#### 证据层
- `01-source-notes/`
- 作用：保存源码阅读证据、关键函数拆解、局部判断

guidebook 成型后，这三层关系应该稳定下来：

> `index.md` 负责导航，`00-guidebook/` 负责讲故事，`01-source-notes/` 负责给证据。

---

## 3. guidebook 的目标读者

### 3.1 工程读者

这类读者关心：
- 系统怎么分层
- 核心 runtime 在哪里
- 关键状态机和数据流怎么运作
- 为什么某些设计是这样，而不是那样

### 3.2 后续继续读源码的人

这类读者关心：
- 哪些文件最关键
- 哪些函数是 pivot
- 哪些地方最容易误读
- 从哪里继续深挖最划算

所以这份 guidebook 不该写成纯概念文，
也不该写成纯源码注释拼接。

更合适的风格是：
- 先给判断
- 再给主线
- 最后给关键文件 / 关键函数入口

---

## 4. guidebook 的推荐总体结构

我建议最终正文结构是：

1. 如何阅读这份导读
2. 系统总图与分层
3. 状态、持久化与恢复
4. app-server / thread / turn 主线
5. turn-history 语义层
6. unified-exec 执行子系统
7. capability system
8. 协作与高级子系统
9. 可观测性与横切层
10. 附录层

其中：
- 第 1 章是入口
- 第 2~9 章是正文
- 附录层承接函数级证据

---

## 5. 建议章节设计

## Chapter 0. 如何阅读这份导读

### 目标
先帮读者建立阅读预期，避免一上来掉进细节。

### 要回答的问题
- 这份 guidebook 不是 API 手册，也不是目录索引
- 仓库主体在哪
- 哪些目录是壳，哪些目录是 runtime 主体
- 这份材料该怎么读

### 应包含的内容
- 阅读目标
- 术语表（至少包括：thread / turn / rollout / app-server / unified-exec / plugin）
- 章节关系说明
- 推荐阅读路径

### 这一章的作用
相当于 guidebook 的“启动页”。

---

## Chapter 1. 系统总图与分层

### 要回答的问题
- `codex-cli/` 为什么只是分发壳
- `codex-rs/cli`、TUI、core、app-server、mcp-server 分别处在哪一层
- 为什么 `ThreadManager` 更接近 runtime owner

### 推荐小节
1. 分发壳与主体代码
2. CLI / TUI / core 的关系
3. app-server 的定位
4. mcp-server 的定位
5. remote app-server 的定位

### 读完后应该得到的判断
- `codex-cli/` 只是壳
- `codex-rs/cli` 是统一入口层
- `core` 是 runtime 聚合中心
- `app-server` 是控制面 facade
- `mcp-server` 是 toolified exposure layer

### 主要引用材料
- 01 / 02 / 03 / 04 / 05 / 09 / 10 / 13 / 20 / 23 / 31

---

## Chapter 2. 状态、持久化与恢复

### 要回答的问题
- config、rollout、SQLite、checkpoint 各自是什么
- 为什么 rollout 是正文真相源
- 为什么恢复不是“查库组装对象”，而是 replay

### 推荐小节
1. 持久化介质总览
2. rollout JSONL 的角色
3. SQLite 的角色
4. 恢复链路：reconstruct → replay → rebuild
5. `state_db_bridge` 为什么不是恢复核心

### 读完后应该得到的判断
- rollout 是 durable body truth
- SQLite 是 metadata / index / sidecar
- 恢复主要靠 replay，而不是 ORM 式重建

### 主要引用材料
- 06 / 11 / 18 / 34 / 47

---

## Chapter 3. app-server / listener / thread 主线

### 要回答的问题
- app-server 怎样接上 `ThreadManager`
- listener 为什么是事件泵
- thread state 和 watch state 为什么拆开
- pending request 为什么能 replay / resolve / abort

### 推荐小节
1. app-server 与 runtime 的关系
2. listener task 的职责
3. `bespoke_event_handling` 的位置
4. thread state / watch state 的职责划分
5. pending request 生命周期
6. 为什么 app-server 依赖很多小修正器

### 读完后应该得到的判断
- app-server 不是 runtime owner
- 它的核心职责是把 runtime event 投影成稳定协议面

### 主要引用材料
- 15 / 16 / 17 / 32 / 33 / 36 / 39 / 40 / 43 / 44 / 45 / 48

---

## Chapter 4. turn-history 语义层

### 要回答的问题
- turn 是怎样从 event stream 里被切出来的
- 显式 turn 边界和 implicit heuristics 怎么共存
- 为什么 `ThreadHistoryBuilder` 是 turn 语义权威

### 推荐小节
1. `ThreadHistoryBuilder` 的角色
2. replay reducer：`build_turns_from_rollout_items(...)`
3. 统一归约入口：`handle_event(...)`
4. 当前 turn 投影：`active_turn_snapshot(...)`
5. implicit turn 收口：`handle_user_message(...)`
6. compaction、late completion、rollback 等兼容语义

### 读完后应该得到的判断
- turn-history 不是 event log 镜像
- 它是 client-facing semantic projection
- 正确性来自 reducer + 一批小型边界函数

### 主要引用材料
- 43 / 47 / 50 / 51 / 58
- 必要时补 37 / 41 / 44 / 45 / 48

---

## Chapter 5. unified-exec 执行子系统

### 要回答的问题
- `exec.rs` 和 `unified_exec` 的边界是什么
- unified-exec 比普通 exec 多出了哪些产品语义
- output、transcript、end event、process store 如何闭环

### 推荐小节
1. `exec.rs`：primitive layer
2. `unified_exec`：sessionful execution subsystem
3. handler / runtime / spawn / store 主链
4. output lifecycle：bytes → UTF-8-safe chunk → transcript
5. end lifecycle：success / failure packaging
6. process store 与真实生命周期对账
7. exec-server / environment / sandbox 的位置

### 读完后应该得到的判断
- unified-exec 不是“多包一层 exec”
- 它是会话化、可恢复、可交互、可协议化的执行子系统

### 主要引用材料
- 08 / 12 / 25 / 27 / 35 / 38 / 42 / 46 / 49 / 52 / 53 / 54 / 55 / 56 / 57

---

## Chapter 6. capability system

### 要回答的问题
- skills / plugins / MCP / apps / connectors 到底怎么分层
- plugin 为什么像更上层 capability packaging unit
- mcp-client / mcp-server 为什么是两条线

### 推荐小节
1. skills / hooks / MCP 接入层
2. plugin 的抽象地位
3. plugin 安装与生效链
4. rmcp-client 的角色
5. apps / connectors 的系统位置

### 读完后应该得到的判断
- plugin 是能力打包单元
- skills / MCP / apps / connectors 不是同一层概念

### 主要引用材料
- 07 / 14 / 19 / 21 / 26 / 30

---

## Chapter 7. 协作与高级子系统

### 要回答的问题
- `/review` 和 guardian 为什么不是一套系统
- realtime 和 collab 为什么不是一回事
- memories / agents / external-agent-config 各自处在哪层

### 推荐小节
1. review vs guardian
2. realtime vs collab
3. memories：startup pipeline
4. agents：session-scoped control plane
5. external-agent-config：当前更像迁移基础设施

### 读完后应该得到的判断
- 这些“高级功能名词”背后是不同子系统，不该混读

### 主要引用材料
- 22 / 24 / 29

---

## Chapter 8. 可观测性与横切层

### 要回答的问题
- analytics 为什么说 reducer-centered
- model transport 为什么是分层栈
- `backend-client` 为什么不该跟 model transport 混成一层

### 推荐小节
1. analytics / telemetry
2. model transport stack
3. backend-client
4. 这些横切层如何服务前面正文主线

### 读完后应该得到的判断
- 这些不是边角料，而是影响整个系统边界感的横切能力

### 主要引用材料
- 28 / 31

---

## 6. 附录层怎么设计

正文不要塞过多函数级细节。

更合理的做法是把函数级材料下沉为附录：

### 附录 A：关键函数索引
按主题分：
- app-server / turn-history
- unified-exec
- persistence / restore

每项只保留：
- 一句话角色定义
- 所属主线
- 对应共读稿链接

### 附录 B：call-chain 草图索引
- interactive / TUI 主链
- exec / unified-exec 主链
- 后续如果有必要，再补 recovery / resume 主链

### 附录 C：开放问题
把 `STATUS.md` 里的 open questions 收进来，作为继续研究入口。

---

## 7. 建议的文件组织

建议 guidebook 正文最终落到：

```text
00-guidebook/
  README.md
  00-如何阅读这份导读.md
  01-系统总图与分层.md
  02-状态持久化与恢复.md
  03-app-server与thread-turn主线.md
  04-turn-history语义层.md
  05-unified-exec执行子系统.md
  06-capability-system.md
  07-协作与高级子系统.md
  08-可观测性与横切层.md
  appendix-A-关键函数索引.md
  appendix-B-call-chain草图索引.md
  appendix-C-开放问题.md
```

其中：
- `index.md` 保持为导航页
- `00-guidebook/` 存正文
- `01-source-notes/` 保持为证据仓

这套结构的好处是：
- 导航、正文、证据分层稳定
- 以后重写正文不影响原始调研成果
- 多 session / 多 agent 协作也不容易打架

---

## 8. 章节依赖关系

建议依赖关系如下：

```text
Chapter 1 系统总图与分层
  -> Chapter 2 状态、持久化与恢复
  -> Chapter 3 app-server / listener / thread 主线
      -> Chapter 4 turn-history 语义层
  -> Chapter 5 unified-exec 执行子系统
  -> Chapter 6 capability system
  -> Chapter 7 协作与高级子系统
  -> Chapter 8 可观测性与横切层
```

关键点：
- Chapter 1 必须先写，负责定边界
- Chapter 3 和 Chapter 5 是两条最重要主线
- Chapter 4 是 Chapter 3 的深入层
- Chapter 6 / 7 / 8 更适合后置

---

## 9. 每章建议使用统一模板

为了避免后面越写越散，建议每章都用同一模板：

### 1. 这一章回答什么问题
先校准阅读目标。

### 2. 先给结论
直接拍板，不绕。

### 3. 系统图 / 分层图 / 主链图
优先用简单图，不追求花哨。

### 4. 主叙事
讲主线，不先掉进局部细节。

### 5. 关键边界与 trade-off
这部分最值钱。

### 6. 关键文件 / 关键函数
给继续深读的人入口。

### 7. 常见误读
单独列出来，避免读者走偏。

### 8. 小结
3~5 句收口。

---

## 10. 什么内容不该进正文主线

以下内容不建议大段放进正文：

### 10.1 逐目录罗列
会把 guidebook 写回百科全书。

### 10.2 过细的函数逐行解释
这些适合放附录或章节内“关键边界函数”小节。

### 10.3 未验证的推测
应放到开放问题，不要伪装成结论。

### 10.4 研究过程流水账
读者关心的是系统，不关心调研轨迹。

---

## 11. 推荐写作顺序

如果从现在开始真正落正文，我建议顺序是：

1. `00-如何阅读这份导读.md`
2. `01-系统总图与分层.md`
3. `05-unified-exec执行子系统.md`
4. `03-app-server与thread-turn主线.md`
5. `02-状态持久化与恢复.md`
6. `04-turn-history语义层.md`
7. `06-capability-system.md`
8. `07-协作与高级子系统.md`
9. `08-可观测性与横切层.md`
10. 各附录

这样排的原因：
- 先把总图定住
- 再优先写证据最充分的两条主线：unified-exec、app-server/thread
- 再补持久化与 turn-history
- 最后收 capability / collaboration / 横切层

---

## 12. 当前阶段结论

现在已经不适合继续横向增加很多新的 source notes 了。

更合理的阶段切换是：

> Phase 1 已经完成“拆散、验证、补证据”；
> 下一阶段应该转向“重组、讲顺、形成一套可连续阅读的 guidebook”。

所以从这个节点开始：
- `index.md` 继续做导航
- `00-guidebook/` 开始承担正文
- `01-source-notes/` 作为证据层继续保留，但不再当主阅读层

这才是下一阶段最对的推进方式。
