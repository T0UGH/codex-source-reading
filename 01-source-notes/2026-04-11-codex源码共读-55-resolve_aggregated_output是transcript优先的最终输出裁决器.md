---
title: resolve_aggregated_output(...) 是 transcript 优先的最终输出裁决器
date: 2026-04-11
tags:
  - Codex
  - 源码共读
  - unified-exec
  - transcript
  - aggregated-output
source_files:
  - /Users/wangguiping/workspace/codex/codex-rs/core/src/unified_exec/async_watcher.rs
status: done
---

# Codex 源码共读 55：resolve_aggregated_output(...) 是 transcript 优先的最终输出裁决器

## 这篇看什么

`resolve_aggregated_output(...)` 很短：
- 锁 transcript
- 如果空就回 fallback
- 不空就把 transcript bytes 转成字符串

但这个函数很值，因为 unified-exec 最终 end payload 到底信谁，实际就是它在拍板。

## 先给主结论

如果先留一句话，我会写：

> `resolve_aggregated_output(...)` 的本质不是一个小工具函数，而是 unified-exec 最终输出来源的裁决器：只要 transcript 已经保留到任何字节，它就以 transcript 为终态 aggregated output 的权威来源；只有 transcript 完全为空时，才回退到调用方提供的 fallback 文本。也就是说，Codex 在最终输出语义上明确偏向 watcher/streaming 路径累积出来的事实，而不是临时调用现场手上的局部文本。

再压缩一点：

> **最终输出优先信 transcript，fallback 只是兜底。**

---

## 第一层：它解决的不是“怎么转字符串”，而是“最终信哪个来源”

实现很小：
- `if retained_bytes == 0 => fallback`
- else `String::from_utf8_lossy(transcript.to_bytes())`

真正重要的不是 `from_utf8_lossy`，
而是这个条件判断。

因为到调用它的时候，通常已经有两个可能来源：
1. watcher 长期累积出来的 transcript
2. 当前调用现场手上的 fallback 文本

而它明确给了优先级：
- transcript > fallback

所以它不是 formatter，
而是 source arbitration。

---

## 第二层：这里把 transcript 视为 output lifecycle 的主真相源

这和前面几篇是完全一致的：
- `start_streaming_output(...)` 持续喂 transcript
- `process_chunk(...)` 先写 transcript，再决定是否发 delta
- `spawn_exit_watcher(...)` 等 output drained 后再发 end
- `emit_exec_end_for_unified_exec(...)` / `emit_failed_exec_end_for_unified_exec(...)` 最终都调 `resolve_aggregated_output(...)`

这条链连起来就得到一个很清楚的判断：

> **unified-exec 最终输出的主真相源，不是任意时刻抓到的一段文本，而是被 output lifecycle 排干后的 transcript。**

这层设计非常对。

---

## 第三层：fallback 只在 transcript 完全为空时启用，说明作者不想做“合并两个来源”

这里不是：
- transcript 不够长就补 fallback
- transcript 和 fallback 拼起来

而是非常干脆地二选一：
- transcript 有字节 → 全信 transcript
- transcript 没字节 → 才用 fallback

这意味着作者故意避免：
- 两个来源重复拼接
- 输出顺序被破坏
- 终态重复内容

所以这个函数背后的原则是：

> **最终输出只认一个权威来源，不做模糊拼装。**

这是很成熟的选择。

---

## 第四层：`retained_bytes == 0` 而不是 `to_bytes().is_empty()`，说明它判断的是“是否真的保留过输出”

这里用的是：
- `guard.retained_bytes() == 0`

而不是直接把 bytes 拿出来看空不空。

这说明 `HeadTailBuffer` 本身就带“保留了多少有效字节”的语义，
函数尊重的是这个 buffer 的 retained-state，而不是粗糙地再算一遍。

虽然是小细节，但说明作者没有把 transcript 当普通 Vec。

---

## 第五层：`from_utf8_lossy` 说明终态可读性优先于字节级还原

这里最终是：
- `String::from_utf8_lossy(...)`

这说明 end payload 的 aggregated output 是面向：
- 用户 / 客户端 / agent 消费的文本视图

而不是原始字节流归档。

也就是说，这里优先级是：

> **给出尽可能可读的最终文本，而不是保证 byte-perfect 还原。**

这和整条 unified-exec 路线的定位是一致的。

---

## 第六层：它让 success / failure 两个终态封装器共享同一输出裁决规则

成功终态和失败终态都调用它。

这很重要，
因为如果两边各自写一套：
- 一个优先 transcript
- 一个优先 fallback

后面一定会漂。

现在统一到一个函数里，就把“最终输出源选择规则”收成了单点真相。

这就是好抽象。

---

## 一句角色定义

如果按源码共读风格压一句话，我会这么写：

> **`resolve_aggregated_output(...)` 是 unified-exec 终态输出的来源裁决器：只要 transcript 有保留内容，就以它为最终 aggregated output 的权威来源；fallback 只负责兜底空 transcript。**

---

## 为什么这层设计是对的

### 1. 输出来源优先级单点统一
避免 success/failure 漂移。

### 2. transcript 优先符合完整 output lifecycle
比现场局部文本更可信。

### 3. 不混拼 transcript 和 fallback
避免重复和顺序混乱。

### 4. 文本可读性优先
更符合 agent/tool 协议消费场景。

---

## 继续细拆的话，下一篇最自然的点

最自然继续拆的是：

1. `split_valid_utf8_prefix(...)`
2. `refresh_process_state(...)`
3. `HeadTailBuffer`
