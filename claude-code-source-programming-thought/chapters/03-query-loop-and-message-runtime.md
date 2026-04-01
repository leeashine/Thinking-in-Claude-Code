# CH03 一切皆回合：`query` 主循环与消息驱动运行时

状态：`completed`

## 一句话摘要

Claude Code 的核心不是某个单独工具，而是一个围绕消息、工具结果、上下文预算和恢复路径持续推进的回合执行主循环。

## 问题引入

很多人理解 Agent 系统时，会天然把注意力集中在 Tool 上：能不能读文件、能不能执行命令、能不能联网、能不能起子代理。Claude Code 的源码提醒我们，这种理解顺序刚好反了。真正先被设计出来的，不是“有什么工具”，而是“工具如何在一轮交互里被组织、被确认、被回填、被中断、被恢复”。这个问题的答案，就写在 `query.ts` 里。

`query()` 和 `queryLoop()` 不是简单包一层 API 调用。它们维护一段可跨多次迭代延续的状态：消息数组、工具上下文、自动 compact 跟踪、恢复次数、上一轮继续原因、待输出的 tool summary 等。只要这段状态还没有落到终止条件上，Claude Code 就会继续在同一回合中前进。因此，理解 Claude Code 的第一把钥匙不是 Tool，而是“回合”。

## 思想命题

### 命题一：回合是主循环持有的状态机，不是一次性 API 请求

`queryLoop()` 一开始就构造了 `State`：`messages`、`toolUseContext`、`autoCompactTracking`、`maxOutputTokensRecoveryCount`、`pendingToolUseSummary`、`transition` 等字段全部在回合内部持续演化。只要触发了继续条件，比如 tool follow-up、token budget continuation、reactive compact retry 或 stop hook blocking，Claude Code 就不会把这当成下一次全新对话，而是同一回合的下一次推进。

### 命题二：消息是统一介质，工具只是消息中的一种事件

在 `query.ts` 里，assistant message、tool use、tool result、system boundary、interruption message、tool use summary 都通过同一条消息流推进。工具调用没有脱离主循环存在，而是被包在消息协议里。这一点使得 transcript、resume、compact、SDK 输出和 UI 渲染都能复用同一套数据形状。

### 命题三：上下文治理与错误恢复是主路径，不是边角逻辑

`queryLoop()` 在真正调用模型前，先后插入 tool result budget、snip、microcompact、context collapse、autocompact、blocking limit 检查。模型返回后，还会根据 prompt-too-long、media size、max_output_tokens、stop hooks 和 token budget 决定是否继续。换句话说，Claude Code 假设“长回合一定会遇到上下文与恢复问题”，于是把这些逻辑放在主路径上。

## 源码剖面

`queryLoop()` 的第一层核心是状态持有。它把所有跨迭代变量收进 `state`，每轮开始时解构，再在 continue site 重新组装。这个写法看起来啰嗦，但价值很高：当主循环因为 compact、fallback、budget continuation 或 tool follow-up 而再次进入下一轮时，所有必须保留的信息都有固定归宿。

第二层核心是模型调用前的上下文治理。`query.ts` 先通过 `applyToolResultBudget()` 限制工具结果体积，再按顺序应用 history snip、microcompact、context collapse、autocompact。这里的顺序本身就是设计结论：先做低损失压缩，再做更重的 summary 化压缩，从而尽量保留粒度更细的上下文。`query/config.ts` 又把 streaming tool execution、tool use summaries、fast mode 等运行时开关做成一次快照，避免在一轮回合中途漂移。

第三层核心是工具调度。`services/tools/toolOrchestration.ts` 会把工具调用按“是否并发安全”分批，读型工具可以并发，写型工具严格串行。到了 `services/tools/toolExecution.ts`，工具调用才真正经历权限判断、hooks、进度消息、错误分类、结果回写。这意味着 Claude Code 不是“模型想调工具就调工具”，而是“主循环决定何时给模型调工具，工具层再决定如何安全执行”。

第四层核心是恢复路径。`query.ts` 中有一整套 prompt-too-long、media error、max_output_tokens、fallback model、stop hooks、token budget continuation 的恢复分支。尤其值得注意的是 `transition` 字段：它让测试和运行时都能知道“为什么继续”，而不必从消息文本反推语义。这是典型的工程化做法。

## 设计取舍

Claude Code 在主循环上选择的是“复杂但可恢复”，而不是“简单但脆弱”。代价是 `query.ts` 非常长，理解门槛很高，很多逻辑必须结合 `services/compact/*`、`services/tools/*` 和 `query/*` 子模块一起看。收益则是明显的：一轮交互不必被简化成“请求一次模型，失败了就报错”，而是可以在预算、压缩、模型切换、工具结果和中断之间持续寻找下一条可行路径。

如果没有这种复杂度，Claude Code 很难支持长时间、多工具、多次恢复的真实工作回合。对于一个要在终端里持续执行复杂任务的 Agent，这种复杂度不是奢侈品，而是生存条件。

## 工程启示

第一，Agent 系统应该优先设计回合状态机，而不是先堆工具。因为只要工具一多，回合一定会遇到恢复、预算和上下文问题。

第二，统一消息形状会显著降低系统复杂度。只要 transcript、UI、SDK、compact、resume 共用同一条消息流，很多横切问题就能用同一组基础设施处理。

第三，工具调度最好分成“主循环决定何时调用”与“工具执行层决定如何调用”两层。这样权限、并发、hook 和结果映射就不会和模型控制流混在一起。

第四，恢复原因应该被显式建模。`transition` 这样的状态字段，比在日志里找关键字可靠得多，也更适合测试。

## 思考题

1. 如果去掉 `queryLoop()` 里的 compact / recovery 分支，只保留一次模型调用，这个系统会在什么情况下先失效？
2. 为什么 `toolOrchestration.ts` 要把只读工具并发、写工具串行？
3. 你的 Agent 系统里，继续一轮回合的原因是显式状态，还是隐含在日志和消息中？
4. 如果消息不是统一介质，而是 UI、SDK、resume 各有一套格式，会给维护带来什么后果？

## 本章不覆盖什么

本章聚焦主循环和消息驱动运行时，不展开 REPL 前台、权限系统、Team Agent、memory 策略和扩展机制的完整实现。后续章节会分别拆开这些主题。
