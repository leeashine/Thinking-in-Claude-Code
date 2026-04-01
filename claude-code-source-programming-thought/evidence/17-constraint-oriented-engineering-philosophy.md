# CH17 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Claude Code 的系统哲学是“约束先于能力”，依赖环、权限、预算、缓存和并发都被写成默认结构。
- 读者收益：理解为什么看似保守、啰嗦的实现组合起来会形成可长期演进的 Agent 系统。
- 本章排除内容：不重复各章节功能细节，不逐一分析所有 gate 与 prompt 文案。
- 关键源码清单：[types/permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/permissions.ts)、[utils/systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts)、[utils/queryContext.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/queryContext.ts)、[utils/permissions/permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/permissions/permissions.ts)、[utils/tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts)、[query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)、[services/compact/autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts)、[constants/prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts)、[constants/systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts)。

## 源码锚点

- [types/permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/permissions.ts)：抽离纯类型以打破 import cycle。
- [systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts)：dependency-free system prompt 类型边界。
- [queryContext.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/queryContext.ts)：高依赖上下文装配被单独拆出。
- [permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/permissions/permissions.ts)：多层权限管线与 denial limit。
- [tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts)：任务依赖、锁与原子 claim。
- [tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)：输出预算控制器。
- [autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts)：压缩 gate、递归保护和熔断。
- [prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts) / [systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts)：缓存边界与危险 uncached section。

## 主调用链

1. 编译期先通过低依赖类型与上下文模块切掉高风险依赖环。
2. 运行时执行工具前，[permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/permissions/permissions.ts) 让调用经过规则、模式、classifier、hook 和 denial limit 管线。
3. 多 agent 协作时，[tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts) 通过锁、依赖与 busy check 保证状态串行化。
4. 主循环继续通过 [tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts) 与 [autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts) 约束资源消耗。
5. prompt 层再用 [prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts) 与 [systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts) 约束缓存破坏与实验漂移。

## 关键 trade-off

Claude Code 接受更重的结构约束，换取系统不失控的长期稳态。它放弃了一部分局部简洁与自由度，但换来了可扩展、可恢复、可治理的运行时。

## 事实校验

- “切依赖环是显式工程选择”来自 `types/permissions.ts` 等低依赖模块的职责。
- “权限是多层管线”来自 `permissions.ts` 的规则、classifier、hook、fallback 与 denial limit。
- “多 agent 协作靠外部状态约束”来自 `tasks.ts` 的锁与 busy check。
- “预算与缓存也被写成约束”来自 `tokenBudget.ts`、`autoCompact.ts` 和 prompt section 机制。
