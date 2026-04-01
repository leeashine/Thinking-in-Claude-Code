# 源码索引

## 运行时主线

- `query.ts`：主循环与回合推进。
- `screens/REPL.tsx`：交互前台、输入处理、会话内 UI 编排。
- `QueryEngine.ts`：SDK / headless 场景下的会话引擎。
- `services/tools/toolOrchestration.ts`：工具调度。
- `services/tools/toolExecution.ts`：单个工具调用的执行与结果处理。
- `Tool.ts`：工具抽象与工具匹配基础。

## 多代理与团队协作

- `tools/AgentTool/runAgent.ts`：子代理运行入口。
- `tools/AgentTool/AgentTool.tsx`：Agent 工具定义。
- `utils/swarm/inProcessRunner.ts`：in-process teammate 的运行内核。
- `utils/swarm/permissionSync.ts`：team agent 权限同步。
- `utils/swarm/leaderPermissionBridge.ts`：leader 与 worker 的权限桥。
- `utils/teammateMailbox.ts`：邮箱通信。
- `utils/tasks.ts`：共享任务系统。
- `tools/TaskCreateTool/TaskCreateTool.ts`：任务创建入口。
- `tools/TeamCreateTool/TeamCreateTool.ts`：团队创建与 task list 绑定。

## 上下文与记忆

- `context.ts`：用户上下文与系统上下文装配。
- `constants/prompts.ts`：system prompt 分层装配与动态边界。
- `constants/systemPromptSections.ts`：prompt section 注册与缓存语义。
- `utils/systemPromptType.ts`：system prompt 协议类型。
- `utils/claudemd.ts`：CLAUDE.md 发现、过滤与聚合。
- `memdir/findRelevantMemories.ts`：相关记忆选择。
- `services/compact/compact.ts`：主要压缩策略。
- `services/compact/autoCompact.ts`：自动 compact 触发策略。
- `services/compact/microCompact.ts`：局部上下文压缩。
- `query/tokenBudget.ts`：回合预算追踪。

## 扩展机制

- `commands.ts`：命令装配总入口。
- `skills/loadSkillsDir.ts`：技能加载。
- `utils/plugins/pluginLoader.ts`：插件发现与缓存。
- `utils/plugins/loadPluginCommands.ts`：插件命令/技能装配。
- `services/mcp/client.ts`：MCP 连接与能力获取。
- `services/mcp/MCPConnectionManager.tsx`：MCP 连接管理。
- `services/mcp/config.ts`：MCP 配置解析。

## 权限与协议

- `tools/EnterPlanModeTool/EnterPlanModeTool.ts`：进入 plan mode。
- `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`：退出 plan mode 并请求批准。
- `utils/permissions/permissionSetup.ts`：权限上下文准备。
- `utils/permissions/permissions.ts`：权限判定核心。
- `types/permissions.ts`：权限低依赖类型边界。

## 观察与工程化

- `services/analytics/index.ts`：埋点入口。
- `services/api/logging.ts`：API 使用与日志。
- `services/diagnosticTracking.ts`：诊断追踪。
- `utils/sessionStorage.ts`：会话记录与恢复。
- `utils/conversationRecovery.ts`：会话恢复与一致性检查。
- `remote/RemoteSessionManager.ts`：远程会话状态管理。
- `remote/remotePermissionBridge.ts`：远程权限回流桥。
- `utils/ide.ts`：IDE 发现与连接。
