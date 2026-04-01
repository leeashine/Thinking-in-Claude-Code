# CH14 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：MCP 在 Claude Code 中承担远程能力总线角色，不只暴露工具，还连接 prompt、resource、instructions 与 agent 运行时。
- 读者收益：理解为什么远程能力整合需要连接管理、配置层和 agent 集成层，而不是一个简单 connector。
- 本章排除内容：不展开协议报文，不讨论外部 MCP server 实现，不虚构当前工作树中不存在的文件。
- 关键源码清单：[services/mcp/client.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/client.ts)、[services/mcp/MCPConnectionManager.tsx](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/MCPConnectionManager.tsx)、[services/mcp/config.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/config.ts)、[services/mcp/types.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/types.ts)、[tools/AgentTool/runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)。

## 源码锚点

- [client.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/client.ts)：远程客户端交互与能力消费入口。
- [MCPConnectionManager.tsx](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/MCPConnectionManager.tsx)：连接生命周期与状态管理。
- [config.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/config.ts)：MCP 配置解析与结构化定义。
- [types.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/types.ts)：远程能力类型面。
- [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts)：MCP 能力进入 agent 运行时的路径。

## 主调用链

1. [config.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/config.ts) 解析远程 server 配置。
2. [client.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/client.ts) 建立连接并消费远端能力。
3. [MCPConnectionManager.tsx](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/mcp/MCPConnectionManager.tsx) 维护连接状态、重连与生命周期。
4. 远程 server 暴露的工具、prompts、resources、instructions 经类型层进入统一能力面。
5. 主运行时与 [runAgent.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/tools/AgentTool/runAgent.ts) 按需把这些能力暴露给主 agent 或子 agent。

## 关键 trade-off

Claude Code 承担了更高的连接与运行时复杂度，换取完整的远程能力面和 agent 级整合能力。系统不满足于“多几个远程工具”，而是把远端 server 当成可参与上下文与动作供给的能力节点。

## 事实校验

- “MCP 不止工具”来自相关 types、prompts 与 resources 路径的职责分布。
- “连接管理是独立子系统”来自 `MCPConnectionManager.tsx` 与 `client.ts` 的分层。
- “当前工作树没有 `McpConnector.ts`”来自实际文件检索结果，因此正文避免虚构单一核心文件。
