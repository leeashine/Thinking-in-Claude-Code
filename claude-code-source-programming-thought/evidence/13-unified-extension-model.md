# CH13 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：统一扩展模型的关键是 `Command` 协议；命令、技能、插件只在来源层有差异，在运行时暴露层尽量共享同一套骨架。
- 读者收益：理解为什么插件更像装配容器，为什么技能可以被看成 prompt 型命令的语义视图。
- 本章排除内容：不把插件 manifest 逐字段讲解，不展开命令执行细节，不与 MCP 章节混写。
- 关键源码清单：[commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts)、[types/command.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/command.ts)、[skills/loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts)、[utils/plugins/pluginLoader.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/pluginLoader.ts)、[utils/plugins/loadPluginCommands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/loadPluginCommands.ts)。

## 源码锚点

- [commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts)：统一汇总与筛选能力条目。
- [types/command.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/command.ts)：提供统一命令协议。
- [loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts)：把技能解析成可治理的能力结构。
- [pluginLoader.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/pluginLoader.ts)：插件装载入口。
- [loadPluginCommands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/loadPluginCommands.ts)：把插件贡献转换为命令条目。

## 主调用链

1. [commands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/commands.ts) 汇总内建命令、技能、插件和工作流来源。
2. [loadSkillsDir.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/skills/loadSkillsDir.ts) 将技能目录转成结构化能力项。
3. [pluginLoader.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/pluginLoader.ts) 加载插件，再由 [loadPluginCommands.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/plugins/loadPluginCommands.ts) 提取命令。
4. [types/command.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/types/command.ts) 定义这些能力在统一协议中的形态。
5. `getCommands()` / `loadAllCommands()` 基于 availability、enabled、来源与模型可见性筛出最终能力面。

## 关键 trade-off

Claude Code 牺牲了部分“各扩展形态高度定制”的纯粹性，换来统一的能力治理入口。这样做让插件、技能与命令不再拥有完全独立的运行时自由，但显著降低了扩展系统碎片化的风险。

## 事实校验

- “`Command` 是统一协议”来自 `types/command.ts` 与 `commands.ts` 的装配方式。
- “插件是容器而不是平行运行时”来自 `pluginLoader.ts` 和 `loadPluginCommands.ts` 的装载链。
- “技能可被看成命令化结构”来自 `loadSkillsDir.ts` 的解析输出与统一筛选流程。
