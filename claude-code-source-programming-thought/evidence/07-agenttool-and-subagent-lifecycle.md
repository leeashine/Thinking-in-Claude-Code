# CH07 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：子代理不是递归调用，而是被定义、被裁剪、被隔离、被回收的受控子运行时。
- 读者收益：看清 `AgentDefinition`、`AgentTool`、`createSubagentContext()` 与 `runAgent()` 如何共同构成子代理生命周期。
- 本章排除内容：不展开 Team Agent 的 mailbox、leader approval、共享 Task List 与团队调度。
- 关键源码清单：`tools/AgentTool/loadAgentsDir.ts`、`tools/AgentTool/AgentTool.tsx`、`tools/AgentTool/runAgent.ts`、`utils/forkedAgent.ts`。

## 源码锚点

- `loadAgentsDir.ts`
  - 原因：定义子代理描述结构。
  - 关注点：`AgentDefinition` 的能力、权限、背景执行、隔离与 MCP/frontmatter 字段。
- `AgentTool.tsx`
  - 原因：筛选可见 agent，并把不同代理运行形态分流。
  - 关注点：`filterAgentsByMcpRequirements()`、`filterDeniedAgents()`、team spawn guard、background 与 isolation 分支、fork path。
- `runAgent.ts`
  - 原因：子代理启动、执行、记录和回收的核心实现。
  - 关注点：权限模式覆盖、`allowedTools` 作用域、system prompt 装配、hooks/skills/MCP 初始化、sidechain transcript、`finally` 清理。
- `forkedAgent.ts`
  - 原因：父子上下文切边。
  - 关注点：`createSubagentContext()` 如何克隆缓存、重建 `agentId`、提升 `queryTracking.depth`、处理 `setAppState` 和 `shouldAvoidPermissionPrompts`。

## 主调用链

1. `loadAgentsDir.ts` 解析 agent frontmatter，形成包含工具、权限、技能、MCP、隔离和背景执行策略的 `AgentDefinition`。
2. 模型在 `AgentTool.prompt()` 阶段只看到通过 `filterAgentsByMcpRequirements()` 和 `filterDeniedAgents()` 的 agent。
3. `AgentTool.call()` 判断本次调用属于普通 subagent、team spawn、fork path、background path 还是 isolation path。
4. `runAgent()` 根据 agent 定义和当前会话状态构造 `agentGetAppState()`、system prompt、工具池、skills、hooks 与 agent MCP。
5. `createSubagentContext()` 创建新的 `ToolUseContext`，显式决定哪些状态共享、哪些克隆、哪些禁用。
6. `runAgent()` 进入 `query()` 主循环，记录 sidechain transcript 与 metadata，并在执行过程中持续转发消息。
7. 结束时 `finally` 统一清理 MCP、hooks、prompt cache tracking、读文件缓存、todos 与后台 shell 任务。

## 关键 trade-off

- 约束：如果把子代理当成父代理的简单复制，就很难控制权限泄漏、缓存稳定性、异步 UI 行为和资源回收。
- 方案：把子代理做成完整运行时对象，显式定义、筛选、隔离、记录并回收。
- 代价：实现复杂度更高，需要额外的 context cloning、transcript、hooks/MCP 生命周期与 cleanup 逻辑。
- 收益：权限边界更清晰，resume/fork/background 更稳，长会话中的状态污染和资源泄漏更可控。

## 事实校验

- 直接来自实现：`AgentDefinition` 字段集合、MCP/权限过滤、team spawn 与 background guard、`allowedTools` 覆盖、`createSubagentContext()` 的克隆/禁用策略、`runAgent()` 的 hooks/skills/MCP 初始化与 `finally` 清理。
- 属于推断：把这套机制概括为“受控子运行时”是对多个实现层的结构性总结。
- 需要避免夸大：
  - 不能说子代理完全不继承父上下文；它会选择性继承部分上下文和缓存状态。
  - 不能说所有子代理都不能交互；同步子代理和某些特殊路径仍可共享交互能力。
  - 不能说 `AgentTool` 只负责普通 subagent；它同时分流 fork、team spawn、background 和 isolation 等多种路径。
