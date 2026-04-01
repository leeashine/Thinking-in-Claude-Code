# TASKS

状态只使用：`pending`、`in_progress`、`completed`。

## P0 基线资产

- `completed` `T00` 冻结书名、副标题、章节命名规则、术语表基线。
- `completed` `T01` 完成全书总纲、卷结构、每章一句话摘要。
- `completed` `T02` 完成章节模板、evidence 模板、写作风格规范。
- `completed` `T03` 初始化全局 `TASKS.md`、章节编号与目录结构。

## P1 卷一

- `completed` `CH01-A` brainstorm：为什么 Claude Code 值得用“编程思想”来阅读。
- `completed` `CH01-B` evidence harvest：收集运行时、扩展、协作、上下文四条主线的总证据。
- `completed` `CH01-C` call-chain tracing：还原全书级主调用链。
- `completed` `CH01-D` chapter draft：完成正文初稿。
- `completed` `CH01-E` fact check：逐条回指源码。
- `completed` `CH01-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH02-A` brainstorm：REPL 与会话边界。
- `completed` `CH02-B` evidence harvest：REPL、PromptInput、session state、UI shell。
- `completed` `CH02-C` call-chain tracing：用户输入进入主循环的路径。
- `completed` `CH02-D` chapter draft：完成正文初稿。
- `completed` `CH02-E` fact check：逐条回指源码。
- `completed` `CH02-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH03-A` brainstorm：`query` 主循环与消息驱动运行时。
- `completed` `CH03-B` evidence harvest：query loop、tool orchestration、budget/compact。
- `completed` `CH03-C` call-chain tracing：一轮 query 的状态推进。
- `completed` `CH03-D` chapter draft：完成正文初稿。
- `completed` `CH03-E` fact check：逐条回指源码。
- `completed` `CH03-F` polish and cross-link：补交叉链接与边界说明。

## P2 卷二

- `completed` `CH04-A` brainstorm：Tool 抽象、工具池与调度。
- `completed` `CH04-B` evidence harvest：Tool 合同、工具池装配、结果回写。
- `completed` `CH04-C` call-chain tracing：tool_use 分批、执行与上下文提交。
- `completed` `CH04-D` chapter draft：完成正文初稿。
- `completed` `CH04-E` fact check：逐条回指源码。
- `completed` `CH04-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH05-A` brainstorm：权限不是附属功能，而是运行时边界。
- `completed` `CH05-B` evidence harvest：权限策略、交互编排、模式迁移。
- `completed` `CH05-C` call-chain tracing：allow / deny / ask 与审批路径。
- `completed` `CH05-D` chapter draft：完成正文初稿。
- `completed` `CH05-E` fact check：逐条回指源码。
- `completed` `CH05-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH06-A` brainstorm：Plan Mode 作为工程协议。
- `completed` `CH06-B` evidence harvest：进入、计划文件、退出审批、恢复执行。
- `completed` `CH06-C` call-chain tracing：plan 模式的状态迁移闭环。
- `completed` `CH06-D` chapter draft：完成正文初稿。
- `completed` `CH06-E` fact check：逐条回指源码。
- `completed` `CH06-F` polish and cross-link：补交叉链接与边界说明。

## P3 卷三

- `completed` `CH07-A` brainstorm：AgentTool 与子代理生命周期。
- `completed` `CH07-B` evidence harvest：agent 定义、工具裁剪、上下文隔离、生命周期清理。
- `completed` `CH07-C` call-chain tracing：`AgentTool` 到 `runAgent()` 再到 cleanup 的主链。
- `completed` `CH07-D` chapter draft：完成正文初稿。
- `completed` `CH07-E` fact check：逐条回指源码。
- `completed` `CH07-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH08-A` brainstorm：Team Agent 的协作内核。
- `completed` `CH08-B` evidence harvest：team identity、mailbox、permission bridge、runner loop。
- `completed` `CH08-C` call-chain tracing：建队、拉起 teammate、权限审批与 idle 回流。
- `completed` `CH08-D` chapter draft：完成正文初稿。
- `completed` `CH08-E` fact check：逐条回指源码。
- `completed` `CH08-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH09-A` brainstorm：共享 Task List 的分工与收敛。
- `completed` `CH09-B` evidence harvest：共享账本、锁、认领、阻塞与前台投影。
- `completed` `CH09-C` call-chain tracing：team 绑定、任务创建、认领、推进与收束。
- `completed` `CH09-D` chapter draft：完成正文初稿。
- `completed` `CH09-E` fact check：逐条回指源码。
- `completed` `CH09-F` polish and cross-link：补交叉链接与边界说明。

## P4 卷四

- `completed` `CH10-A` brainstorm：System Prompt 分层装配。
- `completed` `CH10-B` evidence harvest：system/user 分流、section 装配、缓存边界。
- `completed` `CH10-C` call-chain tracing：`context.ts -> prompts.ts -> query.ts -> runAgent.ts`。
- `completed` `CH10-D` chapter draft：完成正文初稿。
- `completed` `CH10-E` fact check：逐条回指源码。
- `completed` `CH10-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH11-A` brainstorm：Memory 体系与相关性选择。
- `completed` `CH11-B` evidence harvest：多来源发现、过滤、相关性召回、附件注入。
- `completed` `CH11-C` call-chain tracing：`claudemd -> relevant memories -> query attachments`。
- `completed` `CH11-D` chapter draft：完成正文初稿。
- `completed` `CH11-E` fact check：逐条回指源码。
- `completed` `CH11-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH12-A` brainstorm：Compact、Token Budget 与上下文治理。
- `completed` `CH12-B` evidence harvest：预算控制、micro compact、auto compact、post-compact 重建。
- `completed` `CH12-C` call-chain tracing：`query -> tokenBudget -> microCompact -> autoCompact -> compact`。
- `completed` `CH12-D` chapter draft：完成正文初稿。
- `completed` `CH12-E` fact check：逐条回指源码。
- `completed` `CH12-F` polish and cross-link：补交叉链接与边界说明。

## P5 卷五

- `completed` `CH13-A` brainstorm：统一扩展模型。
- `completed` `CH13-B` evidence harvest：命令协议、技能加载、插件装配。
- `completed` `CH13-C` call-chain tracing：`commands -> skills/plugins -> command surface`。
- `completed` `CH13-D` chapter draft：完成正文初稿。
- `completed` `CH13-E` fact check：逐条回指源码。
- `completed` `CH13-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH14-A` brainstorm：MCP 远程能力总线。
- `completed` `CH14-B` evidence harvest：连接管理、配置解析、远程能力面、agent 集成。
- `completed` `CH14-C` call-chain tracing：`config -> client -> connection manager -> runtime exposure`。
- `completed` `CH14-D` chapter draft：完成正文初稿。
- `completed` `CH14-E` fact check：逐条回指源码。
- `completed` `CH14-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH15-A` brainstorm：远程会话、IDE 与外部边界。
- `completed` `CH15-B` evidence harvest：remote session、permission bridge、IDE 发现、外部消息桥接。
- `completed` `CH15-C` call-chain tracing：`setup -> remote session -> adapter -> local permission/IDE bridge`。
- `completed` `CH15-D` chapter draft：完成正文初稿。
- `completed` `CH15-E` fact check：逐条回指源码。
- `completed` `CH15-F` polish and cross-link：补交叉链接与边界说明。

## P6 卷六

- `completed` `CH16-A` brainstorm：观测、诊断与恢复。
- `completed` `CH16-B` evidence harvest：analytics、API logging、diagnostic tracking、session recovery。
- `completed` `CH16-C` call-chain tracing：`query/api logging -> recovery -> diagnostics -> session resume`。
- `completed` `CH16-D` chapter draft：完成正文初稿。
- `completed` `CH16-E` fact check：逐条回指源码。
- `completed` `CH16-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH17-A` brainstorm：面向约束的工程哲学。
- `completed` `CH17-B` evidence harvest：依赖环、权限管线、预算控制器、锁与缓存边界。
- `completed` `CH17-C` call-chain tracing：`type boundary -> permission gate -> task lock -> budget/compact guard`。
- `completed` `CH17-D` chapter draft：完成正文初稿。
- `completed` `CH17-E` fact check：逐条回指源码。
- `completed` `CH17-F` polish and cross-link：补交叉链接与边界说明。
- `completed` `CH18-A` brainstorm：设计你自己的 Claude Code 型系统。
- `completed` `CH18-B` evidence harvest：能力编目、主循环、工具 ABI、子 agent、任务图、上下文治理。
- `completed` `CH18-C` call-chain tracing：`commands -> query -> tools -> agents -> tasks -> compact/recovery`。
- `completed` `CH18-D` chapter draft：完成正文初稿。
- `completed` `CH18-E` fact check：逐条回指源码。
- `completed` `CH18-F` polish and cross-link：补交叉链接与边界说明。

## P7 收束

- `completed` `A01` 前言与阅读指南。
- `completed` `A02` 术语表基线。
- `completed` `A03` 源码索引基线。
- `completed` `A04` 全书一致性审校与总稿装配。
