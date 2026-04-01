# CH14 MCP：远程能力总线，而不只是工具接线

状态：`completed`

## 一句话摘要

在 Claude Code 的实现里，MCP 不只是把远程工具接进来，而是把工具、资源、prompts、server instructions 和 agent 运行时一起接到同一条远程能力总线上。

## 问题引入

当很多团队第一次听到 MCP 时，最容易产生的理解是“它是一个工具接线协议”。这种理解不能说错，但远远不够。若 MCP 只是远程 tool adapter，那么它在系统里最多只承担扩展桥接器的角色；而 Claude Code 的源码展示的是另一种更强的用法：MCP 被当成一个能力总线，远端服务器不仅可以贡献工具，还能贡献 prompt、resource、instructions 以及与 agent 生命周期相关的上下文。

这种差异非常重要。把 MCP 看成工具接线，会让系统设计停留在“多接几个外部能力”的层面；把 MCP 看成能力总线，则意味着 Claude Code 正在把本地运行时与远程能力生态接成同一张图。此时，连接管理、配置解析、能力暴露、运行时选择和 agent 继承都会变成第一等问题。

因此，本章要讨论的不是协议细节，而是 Claude Code 如何在源码里把 MCP 从“工具入口”提升为“远程能力总线”。

## 思想命题

本章的核心命题是：Claude Code 使用 MCP 的方式，证明远程能力整合不应该只围绕 tool call 设计，而应该围绕“能力面”设计。

第一，MCP 在这里至少有三层：连接层、暴露层和运行时集成层。连接层负责建立与维护服务器连接，暴露层负责统一呈现远端能力，运行时层决定这些能力如何被主 agent 或子 agent 使用。

第二，MCP 能力不止工具。Claude Code 的相关实现同时处理 prompts、resources 和 server instructions，说明系统把“远程上下文供给”与“远程动作执行”放在同一总线里思考。

第三，MCP 与 agent 生命周期耦合。某些 agent 可以拥有自己的 MCP server 配置或选择性暴露的远程能力，这意味着 MCP 已经进入多代理运行时，而不是停留在应用启动阶段。

第四，MCP 的治理中心不在某个单一 connector 文件，而在连接管理、配置解析与工具执行协同之间。当前工作树里没有看到某个名为 `McpConnector.ts` 的核心枢纽，真正的职责分散在 client、connection manager 和 hook 逻辑里。

## 源码剖面

CH14 的第一组关键文件是 [services/mcp/client.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/client.ts) 与 [services/mcp/MCPConnectionManager.tsx](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/MCPConnectionManager.tsx)。前者更靠近实际客户端交互与能力调用，后者则承担连接状态管理、UI/状态协调与连接生命周期管理职责。这个分层本身已经说明 Claude Code 不把 MCP 当作简单函数调用，而是承认“远程能力可用性”本身就是一类运行时状态。

[services/mcp/config.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/config.ts) 则展示了第二个关键点：MCP 服务器配置不是临时字符串，而是系统可解析、可校验、可被多运行时共享的结构。只要远程能力来自不同 transport、不同 server 和不同作用域，配置层就必须先稳定下来，否则上层运行时只会收到一堆不可治理的连接碎片。

连接之后，Claude Code 并不只把“tool 列表”暴露给系统。根据 [services/mcp/types.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/types.ts) 与相关工具执行逻辑可以看出，MCP 侧至少还会产出 prompts、resources 和 instructions 之类能力。也就是说，本地系统不是只问“你能执行什么动作”，还问“你能提供什么上下文与引导”。这正是“能力总线”与“工具接线”最本质的区别。

运行时集成则出现在主工具编排与子 agent 启动链中。MCP 提供的远程能力最终要被模型看见、被工具层消费，甚至被某些 agent 选择性继承或覆盖。Laplace 的追踪结果指向 [tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)、MCP prompts 与 tool execution 的接合点，说明 Claude Code 并没有把 MCP 限定在应用级，而是允许它深入 agent 级运行时。

这一章还需要一个重要的事实边界：当前工作树里没有找到 `services/mcp/McpConnector.ts`。因此，不能把某个不存在的单文件写成 MCP 核心。根据现有实现，更合理的说法是：MCP 的职责分布在 `client.ts`、`MCPConnectionManager.tsx`、配置解析与使用 hook 上，由这些组件共同构成远程能力总线。

因此，本章可以还原出一条主链：`config.ts` 解析远程 server 配置，`client.ts` 建立与消费连接，`MCPConnectionManager.tsx` 维护连接生命周期与状态，远端能力经类型层统一呈现为工具/资源/prompt/instructions，再由主运行时和 `runAgent.ts` 等路径把这些能力暴露给主 agent 或子 agent。

## 设计取舍

Claude Code 在这里的第一项取舍，是能力广度与系统复杂度之间的平衡。若 MCP 只接工具，实现会简单很多；但系统也会失去远程 prompt、resource 和 instruction 的统一整合能力。Claude Code 显然选择承担复杂度，换更完整的远程能力面。

第二项取舍，是集中枢纽与分层职责之间的平衡。把所有 MCP 逻辑都塞进单文件看似直观，却不利于连接状态、配置、UI 管理和运行时集成的独立演进。Claude Code 当前的做法更分散，但职责更明确。

第三项取舍，是全局共享与 agent 局部定制之间的平衡。让所有 agent 看到完全相同的 MCP 能力最省事；但 Claude Code 更看重按 agent 角色暴露不同远程能力的可能性，因此允许 MCP 深入 agent 运行时。

## 工程启示

如果你要在自己的系统里接入 MCP 或类似协议，Claude Code 给出的启示很直接。

第一，不要只设计远程工具桥。把资源、提示、指令等上下文能力一并纳入远程能力面。

第二，把连接管理当成运行时子系统，而不是初始化细节。远程能力的可用性本身就是状态。

第三，让配置、连接和能力暴露分层。这样系统才能承受 transport、多 server 与多作用域带来的复杂性。

第四，预留 agent 级能力定制。远程能力一旦进入多代理系统，就不应默认全局无差别暴露。

## 思考题

1. 如果 MCP 只提供工具，不提供 prompts、resources 和 instructions，Claude Code 会在哪些地方失去统一性？
2. 为什么把连接管理做成独立子系统，比把所有逻辑塞进一个 connector 文件更适合长期演进？
3. 当子 agent 可以拥有不同 MCP 能力面时，权限和上下文治理会增加哪些新要求？

## 本章不覆盖什么

本章不深入 MCP 协议报文细节，不讨论外部 server 的具体实现，也不把不存在于当前工作树的文件写成已验证事实。这里关注的是 Claude Code 如何把 MCP 用成远程能力总线。
