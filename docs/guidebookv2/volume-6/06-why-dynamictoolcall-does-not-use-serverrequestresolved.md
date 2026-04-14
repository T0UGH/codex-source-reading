---
title: 补充阅读｜DynamicToolCall 为什么不走 ServerRequestResolved
status: draft
updated: 2026-04-14
---

# 补充阅读｜DynamicToolCall 为什么不走 ServerRequestResolved

这篇是卷六里给微缺口阅读链补上的补充阅读页。

它对应仓库中的边界判断文档，核心回答是：

> **`DynamicToolCall` 不该再被理解成一个“还没迁移进 ServerRequestResolved 的遗漏点”，而更适合被理解为：在 transport / pending-request machinery 上有复用，但在产品语义上明确走 item lifecycle 的另一条线。**

## 建议怎么读

如果你是从卷六正文过来的，这篇补充阅读主要用于把 `ServerRequestResolved` 的覆盖边界彻底看清。

建议配合下面两篇一起读：

- [卷六 04｜ServerRequestResolved 覆盖面与未迁移疑点](./04-serverrequestresolved-coverage-and-gaps.md)
- [卷六 05｜guardian analytics 为何还没真正接上](./05-why-guardian-analytics-is-not-fully-wired.md)

## 原始边界判断文档

公开站点这里先给稳定入口；仓库里的原始判断文档仍保留在：

- `03-boundary-judgments/2026-04-12-DynamicToolCall为什么不走ServerRequestResolved.md`

后续如果你希望，我可以再把这篇边界判断文档也完全纳入公开导航。 
