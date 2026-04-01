# CH12 Compact、Token Budget 与上下文折叠

状态：`completed`

## 一句话摘要

在 Claude Code 里，Compact 和 Token Budget 不是性能优化附录，而是保证长会话能持续运行的上下文治理协议。

## 问题引入

长对话型 Agent 最常见的失控方式，不是模型答错一题，而是系统在若干轮之后逐渐失去工作能力。窗口被旧消息挤满，工具结果把上下文拖得越来越长，模型输出被过早截断，压缩动作又破坏当前工作面。很多实现把这些问题视为“之后再优化”的性能事项，但 Claude Code 的源码明确告诉我们：只要 Agent 要处理多回合、多工具、多文件和多代理协作，预算治理就必须成为主运行时的一部分。

Claude Code 在这里做了一个非常关键的区分：输出预算和上下文预算不是同一件事。前者关心“这轮回答还能不能继续写下去”，后者关心“系统是否还能带着有效工作集继续思考”。因此，Token Budget、Micro Compact、Auto Compact、Full Compact、Context Collapse 和各种恢复机制并不是互相替代的功能，而是一组依次接管不同风险的治理层。

从这个角度看，CH12 讨论的不是“什么时候摘要”，而是“系统如何把有限窗口当成一项被持续调度的资源”。

## 思想命题

本章的核心命题是：Claude Code 把上下文窗口当成一种需要实时治理的有限资源，因此 compact 不是按钮，budget 不是指标，它们共同组成主循环里的控制回路。

第一，输出预算与上下文预算分离。`query/tokenBudget.ts` 处理的是每轮回答是否需要自动续写、何时进入收益递减和何时该停止；而 compact 相关代码处理的是对话工作集是否已经挤压到影响下一轮推理。

第二，治理是分层的，不是一步到位的。系统不会一上来就 full compact，而会先尝试更便宜的手段，如工具结果裁剪、snip、micro compact；只有当这些仍不足以维持有效窗口时，才进入 auto/full compact。

第三，compact 是重建协议，而不是摘要协议。`compact.ts` 并不满足于生成一段总结，而是要重建 post-compact 的消息工作集，保留必要边界、尾段、文件状态和计划信息，让系统能够继续工作。

第四，compact 自身也受策略治理。`autoCompact.ts` 里有 buffer、warning/error/blocking、多重 gate、递归保护和连续失败熔断。这说明 Claude Code 很清楚：压缩动作本身也可能制造风险，因此不能让它无限触发。

## 源码剖面

最先要看的文件是 [query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts)。这里的逻辑不是简单地“超过某个百分比就停”，而是围绕 continuation 次数、目标 token、最小剩余空间和收益递减来做判断。系统真正关心的是：如果用户显式要求长答案，这轮是否应该继续；如果继续下去只会付出巨大成本却带来很少新增内容，应该在哪个点收手。这个控制器的存在说明，Claude Code 把输出长度视为可调度的行为，而不是模型自然表现。

这套控制器直接接入 [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)。当一轮输出触发 budget 条件时，主循环并不会简单结束，而会注入一条 meta user message 让模型继续，直到达到用户要求或进入收益递减。这里非常像一个控制回路：`query.ts` 不是被动地等待模型生成结果，而是在回合之间主动修正模型的停止条件。

上下文治理的第二层在 [services/compact/microCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/microCompact.ts)。这个文件代表 Claude Code 的一个重要判断：很多时候并不需要 full compact，只需要先压缩历史 `tool_result` 或局部冗余内容，就能恢复足够的工作空间。这种先小修、后大修的顺序，避免了系统过于频繁地执行重型压缩。

真正的 full compact 入口则在 [services/compact/compact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/compact.ts)。这一层最值得注意的不是摘要模型怎么调用，而是 `buildPostCompactMessages` 一类函数如何重建消息工作集。Claude Code 在 compact 之后仍然要维持会话边界、保留尾段、恢复文件工作面、计划状态和必要上下文。换言之，compact 的目标不是“让上下文变短”，而是“让系统在更短上下文里继续保持同一项工作”。

[services/compact/autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts) 则展示了这套机制如何进入自动控制模式。这里不仅会计算窗口利用率，还会为摘要输出预留 buffer，区分 warning、error 与 blocking 区间，并协调 reactive compact、session memory compact、context collapse 等其他策略。尤其重要的是连续失败熔断和递归保护：Claude Code 明确知道 compact 可能失败，也知道系统在极端情况下会陷入“压缩自己失败，再次尝试，再次失败”的死循环，因此在控制器里直接编码了停止条件。

这套结构还和 [screens/REPL.tsx](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/screens/REPL.tsx) 及状态持久化协同。用户在前台看到的不是“窗口满了所以结束”，而是系统在预算、水位和恢复动作之间进行调度后的结果。结合 `sessionMemoryCompact` 与 `sessionStorage` 相关逻辑，可以看出 compact 并非一次性摘要，而是长期会话保存协议的一部分。

因此，本章可以还原出一条非常清晰的治理顺序：主循环先进行普通消息与工具结果处理，再由 token budget 决定是否需要续跑；若上下文膨胀，则优先尝试 snip 与 micro compact；仍然不够时，`autoCompact.ts` 决定是否进入 full compact；`compact.ts` 完成摘要与 post-compact 工作集重建；最后系统再带着新工作集继续进入下一轮 query。

## 设计取舍

Claude Code 在 CH12 的第一项取舍，是响应连续性与治理开销之间的平衡。持续自动续写能更好满足用户对长回答的期待，但也会增加请求次数和预算消耗；系统因此引入收益递减阈值，而不是无条件续跑。

第二项取舍，是轻量局部修复与重型整体重建之间的平衡。micro compact 成本低、破坏小，但并不总能解决问题；full compact 更彻底，却更昂贵、也更容易影响当前工作面。Claude Code 的策略显然是先局部，再整体。

第三项取舍，是自动恢复与可预测性之间的平衡。让 auto compact 自己判断何时触发，能显著提升长会话鲁棒性；但如果没有 gate、buffer、递归保护和熔断，这种自动化很快会变成新的不确定性源头。Claude Code 因此在自动化外面又包了一层治理。

第四项取舍，是摘要简洁性与工作面保留之间的平衡。很多系统的 compact 只是把历史压成一段自然语言总结；Claude Code 则选择更复杂的 post-compact 重建，因为它更在意系统能否继续工作，而不是摘要本身是否优雅。

## 工程启示

你如果要做长会话 Agent，可以直接借鉴 Claude Code 的四个原则。

第一，把输出预算和上下文预算分开治理。一个控制回答是否续写，一个控制会话是否还装得下工作面。

第二，治理动作要分层。先做 snip 和 micro compact，再决定是否 full compact，不要每次都用最重手段。

第三，把 compact 设计成“恢复工作面”的协议，而不是“生成摘要”的工具。摘要不是目的，续跑才是目的。

第四，让 compact 自己也服从策略门控。递归保护、buffer 和熔断器不是锦上添花，而是自动治理能否可靠的前提。

## 思考题

1. 如果只保留 full compact，去掉 micro compact 和工具结果裁剪，Claude Code 的长会话体验会在哪些方面恶化？
2. 为什么输出 token budget 和上下文窗口预算必须分开处理？把它们混成一个阈值会带来什么误判？
3. 如果 compact 后只留下摘要而不重建工作面，哪些后续能力最容易退化？

## 本章不覆盖什么

本章不逐条解释 compact prompt 文案，不讨论底层模型窗口大小的厂商差异，也不展开 session storage 的全部恢复细节。这里关注的是 Claude Code 如何把 budget 与 compact 编进主运行时，形成可续跑的上下文治理协议。
