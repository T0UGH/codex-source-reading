---
title: Codex guidebook 13｜Open questions 与后续深挖方向
date: 2026-04-12
status: draft
purpose: appendix
---

# 13｜Open questions 与后续深挖方向

这份附录只做两件事：

1. 把当前还没打死的 open questions 显式列出来
2. 给出后续继续深挖时，哪几刀最值

它不是问题清单大全，而是：

> **下一位继续读源码的人，最该从哪里接。**

---

## 1. app-server / thread / request 线还没完全打死的问题

### 1. 哪些 request type 天生就不走 `ServerRequestResolved`，哪些只是还没迁完
这是目前很值继续确认的一点。

为什么重要：
- 它关系到 pending request replay / resolve 语义是否统一
- 也关系到 app-server 协议面到底是不是已经完整收口

继续建议：
- 沿 `ResolveServerRequest` 相关路径反查所有 request 类型
- 对照 app-server protocol 和 `bespoke_event_handling` 的投影面

---

## 2. 状态与恢复线还没打死的问题

### 2. SQLite 搜索 / indexing 会不会继续扩张
现在的判断是：
- rollout 是正文真相源
- SQLite 是 metadata/index/sidecar

但还不完全确定：
- SQLite 之后会不会从“索引层”继续长成更强的查询层
- 当前 substring/filter 方案是不是过渡阶段

为什么重要：
- 这关系到未来 repo 的状态架构会不会再往 DB 中心偏一点

继续建议：
- 追 `state/src/runtime` 和搜索相关模块
- 看 listing/search 调用点是否还在扩张

---

## 3. capability system 线还没打死的问题

### 3. `interface.capabilities` 以后会不会变成 runtime-significant
现在它更像 presentation/metadata 面。

还不确定的是：
- 未来它会不会从“展示能力描述”变成真正参与 runtime gating 的结构

为什么重要：
- 这关系到 capability system 会不会再往 declarative runtime contract 演化

继续建议：
- 顺 `plugin` / `apps` / `connectors` / app-server capabilities 投影继续查

---

## 4. app-server 长期边界还没打死的问题

### 4. app-server 会不会成为所有 SDK 的主 embedding surface
当前判断是：
- app-server 已经是主要 control plane facade
- TUI 越来越往它收敛

但还没完全确认的是：
- 所有 SDK 最终是否都会完全走 app-server contract
- 还是会长期保留更多 direct-core edge path

为什么重要：
- 这关系到 guidebook 后面应不应该更强地把 app-server 写成第一公民

继续建议：
- 查 SDK / remote-app-server / app-server client usage 扩张趋势

---

## 5. TUI 边界还没完全打死的问题

### 5. TUI 会在多长时间内继续保留 hybrid app-server + legacy-core edge architecture
当前判断是：
- 主体已向 app-server 收敛
- 但边缘还残留 legacy helper

还没完全确认的是：
- 这些 edge 是临时兼容，还是长期保留的 pragmatic architecture

为什么重要：
- 这会影响之后“系统总图”那章要不要进一步改写成更强的 app-server 中心视角

---

## 6. review / guardian 线还没打死的问题

### 6. guardian analytics 到底在哪里真正发射
现在的判断已经很明确：
- schema 有
- reducer 有
- 但 runtime emission 似乎没完全接上

还需要确认的是：
- 生产链路是不是在别处发
- 还是确实只是 analytics scaffold 超前于接线

为什么重要：
- 这直接关系到 guardian 是不是已经从“逻辑系统”进化到“完整可观测系统”

继续建议：
- 继续反查 `track_guardian_review(...)` 全部调用面
- 查 analytics 事件生成边界是不是在别的 crate

---

## 7. transport 线还没打死的问题

### 7. WebSocket transport 未来会不会抽到比 `codex-api` 更低的一层
现在很明确：
- HTTP substrate 在 `codex-client`
- WebSocket 还主要留在 `codex-api` endpoint 层

还不确定的是：
- 这是不是有意为之的长期边界
- 还是未来会抽象成更通用 transport substrate

为什么重要：
- 这关系到 `codex-client` / `codex-api` 的长期职责边界是否稳定

---

## 8. memories 线还没打死的问题

### 8. phase2 会不会一直保持 singleton-global
现在 phase2 更像：
- 单例全局 consolidation job
- 通过 watermark 和 selection baseline 做控制

还不确定的是：
- 后面会不会分片
- 会不会按 namespace/project/user 做 partitioned consolidation

为什么重要：
- 这关系到 memories 是不是会从“单全局后台作业”进化成更强的数据管线

---

## 9. external-agent-config 线还没打死的问题

### 9. external-agent-config 会不会一直保持 Claude-first
当前判断是：
- 明显偏 migration / compatibility
- 明显带 Claude-oriented 历史痕迹

还不确定的是：
- 它以后会不会真的变成通用 external-agent migration layer
- 还是长期只是一个兼容入口

为什么重要：
- 这决定它是临时边缘层，还是会成长为一个正式系统

---

## 10. 如果继续深挖，下一刀最值的顺序

如果未来还要继续推进，我建议按下面顺序，不要乱铺：

### 第一优先级
1. guardian analytics emission
2. `ServerRequestResolved` 覆盖面
3. app-server 是否继续吞并更多 embedding surface

### 第二优先级
4. SQLite search/indexing 演化方向
5. `interface.capabilities` 是否 runtime-significant
6. WebSocket transport 是否继续下沉抽象

### 第三优先级
7. memories phase2 global/splitting 演化
8. external-agent-config 的长期定位

---

## 11. 如果继续写文，最值得补的文档类型

当前不建议再继续堆大段 guidebook 主章。更值的是：

### 1. 附录型索引
- key function index
- call-chain index
- subsystem continuation map

### 2. 小专题勘误/状态文
例如：
- `guardian analytics 当前接线状态`
- `app-server unresolved request matrix`
- `TUI 剩余 legacy-core edge 列表`

### 3. 有明确缺口时再补函数级共读
标准是：
- 正文已经讲不清
- 附录索引也无法满足
- 这时再补 1~2 篇函数级 note

---

## 最后一句判断

到现在为止，Codex 这套材料已经不是“还不知道要看什么”的状态了。

真正重要的已经变成：

> **哪些地方已经足够稳定，可以开始收束；哪些地方还没完全打死，值得继续下刀。**

这份附录就是给这个问题的答案。
