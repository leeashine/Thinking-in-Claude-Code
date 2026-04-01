# CH12 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Compact 与 Token Budget 不是性能附录，而是主循环内部的上下文治理控制回路。
- 读者收益：理解 Claude Code 为什么把输出续写、微型压缩、自动压缩和 full compact 做成分层机制，而不是一个“摘要按钮”。
- 本章排除内容：不展开底层模型窗口差异，不细讲 compact prompt 文案，不复述 session storage 的完整持久化协议。
- 关键源码清单：[query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)、[query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)、[services/compact/compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)、[services/compact/autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts)、[services/compact/microCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/microCompact.ts)。

## 源码锚点

- [query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)：输出预算控制器。
- [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)：把预算续跑和 compact 治理接入主循环。
- [microCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/microCompact.ts)：优先压缩局部 `tool_result` 和冗余内容。
- [autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts)：自动压缩的 gate、buffer、熔断与递归保护。
- [compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)：full compact 后重建 post-compact 工作集。

## 主调用链

1. [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 组织一轮 query 与工具执行。
2. [query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts) 判断是否需要自动续写、何时停止。
3. 若上下文开始膨胀，系统优先尝试 snip 和 [microCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/microCompact.ts)。
4. 若仍接近窗口极限，[autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts) 决定是否触发自动压缩。
5. [compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts) 完成摘要与 post-compact 工作集重建。
6. 主循环带着重建后的工作集继续后续回合。

## 关键 trade-off

Claude Code 选择更复杂的多层治理链，而不是简单“超长就摘要”。代价是 compact 逻辑更像一个子系统；收益是长会话里能保留工作面、降低不必要 full compact 频率，并避免 compact 自身失控。

## 事实校验

- “输出预算与上下文预算分离”来自 `query/tokenBudget.ts` 与 compact 相关文件职责不同。
- “compact 先局部、后整体”来自 `microCompact.ts` 与 `compact.ts` 的分层位置。
- “auto compact 带 gate 与熔断”来自 `autoCompact.ts` 的 buffer、递归保护与失败控制逻辑。
- “full compact 会重建工作集”来自 `compact.ts` 的 post-compact message builder。
