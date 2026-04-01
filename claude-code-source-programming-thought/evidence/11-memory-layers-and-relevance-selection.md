# CH11 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Memory 不是静态前缀，而是“多来源发现 + 多层过滤 + 多事件触发 + 双通道注入”的运行时系统。
- 读者收益：理解 Claude Code 为什么能在长会话、多记忆来源和多 agent 场景下控制前缀膨胀与相关性丢失。
- 本章排除内容：不写成向量数据库教程，不展开每类 memory 文件的生成链，也不把“三层记忆”误写成源码里的硬编码枚举。
- 关键源码清单：[memdir/findRelevantMemories.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/memdir/findRelevantMemories.ts)、[utils/claudemd.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/claudemd.ts)、[query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)、[utils/forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts)、[skills/loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts)。

## 源码锚点

- [findRelevantMemories.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/memdir/findRelevantMemories.ts)：负责相关性召回与候选压缩。
- [claudemd.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/claudemd.ts)：负责发现与整理不同作用域的记忆文件。
- [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)：消费 memory prefetch 结果，并在合适时机转为 attachment messages。
- [forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts)：展示子 agent 继承必要上下文、隔离可变状态的方式。
- [loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts)：提供路径触发与 ignore 语义的旁证。

## 主调用链

1. [claudemd.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/claudemd.ts) 发现 managed/user/project/local/auto/team 等候选记忆来源。
2. 过滤逻辑决定哪些文件常驻注入，哪些文件转入按需召回路径。
3. [findRelevantMemories.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/memdir/findRelevantMemories.ts) 通过头部信息、frontmatter 与 side query 选出少量高相关记忆。
4. [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 消费 pending memory prefetch，将结果转成附件消息并注入当前回合。
5. [forkedAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/forkedAgent.ts) 让子 agent 继承必要 memory 相关上下文，但不共享全部可变状态。

## 关键 trade-off

Claude Code 放弃了“简单粗暴的全量前缀注入”，换来更复杂的发现、过滤和召回链。代价是实现更分层；收益是上下文窗口更可控、记忆相关性更高、团队代理更容易保持认知一致而不互相污染。

## 事实校验

- “记忆来源多于三层”来自 `claudemd.ts` 的多来源发现逻辑。
- “三层记忆是写作抽象”是基于源码实现做出的归纳，不是直接枚举值。
- “存在常驻前缀与按需召回双通道”来自过滤逻辑与 `query.ts` 的附件注入路径。
- “子 agent 继承必要上下文但隔离状态”来自 `forkedAgent.ts`。
