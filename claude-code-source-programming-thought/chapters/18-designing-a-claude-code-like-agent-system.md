# CH18 如何设计你自己的 Claude Code 型 Agent 系统

状态：`completed`

## 一句话摘要

Claude Code 型系统真正可迁移的，不是某一段 prompt 或某一个工具，而是一组稳定边界：能力编目、回合主循环、工具 ABI、子 agent 运行时、共享任务状态和上下文治理。

## 问题引入

读完前十七章之后，最自然的问题不是“Claude Code 还有哪些功能”，而是“如果我要自己做一个类似系统，应该从哪里开始”。这是最容易出偏差的一步。很多团队会直接从 supervisor/worker、多 agent 分工、长提示词或外部工具开始，结果很快发现系统能演示、不能稳跑。原因并不复杂：真正支撑 Claude Code 的，不是最表面的能力，而是那些被埋在骨架里的边界。

换句话说，Claude Code 型系统的关键不是“让模型像人一样工作”，而是“把智能拆成一组能被治理的接口和状态”。如果没有统一能力协议，你扩展几个阶段之后就会碎片化；如果没有主循环，所谓多步智能只是一串偶然连续调用；如果没有任务图和锁，多 agent 只会互相打架；如果没有 compact 与恢复机制，长会话迟早会失忆或自我拖死。

因此，CH18 的目标不是再讲一遍源码，而是把前面十七章抽成一套构建顺序和系统设计法。

## 思想命题

本章的核心命题是：Claude Code 型 Agent 系统应该被设计成六层稳定边界，而不是一团会动的 prompt。

第一层是能力编目层。系统先要知道“自己拥有哪些能力、这些能力何时可见、来自哪里、是否允许模型直接调用”。没有这层，扩展越多越混乱。

第二层是回合主循环。Agent 之所以具备多步执行能力，不是因为 prompt 足够长，而是因为存在一个能反复组织消息、调用模型、执行工具、处理恢复和决定下一步的循环。

第三层是工具 ABI。只要系统允许模型调用外部能力，就必须先定义统一的工具协议，把 schema、权限、并发安全和结果回写变成显式接口。

第四层是子 agent 运行时。子 agent 不是“再调一次模型”，而是带独立身份、权限、能力面、转录和清理逻辑的受控运行时。

第五层是协作状态层。多 agent 协作不应该只靠对话，而要有共享任务状态、依赖关系、原子 claim 和阻塞表达。

第六层是上下文治理层。memory、budget、compact、恢复和持久化，决定系统是否能长时间持续工作。

只有这六层都成立，系统才配得上被称为“Claude Code 型”，否则它更像是某种功能集合而非稳定架构。

## 源码剖面

CH18 的证据来自多章交叉，而不是单一模块。能力编目层最直接的入口仍是 [commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts) 与 [types/command.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/command.ts)。前者负责汇总命令、技能、插件与工作流来源，后者负责把这些能力投影到统一协议里。由此可以得出第一个设计原则：不要从“某个能力怎么实现”开始设计，要先从“能力如何被编目和治理”开始设计。

回合主循环层则由 [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 提供。前文已经看到，Claude Code 的智能不是单次推理，而是 `query.ts` 在消息构造、模型调用、工具执行、附件注入、错误恢复和继续条件之间不断循环。由此可以抽象出第二个原则：一个 Agent 系统必须先有稳定的回合循环，再谈智能行为堆叠。

工具 ABI 在 [Tool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/Tool.ts) 与 [services/tools/toolOrchestration.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/tools/toolOrchestration.ts) 中体现得最直接。工具不仅有 `call` 和 `inputSchema`，还有权限检查、并发安全、只读属性和消息渲染方式。Claude Code 的经验是：工具协议越统一，系统越容易约束副作用与调度策略。

子 agent 运行时的核心是 [tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) 与 [utils/forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts)。这里清楚地表明，子 agent 拥有独立 `agentId`、独立工具集、可覆盖权限、可预装技能和独立 transcript，同时又会继承必要上下文。由此可以抽象出第四个原则：子 agent 必须是“受控运行时”，而不是“换 prompt 的再次调用”。

协作状态层来自 [utils/tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts)、[tools/TaskCreateTool/TaskCreateTool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/TaskCreateTool/TaskCreateTool.ts)、[tools/TaskUpdateTool/TaskUpdateTool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/TaskUpdateTool/TaskUpdateTool.ts) 和 [tools/TaskListTool/TaskListTool.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/TaskListTool/TaskListTool.ts)。这些文件表明，多 agent 的协作协议在 Claude Code 里是外部任务状态，而不是聊天内容。由此可以抽象出第五个原则：共享任务图比“让 agent 互相发消息”更适合作为协作基座。

最后，上下文治理层由 [services/compact/compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)、[query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)、[utils/claudemd.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/claudemd.ts) 和 [utils/sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts) 一起支撑。预算、记忆、压缩、恢复和持久化共同决定“系统能不能在第十轮之后还保持工作能力”。因此，第六个原则是：上下文治理不是附录，而是主系统的一部分。

把这些层串起来，就是 Claude Code 型系统的建议主链：`用户输入/命令 -> 能力编目层筛选入口 -> query 主循环 -> Tool ABI 与调度 -> 必要时创建子 agent -> 共享任务状态协调 -> compact/memory/session storage 维持长期运行 -> 下一轮继续`。如果你先做这个主链，再往里填具体工具和角色，系统更有机会长成稳定架构。

## 设计取舍

Claude Code 型系统在设计上的第一项取舍，是先搭骨架还是先堆能力。它显然选择前者，这会让前期看起来进展更慢，但能显著降低后续返工成本。

第二项取舍，是集中治理与局部自由之间的平衡。统一能力协议、统一主循环、统一工具 ABI、统一任务图都会限制各模块自由发挥，但它换来的是系统级可控性。

第三项取舍，是短期可演示性与长期可演进性之间的平衡。一个没有 compact、恢复和任务状态层的系统，往往更快做出 demo；但 Claude Code 的做法说明，真正长期可用的系统必须尽早把这些底座补齐。

## 工程启示

如果你要从零开始做一个 Claude Code 型 Agent 系统，最稳妥的顺序不是按“功能吸引力”排，而是按架构依赖排。

第一步，先做能力编目层和统一命令/技能/插件协议。

第二步，实现稳定的回合主循环，让系统能组织消息、调用模型、执行工具和处理继续条件。

第三步，定义 Tool ABI，并把权限与并发属性纳入协议。

第四步，再引入子 agent 运行时，但务必给它独立身份、权限和清理逻辑。

第五步，引入共享任务状态，而不是只靠聊天内容协调团队。

第六步，补齐 memory、budget、compact、恢复和持久化，让系统真正能长时间运行。

如果顺序反过来，例如一开始就做多 agent 编排，却没有任务状态、预算治理和统一能力协议，最终只会得到一套不断失控的 orchestration 外壳。

## 思考题

1. 为什么 Claude Code 型系统最不应该跳过的底层，是主循环和工具 ABI，而不是多 agent 编排？
2. 共享任务状态相比纯对话协作，为什么更适合作为多 agent 系统的基础？
3. 如果只复制 prompt 与工具，而不复制 compact、恢复和 session storage，最先崩溃的会是哪一层？

## 本章不覆盖什么

本章不提供某个具体产品的逐步脚手架教程，也不替代前十七章的源码细节。这里关注的是如何把 Claude Code 的实现抽象成一套可迁移的系统设计方法。
