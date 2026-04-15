# Codex guidebook v2 图资产索引

## 目标
给当前 guidebook v2 的解释图建立一份最小索引，方便后续统一收口、移动端优化和批次复查。

## 图资产清单

| 文件 | 对应章节 | 图类型 | 主问题 |
| --- | --- | --- | --- |
| `codex-volume-1-02-interactive-request-flow.svg` | 卷一 02 | 主链图 | interactive 请求从入口到回流怎么跑 |
| `codex-volume-2-06-action-result-loop.svg` | 卷二 06 | 闭环图 | action 结果为什么会回到当前 turn |
| `codex-volume-2-08-runtime-core-stable-loop.svg` | 卷二 08 | 收口图 | runtime core 为什么是一条不断判断继续/收口的工作回合线 |
| `codex-volume-3-01-rollout-vs-sqlite-source-of-truth.svg` | 卷三 01 | 边界图 | rollout 与 SQLite 谁是正文真相源 |
| `codex-volume-3-03-thread-turn-pending-request.svg` | 卷三 03 | 分层图 | 恢复后继续工作的工作线由什么组成 |
| `codex-volume-5-01-exec-vs-unified-exec.svg` | 卷五 01 | 对照图 | exec.rs 与 unified-exec 为什么不是平替 |
| `codex-volume-5-02-handler-runtime-session.svg` | 卷五 02 | 主链图 | handler、runtime、session 启动之间怎么分工 |
| `codex-volume-5-03-approval-policy-sandbox-route.svg` | 卷五 03 | 分层图 | approval、policy、sandbox、route 为什么都在执行前主链 |
| `codex-volume-5-04-transcript-first-output.svg` | 卷五 04 | 主链图 | transcript 为什么是输出语义主轴 |
| `codex-volume-6-01-review-vs-guardian.svg` | 卷六 01 | 对照图 | /review 与 guardian 为什么不是一回事 |
| `codex-volume-6-02-guardian-infrastructure-layers.svg` | 卷六 02 | 分层图 | guardian 为什么更像基础设施 |
| `codex-volume-6-03-realtime-collab-agentcontrol.svg` | 卷六 03 | 分层图 | realtime、collab、AgentControl 分别是哪一层 |
| `codex-volume-6-04-memories-startup-pipeline.svg` | 卷六 04 | 管线图 | memories 为什么更像启动管线 |
| `codex-volume-6-05-higher-order-runtime-map.svg` | 卷六 05 | 收口图 | 为什么说 Codex 正在长成更高层 runtime |

## 当前统一收口原则

1. 标题尽量压短，做到“图名即结论”。
2. 副标题只保留读者必须知道的一句判断。
3. 底部收口句优先保留一句，而不是两三句堆叠。
4. 同类图优先统一成：主链图 / 边界图 / 对照图 / 分层图 / 收口图。
5. 后续移动端优化，优先处理：右侧过挤、底部收口句过长、图内一行字过宽。
