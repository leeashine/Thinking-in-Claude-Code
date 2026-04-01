# CH10 System Prompt 的分层装配

状态：`completed`

## 一句话摘要

Claude Code 并不把 system prompt 当成一整段一次性字符串，而是当成一条可分片、可缓存、可覆写、可在不同运行时复用的装配流水线。

## 问题引入

很多人第一次读 Agent 源码时，都会把 system prompt 当成“那段最神秘的大文本”。这种读法有两个问题。第一，它容易把系统行为误解为 prompt engineering 的偶然结果，忽略真正稳定的结构。第二，它会让你错过一个关键事实：一旦 Agent 进入长会话、子代理、权限模式、实验开关、缓存命中和多入口复用的场景，prompt 就不再适合被看成静态文案，而更像一条运行时拼装链。

Claude Code 在这里提供了一个很有代表性的答案。它没有把“系统指令”写成单一常量，而是拆成静态段、动态段、危险段和不同上下文的注入通道。这样做并不是为了让代码看起来更优雅，而是为了回答三个现实问题：哪些内容可以稳定缓存、哪些内容必须每轮重算、哪些信息应该进 system 通道、哪些应该伪装成 user 通道、以及这套逻辑如何在主 agent 和子 agent 之间共享。

因此，本章真正要解释的不是“Claude Code 的 prompt 写了什么”，而是“Claude Code 为什么把 prompt 做成装配系统”。只有理解了这个层次，你才能明白为什么它在后面的 memory、compact、team agent 和插件扩展中仍然保持统一口径。

## 思想命题

本章的核心命题是：对 Claude Code 来说，system prompt 不是文本资产，而是运行时接口。它承担的不是一次性告诉模型“你是谁”，而是为整个系统提供四种稳定能力。

第一，它是缓存边界。`constants/prompts.ts` 中的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 明确把 prompt 切成“可以长期复用的前缀”和“必须按回合变化的后缀”。这说明缓存命中率不是部署层面的透明优化，而是 prompt 架构本身的一部分。

第二，它是分层注册表。`constants/systemPromptSections.ts` 把 section 的构造方式、缓存语义和危险原因编码进统一接口里。换言之，Claude Code 不是在多个地方随手拼接字符串，而是在管理一组带行为语义的 prompt section。

第三，它是上下文分流器。`context.ts` 里并不是简单地产出“一坨上下文”，而是区分 `systemContext` 与 `userContext`。前者进入 system prompt，后者通过 `prependUserContext()` 变成特殊 user message。这个分流说明 Claude Code 很在意不同通道对模型行为的影响。

第四，它是多运行时复用接口。主查询循环会用它，子 agent 也会用它。`runAgent.ts` 不是重新手写一套 agent prompt，而是沿用同一条装配逻辑，只在 agent-specific instruction、MCP、技能和权限模式上做差异化叠加。

因此，Claude Code 的 prompt 思想并不是“把文案写得更强”，而是“把 prompt 变成可以治理的系统部件”。

## 源码剖面

如果从主调用链看，这套装配结构非常清楚。入口之一在 [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)。主循环在准备模型请求前，会先拼好消息工作集，再通过上下文构造器补入 system 与 user 两类环境信息。这里的关键不是“在哪一行调用了 get prompt”，而是 `query.ts` 把 prompt 视为每轮构造消息工作集的一部分，而不是应用启动时一次性准备好的常量。

[context.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/context.ts) 是这一章的第一个关键文件。它把上下文拆成 `systemContext`、`userContext`、记忆文件、路径相关附加信息等多个来源。这个拆法说明 Claude Code 不默认“所有背景信息都适合放进系统提示词”。例如，某些更像用户声明的内容并不会直接进入 system prompt，而会被转译成 user message。这样做的直接收益是：系统约束与用户现场输入可以分别治理。

[constants/prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts) 则暴露了第二个事实：system prompt 并不是一段平铺文本，而是多段 section 的组合。`getSystemPrompt()` 会装配基础人格、工具使用约束、输出预算说明、模式说明、实验门控信息和其他动态上下文。尤其值得注意的是 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`。它的存在意味着 prompt 不是“要么全缓存、要么全不缓存”，而是显式告诉系统：从这条边界之前的内容尽量保持稳定，从这条边界之后允许随回合漂移。

[constants/systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts) 把这种语义更进一步制度化。这里既有默认可缓存的 `systemPromptSection()`，也有需要显式说明原因的 `DANGEROUS_uncachedSystemPromptSection()`。这个接口设计非常值得注意，因为它把“破坏缓存”从一个容易被忽略的技术细节，升级成了需要被命名和解释的行为。换句话说，在 Claude Code 里，prompt cache break 不是开发者的自由，而是一种要被显式审查的代价。

[utils/systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts) 看起来只是一个小文件，但它代表了另一个工程判断：system prompt 在跨模块流动时需要被当成独立类型，而不是普通字符串。这样做的意义不在类型体操本身，而在于它让“系统提示词”成为一个有边界的协议对象，避免不同层随手传 `string` 导致初始化和依赖层次失控。

同样重要的是 system 和 user 通道的分流。Claude Code 并没有把所有环境信息都塞进 `getSystemPrompt()`。`context.ts` 产出的 `userContext` 会在 [utils/messages.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/messages.ts) 相关流程中被包装成 meta user message。也就是说，Claude Code 明确区分“应该塑造模型长期行为的规则”和“应该作为本轮工作背景暴露给模型的信息”。这正是很多简单 Agent 实现最容易忽略的地方。

最后，子 agent 的复用说明这不是主线程私有设计。在 [tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) 中，子代理仍然会走上下文构造与 prompt 装配链，只是在技能、MCP、agent 说明和权限上做局部覆盖。换言之，Claude Code 没有为“团队代理”再发明一套独立 prompt 哲学，而是让同一个 prompt runtime 同时服务主线程和 sidechain。

综上，CH10 的源码结论可以概括为一条主链：`query.ts` 发起回合装配，`context.ts` 产出分流上下文，`prompts.ts` 与 `systemPromptSections.ts` 负责 section 注册和缓存边界，`systemPromptType.ts` 保证跨模块协议稳定，`runAgent.ts` 在子 agent 场景下复用同一套装配逻辑。

## 设计取舍

这种设计的第一个 trade-off 是可治理性与直观性之间的取舍。把 prompt 拆成 section、边界和不同注入通道之后，源码比“一大段模板字符串”更难一眼读懂；但换来的好处是，团队可以明确知道哪个 section 在变、哪个 section 会打坏缓存、哪个 section 应该被子 agent 继承。

第二个 trade-off 是缓存命中与表达自由之间的取舍。`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 和 section 缓存机制会约束开发者随意插入动态内容，因为每一次不必要的动态拼接都可能导致 cache miss。Claude Code 的选择显然偏向前者：宁可把一些说明改写成“没有预算时也无害”的稳定文本，也尽量不让整个 prompt 每轮漂移。

第三个 trade-off 是 system 与 user 通道的分流成本。把上下文拆成两条路径，会让消息构造逻辑更复杂，也要求作者时刻思考“这段信息应该站在哪个语义位置上”。但收益也很明显：系统规则更稳定，用户现场背景更灵活，二者不会互相污染。

第四个 trade-off 是复用与定制的平衡。主 agent 与子 agent 共享同一套 prompt runtime，意味着系统可以保持统一语言和治理逻辑；但与此同时，每个特殊 agent 的个性化空间就必须通过局部覆盖而不是重写实现。Claude Code 明显更看重前者。

## 工程启示

如果你要设计自己的 Agent 系统，最值得迁移的不是 Claude Code 的文案，而是它对 prompt 的工程化处理方式。

第一，不要把 system prompt 只当作字符串资源。把它视为运行时装配结果，明确区分静态段、动态段和危险段。

第二，给 prompt 建立 section 注册表，而不是散落在各模块里随手拼接。只有注册表存在，你才能管理缓存、实验、角色覆写和追责。

第三，显式区分 system 通道和 user 通道。很多所谓“模型不听话”，其实来自把本该作为工作背景的内容误放进系统约束，或者反过来把系统规则混进用户消息。

第四，让主 agent 与子 agent 共享同一套装配逻辑。真正可维护的多代理系统，不应该为每个角色复制一份大 prompt，而应该建立统一 runtime，再在局部叠加差异。

第五，把 cache break 当成要付出成本的行为。最好像 Claude Code 一样要求显式声明，而不是默许任意模块随时打破缓存。

## 思考题

1. 如果把 Claude Code 的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 去掉，让整段 prompt 每轮都动态生成，系统会在哪些地方首先失去稳定性？
2. 哪些信息更适合进入 `systemContext`，哪些信息更适合作为 `userContext` 被包装成 user message？这种区分对模型行为有什么影响？
3. 如果你要为团队代理新增一种角色，应该优先复用 section 注册表做局部增量，还是复制一份独立 prompt？为什么？

## 本章不覆盖什么

本章不展开具体 prompt 文案逐句解释，不重复 memory 与 compact 的注入细节，也不讨论模型厂商对 system/user 通道的底层实现差异。这里关注的是 Claude Code 如何把 system prompt 做成一个可缓存、可分层、可复用的运行时系统。
