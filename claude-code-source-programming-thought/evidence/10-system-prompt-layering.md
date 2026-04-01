# CH10 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：System Prompt 不是一段静态文案，而是一条可分片、可缓存、可覆写、可在主 agent 与子 agent 之间复用的装配流水线。
- 读者收益：理解 Claude Code 为什么要把 prompt 结构化为 section 与 boundary，而不是维护一份越来越长的大字符串。
- 本章排除内容：不逐句分析提示词文案，不展开具体模型厂商对 system/user 通道的内部实现，也不重复 memory 与 compact 的后续章节。
- 关键源码清单：[context.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/context.ts)、[constants/prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts)、[constants/systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts)、[utils/systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts)、[query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)、[tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)。

## 源码锚点

- [context.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/context.ts)：区分 `systemContext` 与 `userContext`，说明上下文并不走单一注入通道。
- [prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts)：`getSystemPrompt()` 负责 section 级装配；`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 明确缓存边界。
- [systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts)：`systemPromptSection()` 与 `DANGEROUS_uncachedSystemPromptSection()` 把缓存语义显式编码成接口。
- [systemPromptType.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/systemPromptType.ts)：把 system prompt 作为独立协议类型，而不是任意字符串。
- [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)：在主查询回合中消费 prompt 装配结果。
- [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)：子 agent 沿用同一套 prompt 装配思路。

## 主调用链

1. [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 进入一轮 query 时准备消息工作集。
2. [context.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/context.ts) 生成 `systemContext`、`userContext` 和相关环境信息。
3. [prompts.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/prompts.ts) 通过 `getSystemPrompt()` 将静态 section 与动态 section 装配为 system prompt。
4. [systemPromptSections.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/constants/systemPromptSections.ts) 决定某 section 是可缓存还是显式 uncached。
5. `userContext` 不直接进入 system prompt，而是经消息层转换为 meta user message。
6. [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) 在子 agent 场景复用同一条装配路径，只叠加角色差异。

## 关键 trade-off

Claude Code 牺牲了 prompt 定义的直观性，换来缓存命中、角色复用和注入路径治理。系统不再容易“一眼看完整段 prompt”，但获得了 section 级调整、显式 cache break 和主/子代理统一装配的工程优势。

## 事实校验

- “system prompt 不是单段字符串”来自 `prompts.ts` 和 `systemPromptSections.ts` 的 section 装配接口。
- “存在静态/动态边界”来自 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 的显式常量。
- “system 与 user 通道分流”来自 `context.ts` 的不同上下文产物与后续消息注入路径。
- “子 agent 复用同一装配逻辑”来自 `runAgent.ts` 的上下文与 prompt 复用方式。
