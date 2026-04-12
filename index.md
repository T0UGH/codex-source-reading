# Codex 源码导读 Index

这不是文件堆砌索引，
而是这套材料的**总导航页**。

现在仓库已经不再处在“继续横向扫源码”的阶段，而是进入了 **guidebook 重组阶段**：

- `00-guidebook/` 负责连续正文
- `01-source-notes/` 负责证据层
- `STATUS.md` 负责状态交接

---

## 先给结论

如果你第一次进入这个仓库，先读这 4 个文件：

1. [STATUS.md](STATUS.md)
2. [00-guidebook/00-如何阅读这份导读.md](00-guidebook/00-如何阅读这份导读.md)
3. [00-guidebook/01-系统总图与分层.md](00-guidebook/01-系统总图与分层.md)
4. [03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md](03-boundary-judgments/2026-04-11-codex模块边界判断-v1.md)

读完后，应该先建立这些总判断：

- `codex-cli/` 只是分发壳，不是主体
- `codex-rs/cli` 是统一入口层
- `core` 是 runtime 聚合中心
- `ThreadManager` 比 app-server 更接近 runtime owner
- `app-server` 是控制面 facade，不是另一套 runtime
- 这套系统的很多一致性来自一批 reducer / projection / reconciliation helper，而不是一个中心大状态机

---

## 正文阅读顺序

### Chapter 0
- [00-guidebook/00-如何阅读这份导读.md](00-guidebook/00-如何阅读这份导读.md)

### Chapter 1
- [00-guidebook/01-系统总图与分层.md](00-guidebook/01-系统总图与分层.md)

### Chapter 2
- [00-guidebook/02-状态持久化与恢复.md](00-guidebook/02-状态持久化与恢复.md)

### Chapter 3
- [00-guidebook/03-app-server与thread-turn主线.md](00-guidebook/03-app-server与thread-turn主线.md)

### Chapter 4
- [00-guidebook/04-turn-history语义层.md](00-guidebook/04-turn-history语义层.md)

### Chapter 5
- [00-guidebook/05-unified-exec执行子系统.md](00-guidebook/05-unified-exec执行子系统.md)

### Chapter 6
- [00-guidebook/06-capability与高级子系统.md](00-guidebook/06-capability与高级子系统.md)

---

## 推荐阅读路径

### 路径 A：先建立全局认知
按这个顺序：

1. `00-如何阅读这份导读`
2. `01-系统总图与分层`
3. `02-状态持久化与恢复`
4. `03-app-server与thread-turn主线`
5. `05-unified-exec执行子系统`
6. `06-capability与高级子系统`

适合：
- 第一次进入 Codex 源码
- 想快速建立整体架构认知

### 路径 B：只看 app-server / thread / turn 主线
按这个顺序：

1. `01-系统总图与分层`
2. `02-状态持久化与恢复`
3. `03-app-server与thread-turn主线`
4. `04-turn-history语义层`

适合：
- 想搞清 listener / thread state / turn history
- 想理解 app-server 为什么能恢复、rejoin、修正状态

### 路径 C：只看执行主线
按这个顺序：

1. `01-系统总图与分层`
2. `05-unified-exec执行子系统`
3. `02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md`

适合：
- 想搞清 exec 和 unified-exec 的差别
- 想看 output / transcript / end-event / process-store 怎么闭环

---

## 证据层入口

如果你已经读完正文，想追到证据层，可以按专题回看这些目录：

### 1. 架构拆解批次
- [00-index/00-Codex源码拆解总索引.md](00-index/00-Codex源码拆解总索引.md)
- [00-index/01-代码路径索引-入口与分发.md](00-index/01-代码路径索引-入口与分发.md)

### 2. 函数级源码共读
重点函数级笔记在：
- `01-source-notes/2026-04-11-codex源码共读-32-...` 到 `58-...`

### 3. 调用链草图
- [02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md](02-call-chain-drafts/2026-04-11-codex主链草图-interactive-TUI链.md)
- [02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md](02-call-chain-drafts/2026-04-11-codex主链草图-exec链.md)

---

## 现在的建议

这份仓库接下来不该再继续横向铺更多 source notes。

更合理的下一步是：
1. 以 guidebook 正文为主阅读层
2. 把 `01-source-notes/` 当作证据层回查
3. 后续只在 guidebook 出现真实缺口时，才补新的函数级或专题笔记

一句话说：

> **Phase 1 的“拆散、验证、补证据”已经完成，当前阶段应该以“重组、讲顺、形成连续正文”为主。**
