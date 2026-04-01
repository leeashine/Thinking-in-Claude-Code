# CH11 Memory 体系：用户、项目、团队三层记忆

状态：`completed`

## 一句话摘要

Claude Code 的 memory 不是把一堆 `CLAUDE.md` 粗暴拼到前缀里，而是一个包含发现、筛选、相关性召回、附件注入和跨 agent 继承的运行时系统。

## 问题引入

记忆系统几乎是所有 Agent 产品最容易被简化误读的部分。很多实现会把 memory 描述成“把用户偏好、项目说明和团队规范存成一些 Markdown 文件，然后在每次对话开始时统统塞给模型”。这种做法在 demo 阶段似乎足够，但一旦进入真实工程环境，很快就会遇到三个问题：第一，前缀会膨胀得过快；第二，真正相关的内容往往被海量弱相关说明淹没；第三，子 agent、技能和路径触发信息会不断打破“单次全量注入”的假设。

Claude Code 对这个问题的回答非常成熟。它没有把 memory 当成静态资源，而是把它看成一条运行时召回链。系统先发现有哪些记忆文件，再决定哪些常驻、哪些按需召回、哪些受路径和模式影响，最后再通过不同通道注入模型。这样一来，memory 不只是“让模型记住一些东西”，而是在长会话和多代理环境里维持行为稳定的上下文控制层。

因此，本章虽然沿用“用户、项目、团队三层记忆”的表述，但要特别强调：这是一种帮助读者理解的抽象，不是源码里直接写死的三分类枚举。真实实现比这个抽象更复杂，也更工程化。

## 思想命题

本章的核心命题是：Claude Code 的 memory 体系，本质上是“多来源发现 + 多层过滤 + 多事件触发 + 双通道注入”的运行时协议，而不是前缀拼接技巧。

第一，它是发现系统。`utils/claudemd.ts` 与 memdir 相关代码并不假设只有一个固定记忆文件，而是会从托管记忆、用户记忆、项目记忆、本地记忆、自动记忆和团队记忆等来源中发现候选集。

第二，它是过滤系统。发现到候选集之后，Claude Code 不会默认全部常驻。`filterInjectedMemoryFiles()` 会根据配置和模式把部分文件从长期前缀里拿掉，转入后续的相关性召回路径。

第三，它是相关性系统。`findRelevantMemories.ts` 不读取整份大文件来做昂贵筛选，而是偏向读取头部信息、frontmatter 和摘要线索，通过 side query 从候选集中找出最值得注入的一小部分。

第四，它是双通道注入系统。一部分 memory 以稳定上下文前缀的形式出现，另一部分则在 `query.ts` 的运行过程中被转成附件消息按需注入。换言之，Claude Code 明确区分“始终该带着的记忆”和“只有这轮任务相关时才该出现的记忆”。

第五，它是可继承但不完全共享的系统。子 agent 会继承必要的用户与系统上下文，但并不等于主线程所有可变状态都直接复制过去。Claude Code 在这里追求的是“共享必要认知，隔离易漂移状态”。

## 源码剖面

CH11 的主入口在 [memdir/findRelevantMemories.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/memdir/findRelevantMemories.ts) 和 [utils/claudemd.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/claudemd.ts)。前者负责相关性召回，后者负责记忆文件的发现与整理。这个拆法已经说明 Claude Code 不把 memory 看成单次加载动作，而是至少分为“找到候选”和“从候选中选择”两个阶段。

先看发现阶段。`getMemoryFiles()` 会从多个来源汇总候选，包括 managed memory、用户级记忆、项目/本地 CLAUDE 文件、自动生成的 memory，以及团队级协作记忆。这里的重点不是来源名字本身，而是系统愿意承认“记忆存在不同作用域”。用户偏好属于跨项目长期记忆，项目说明属于仓库局部记忆，团队产物则更接近多 agent 协作时的共享痕迹。所谓“三层记忆”，正是从这种多作用域实现中抽象出来的叙事结构。

然后进入过滤阶段。并不是所有发现到的文件都会变成 system prompt 前缀。`filterInjectedMemoryFiles()` 的作用之一，就是让某些 memory 不再走“默认常驻”的路径，而改为后续按需检索。这个决定非常关键，因为它直接回应了长前缀膨胀的问题。Claude Code 的判断是：稳定重要的记忆应当常驻，但那些只有在特定任务、路径或上下文下才相关的记忆，不值得占据每一轮的固定窗口。

相关性召回是第三层。[findRelevantMemories.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/memdir/findRelevantMemories.ts) 不是暴力读取所有文件全文，而是优先根据头部信息、frontmatter 和摘要线索建立轻量候选，再借助 side query 选出最多几条真正相关的记忆。这个实现细节说明 Claude Code 在 memory 上同样遵循“先便宜过滤，再昂贵裁决”的原则。也就是说，相关性不是靠无穷算力解决，而是靠分层筛选解决。

真正把这些记忆接入主循环的是 [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)。在一轮 query 开始前或运行过程中，系统会预取 memory 相关结果；当这些结果成熟后，再将其转成 attachment messages 注入消息工作集。这一点很重要，因为它意味着 memory 并不一定提前硬编码进 prompt，而可以在运行中作为带出处的附加上下文被模型消费。这种形式比“塞进一段长前缀”更有弹性，也更容易在 compact 时保留边界。

多 agent 语义则体现在 [utils/forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts) 一类文件中。子 agent 会继承必要的 `userContext` 和 `systemContext`，但并不会把主线程所有可变工作状态完全共享过去。Claude Code 在这里的想法很清楚：memory 要为整个团队维持认知一致性，但 agent 的局部行动状态不应该彼此污染。

还有一个容易被忽略但很有启发的旁证来自 [skills/loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts)。技能系统与 memory 系统都共享类似的路径触发和 ignore 语义。这说明 Claude Code 在更高层上正在构造一套统一的“路径关联知识触发协议”：无论是技能还是记忆，只要其价值受当前工作目录、文件模式或任务上下文影响，都不应该一刀切全量注入，而应按触发条件暴露。

因此，CH11 可以还原出这样一条主链：`claudemd.ts` 发现候选记忆，过滤逻辑决定常驻与按需注入的分工，`findRelevantMemories.ts` 做轻量相关性召回，`query.ts` 消费 prefetch 结果并以附件消息形式注入，`forkedAgent.ts` 保证子 agent 继承必要认知但隔离可变状态。

## 设计取舍

Claude Code 的第一项取舍，是前缀稳定性与召回实时性之间的平衡。把所有 memory 常驻前缀最简单，但会迅速吞噬上下文窗口；完全按需检索又容易让系统在某些回合忘掉关键长期规范。Claude Code 的选择是双轨制：少量稳定重要记忆常驻，其余走召回与附件路径。

第二项取舍，是检索精度与成本之间的平衡。`findRelevantMemories.ts` 并不追求最重的全文语义搜索，而是先做便宜的候选筛选，再做小规模 side query 判断。这样做可能错过极少数深埋在长文档中的相关段落，但能显著降低每轮的成本与复杂度。

第三项取舍，是团队共享与 agent 隔离之间的平衡。Claude Code 让子 agent 继承必要的上下文，但保留局部执行状态隔离。这会增加上下文装配复杂度，却能避免多个 agent 在一个会话里互相污染工作面。

第四项取舍，是抽象简洁与真实实现之间的平衡。对读者来说，“用户、项目、团队三层记忆”更好理解；对源码来说，真实结构显然更细碎、更运行时化。Claude Code 接受这种复杂性，因为它面对的是长期运行系统，而不是教学示例。

## 工程启示

如果你想复刻类似的记忆系统，可以直接借走 Claude Code 的五个方法。

第一，把 memory 做成发现协议，而不是约定死一个文件。不同作用域的知识应该被系统承认。

第二，不要把所有记忆都放进常驻前缀。把“长期稳定约束”和“按任务相关的局部知识”分成两条注入路径。

第三，让相关性召回分层进行。先做便宜的元数据筛选，再对少量候选做更昂贵判断。

第四，让 memory 成为带出处的附件，而不是匿名大段文案。这样做更容易追踪来源，也更方便 compact 和恢复。

第五，在多 agent 场景下共享认知，不共享所有状态。长期知识可以继承，但局部执行面必须隔离。

## 思考题

1. 如果把所有 memory 永久常驻在 system prompt 前缀里，Claude Code 的 compact 与 token budget 会首先在哪些地方承压？
2. “三层记忆”作为写作抽象，和源码中的多来源发现模型有什么差异？这种抽象在什么地方有用，在什么地方会误导？
3. 如果你要让团队代理共享更多知识，应该扩展常驻前缀，还是增强按需召回？为什么？

## 本章不覆盖什么

本章不展开记忆文件的具体文档格式，不详细分析每一类 memory 文件的生成来源，也不讨论向量数据库式长期记忆实现。这里关注的是 Claude Code 如何在当前源码里把 memory 做成运行时发现、过滤与注入系统。
