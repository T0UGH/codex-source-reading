# Codex 源码导读手册

> 一套按 **6 个认知台阶** 重组 OpenAI Codex 内部结构的中文源码导读仓库。

这个仓库已经不再停留在早期“Phase 1 原始笔记堆积”阶段。当前正式阅读入口已经切到 **guidebook v2**，目标是把源码证据、边界判断和卷级正文组织成一条可连续阅读的架构叙事主线。

## 从哪里开始

- **在线阅读**：<https://t0ugh.github.io/codex-source-reading/>
- **总入口**：[`guidebookv2/README.md`](./guidebookv2/README.md)
- **站点首页说明**：[`docs/index.md`](./docs/index.md)

如果你是第一次进入，建议按这个顺序：

1. [卷一｜系统入口与总图](./guidebookv2/volume-1/index.md)
2. [卷二｜runtime core 主线](./guidebookv2/volume-2/index.md)
3. [卷三｜状态、恢复与持续工作线](./guidebookv2/volume-3/index.md)
4. [卷四｜控制面与 app-server 语义](./guidebookv2/volume-4/index.md)
5. [卷五｜统一执行子系统](./guidebookv2/volume-5/index.md)
6. [卷六｜审查协作与高级 runtime](./guidebookv2/volume-6/index.md)

## 这套手册在回答什么

这不是一份“按源码目录散逛”的索引，而是在回答一串连续问题：

- Codex 到底是什么系统，而不是什么系统
- 一条请求怎么真正进入 runtime core 主工作线
- rollout、replay、thread、turn、pending request 怎样让系统持续工作
- app-server 为什么更像控制面 facade，而不是另一套平行 runtime
- unified-exec 怎样把动作装成正式 execution session
- `/review`、guardian、realtime、collab、AgentControl、memories 怎样把系统推向更高层 runtime 组织

一句话说：

> **这 6 卷不是材料分桶，而是一条连续问题链：先看清整机，再看主线、持续性、控制面、执行子系统，最后看更高层 runtime。**

## 当前仓库结构

### 正式阅读入口
- `guidebookv2/`：当前正式主线入口

### 证据与研究层
- `00-index/`：总索引、阅读地图、代码路径索引
- `01-source-notes/`：高密度源码共读笔记
- `02-call-chain-drafts/`：调用链草图与主链草案
- `03-boundary-judgments/`：模块职责 / 非职责判断

### support / reference 层
- `references/guidebookv2-support/`：各卷写作卡片、production-order 等生产辅助文件
- 这层文件默认不算网站正文，只作为策划、验收、执行说明与后续回看材料

### 旧内容与迁移资产
- `00-guidebook/`：较早一轮正文资产
- `trash/2026-04-14-guidebookv2-replaced-content/`：从主入口退出后的旧内容备份

## 当前状态

当前状态可以直接理解为：

- `guidebookv2/` 已经是默认入口
- 卷一到卷六都已在仓库中落地
- 网站导航实际使用的文章视为正文层
- 未进入网站主导航、主要服务于策划/生产/执行说明的文件，默认归入 `references/` 等 support 层
- 证据层、调用链草图、边界判断层继续保留，作为正文下钻依据
- 后续工作重点更偏向：验收、轻修、结构同步、插图补充与导航整理

如果你想看更细的站点与卷级入口说明，优先读：

- [`guidebookv2/README.md`](./guidebookv2/README.md)
- [`docs/index.md`](./docs/index.md)
- [`mkdocs.yml`](./mkdocs.yml)

## 本地运行

```bash
pip install mkdocs-material pymdown-extensions
mkdocs serve
```

默认站点地址：<http://127.0.0.1:8000>

## GitHub Pages

公开站点地址：

<https://t0ugh.github.io/codex-source-reading/>
