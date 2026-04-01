# CH18 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Claude Code 型系统可迁移的不是单个 prompt，而是六层稳定边界：能力编目、主循环、工具 ABI、子 agent 运行时、共享任务状态、上下文治理。
- 读者收益：得到一套可以直接用于自建 Agent 系统的构建顺序和方法论。
- 本章排除内容：不提供具体脚手架教程，不重复前十七章局部源码细节。
- 关键源码清单：[query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)、[Tool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/Tool.ts)、[services/tools/toolOrchestration.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/tools/toolOrchestration.ts)、[tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)、[utils/forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts)、[utils/tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts)、[commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts)、[services/compact/compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)、[query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)、[utils/sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts)。

## 源码锚点

- [commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts)：能力编目层。
- [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)：回合主循环。
- [Tool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/Tool.ts) / [toolOrchestration.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/tools/toolOrchestration.ts)：工具 ABI 与调度层。
- [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) / [forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts)：子 agent 运行时。
- [tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts)：共享任务状态层。
- [compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)、[tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)、[sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts)：上下文治理层。

## 主调用链

1. 用户输入先进入 [commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts) 所管理的能力编目与入口选择。
2. [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 组织主回合循环。
3. [Tool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/Tool.ts) 与 [toolOrchestration.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/tools/toolOrchestration.ts) 负责能力调用与调度。
4. [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) 在需要拆分工作时拉起子 agent。
5. [tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts) 为多 agent 提供共享任务状态。
6. [compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)、[tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts) 和 [sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts) 维持长期运行与恢复。

## 关键 trade-off

Claude Code 型系统优先建设统一骨架，而不是尽快堆出花哨能力。代价是前期架构投入较重；收益是后续扩展仍能纳入统一治理，系统不容易在多 agent、长会话和外部扩展下碎裂。

## 事实校验

- “六层稳定边界”是基于多章源码的抽象总结，不是源码中的直接枚举。
- “建议构建顺序”来自这些边界在源码里的依赖关系，而不是任意排序。
- “上下文治理属于主系统”来自 compact、token budget 与 session storage 在主循环中的位置。
