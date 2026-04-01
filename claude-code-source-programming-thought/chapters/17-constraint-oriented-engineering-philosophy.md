# CH17 面向约束的工程哲学：依赖环、权限、预算、策略

状态：`completed`

## 一句话摘要

Claude Code 源码最值得学习的，不是它提供了多少能力，而是它不断把“不要失控”写成默认结构：切依赖环、做权限管线、编预算控制器、加锁、设缓存边界、放熔断器。

## 问题引入

很多工程团队在谈 Agent 系统时，容易把注意力放在“能力能不能再强一点”。工具要更多，自动化要更激进，模型要更聪明，团队代理要更会分工。但真正让系统能走远的，往往不是能力上限，而是约束设计。没有约束的能力会迅速变成失控：依赖环让系统初始化脆弱，权限模型让副作用失守，预算失控让长会话自行拖死，任务并发让团队代理互相踩踏，缓存随手打破让性能和行为都变得不可预测。

Claude Code 的一个突出特征，就是它不把这些问题当成后期补丁，而是把约束直接写进源码骨架。于是你会看到一些看上去“不够优雅”的实现：专门抽一个 types 文件只为切掉 import cycle，把 system prompt 类型做成 dependency-free branded type，在权限管线里层层分类和拒绝计数，在任务系统里加文件锁和 high-water mark，在 auto compact 里放递归保护和连续失败熔断。它们看起来都不是炫技代码，但组合起来就是一套非常扎实的工程哲学。

## 思想命题

本章的核心命题是：Claude Code 的真正方法论，不是“能力优先”，而是“约束先于能力”。系统之所以能不断扩展，是因为它把风险预先转写为类型边界、状态机、预算控制器、锁和门控，而不是寄希望于模型懂分寸。

第一，编译期约束先于运行期约束。很多稳定性问题在运行时补救代价很高，因此 Claude Code 会先在模块层拆依赖、中心化核心类型、隔离高风险初始化路径。

第二，权限不是一个 if，而是一条多层管线。工具是否可执行，要经过规则、模式、sandbox、allowlist、classifier、hook、headless fallback 和 denial limit 的共同裁决。

第三，预算是控制器，而不是统计数字。无论是 token budget 还是 auto compact，它们都被写成能主动介入主循环的决策逻辑。

第四，并发协作必须被外部状态约束。多 agent 不靠“彼此礼让”，而靠任务图、锁、busy check 和可串行化更新。

第五，缓存与策略门控也是约束。动态 prompt 边界、危险 uncached section、feature gate 和熔断器，都是在告诉系统哪些自由是昂贵的、应被审查的。

## 源码剖面

编译期约束最直接的证据来自 [types/permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/permissions.ts)、[utils/systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts) 和 [utils/queryContext.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/queryContext.ts)。这些文件的价值不在于各自代码量，而在于它们承担了“低依赖中心节点”的角色。尤其像 `types/permissions.ts` 这种专门抽离纯类型定义以打破 import cycle 的做法，非常能说明 Claude Code 团队的思路：依赖环不是等报错了再清，而是在结构上预防。

运行期约束中最典型的是 [utils/permissions/permissions.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/permissions/permissions.ts)。这里的 `hasPermissionsToUseTool()` 一类逻辑并不是做一次 allow/deny 判断，而是把工具调用依次送过规则检查、自动模式快路、allowlist、classifier、hook、headless 场景降级和拒绝次数熔断。这个结构说明权限系统的真实目标不是“偶尔提醒用户”，而是把副作用行动关进可控状态机。

任务系统是并发约束的另一处证据。[utils/tasks.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/tasks.ts) 不只是保存一个待办列表，而是内建 `blocks/blockedBy` 依赖关系、文件锁、high-water mark 和原子 claim/busy check。换言之，Claude Code 认为多 agent 协作最危险的不是“不会分工”，而是“分工后状态失真”，所以它先把任务外部状态做成一个最小事务系统。

预算约束则体现在 [query/tokenBudget.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query/tokenBudget.ts) 与 [services/compact/autoCompact.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/compact/autoCompact.ts)。前者决定是否自动续写、何时停止，后者决定何时压缩、何时阻断、何时熔断。它们共同说明：Claude Code 不把预算看成监控数字，而把它当成能改变主循环行为的控制器。

缓存与策略门控的例子可以在 [constants/prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts) 和 [constants/systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts) 中看到。`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`、默认 memoize 的 section、危险 uncached section 都说明 cache break 不是开发者的随意行为，而是一种必须被显式声明和局部化的代价。

把这些点连起来，你会看到一条非常清楚的主链：编译期先切依赖环并中心化核心类型，运行时通过权限管线约束副作用，通过任务锁约束并发，通过预算控制器约束资源消耗，通过缓存边界与策略门控约束 prompt 与实验漂移。Claude Code 的“能力”正是建立在这张约束网之上。

## 设计取舍

Claude Code 在 CH17 的第一项取舍，是代码简洁性与系统稳态之间的平衡。很多约束机制都会让实现看起来更啰嗦、更保守，但它换来的是系统在真实环境中的承压能力。

第二项取舍，是局部最优与全局可演进之间的平衡。比如专门拆模块切环、对 cache break 做显式声明、在任务系统里做锁和 busy check，这些决定在局部看来都有点“重”，但它们服务的是系统长期可扩展性。

第三项取舍，是自动化自由度与风险兜底之间的平衡。Claude Code 并不追求让模型拥有尽可能少的束缚，而是默认副作用、预算、并发和缓存都需要闸门。

## 工程启示

如果你要从 Claude Code 学方法，而不是学功能，CH17 是最直接的一章。

第一，把高风险自由转写成结构。依赖环用模块边界切，权限用状态机管，预算用控制器管，并发用锁和外部状态管。

第二，把默认路径设计得保守。危险动作、危险缓存破坏和危险自动化都应该显式而昂贵。

第三，不要把约束看成阻碍创新。真正的复杂系统，正是因为约束稳定，能力才有地方叠加。

第四，让每种约束都有可检查的载体。类型、锁、gate、counter、boundary、熔断器，都是让约束从口头原则变成代码事实的方式。

## 思考题

1. 如果把 Claude Code 的权限管线简化成单次 allow/deny，会失去哪些关键风险控制能力？
2. 为什么“先切依赖环，再谈功能扩展”是 Agent 系统里比一般脚本工程更重要的原则？
3. 如果没有任务锁和 busy check，多 agent 协作会出现哪些经典 TOCTOU 问题？

## 本章不覆盖什么

本章不重讲前面章节的具体功能，不逐项分析所有 feature gate，也不讨论产品路线选择。这里关注的是 Claude Code 如何把约束做成架构，而不是把约束留给人肉自觉。
