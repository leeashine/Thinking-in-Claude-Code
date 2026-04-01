# CH07 AgentTool：子代理如何被定义、启动与回收

状态：`completed`

## 一句话摘要

本章将解释 Claude Code 怎样把子代理做成受控、可回收、可继承上下文的运行时单元。

## 问题引入

很多人第一次读 Claude Code 的子代理实现，都会产生一个直觉：`AgentTool` 不就是再起一个 `query()` 吗？如果只看表面调用，这个判断并不完全错，因为源码里确实会在子代理路径上再次进入 `runAgent()`，而 `runAgent()` 又会进入 `query()`。但如果停在这里，整章的重点就会被看丢。Claude Code 真正做的，不是“把任务再问一次模型”，而是“构造一个新的、受边界约束的执行主体”。

这个差异决定了系统复杂度的来源。如果子代理只是递归调用，那么它天然继承父代理的一切：上下文、权限、工具、UI、回收责任、缓存策略都不需要重新定义。可 Claude Code 并没有这样做。它先用 agent 定义描述出子代理的边界，再在 `AgentTool` 里按 MCP、权限和团队约束筛掉不可用候选，随后通过 `createSubagentContext()` 切开可变状态，最后由 `runAgent()` 装配系统提示词、工具池、MCP、hooks、transcript 和清理逻辑。换句话说，子代理不是函数栈帧，而是一个小型运行时。

因此，本章关心的问题不是“如何调用子代理”，而是“Claude Code 为什么必须把子代理做成受控运行时”。只有先把这个问题讲清，后面关于 Team Agent、共享 Task List、leader approval 和 mailbox 的设计才会显得自然。否则，多代理系统看起来只是多开几个会话窗口，读不出它真正的工程内核。

## 思想命题

### 命题一：子代理不是一次递归，而是先被定义出来的运行时单元

`loadAgentsDir.ts` 里的 `AgentDefinition` 并不只是“给模型起个名字”。它把 `tools`、`disallowedTools`、`mcpServers`、`hooks`、`permissionMode`、`maxTurns`、`background`、`skills`、`memory`、`isolation` 等信息一起编译进 agent 定义。也就是说，Claude Code 并不是在调用 `AgentTool` 时才临时决定子代理能做什么，而是在定义阶段就把它当成一个可配置、可裁剪、可审计的执行单元。

### 命题二：子代理不是父代理的复制品，而是被显式切边之后的新上下文

`createSubagentContext()` 最关键的工作不是“复制 context”，而是决定哪些状态可以继承，哪些必须克隆，哪些必须变成 no-op。`readFileState`、`contentReplacementState`、`agentId`、`queryTracking.depth`、`localDenialTracking` 的处理方式都不一样。Claude Code 不追求完整复刻父代理，而是追求“在必要继承之上建立新边界”。

### 命题三：子代理不是一段临时执行，而是有完整启动和回收协议的生命周期对象

`runAgent()` 在进入 `query()` 之前，要先装配系统提示词、技能、hooks、agent 专属 MCP、metadata 和 sidechain transcript；退出时，又必须在 `finally` 中清理 MCP、session hooks、prompt cache tracking、读文件缓存、transcript subdir、todos 和后台 shell task。这里的重点不是“写得很细”，而是 Claude Code 把子代理当成会占用资源、会污染全局状态、会留下后台进程的真实运行时对象来治理。

## 源码剖面

第一层是定义阶段。`loadAgentsDir.ts` 中的 `AgentDefinition` 联合类型把内建 agent、用户自定义 agent 和插件 agent 统一成同一套描述结构。这里最值得注意的不是字段数量，而是字段的工程含义。`permissionMode` 决定子代理进入什么权限语义，`background` 决定是否允许异步脱离当前交互节奏，`isolation` 决定是否进入 worktree 或更强的隔离路径，`mcpServers` 和 `skills` 决定它能接入哪些外部能力和提示资产。也就是说，子代理在真正运行前，已经先被“描述”为一种受约束的执行形态。

第二层是可见性与选择阶段。`AgentTool.prompt()` 不会把全部 agent 原样交给模型。它先扫描当前工具池，提取实际可见的 MCP server，再用 `filterAgentsByMcpRequirements()` 去掉依赖条件不满足的 agent，最后再用 `filterDeniedAgents()` 套一层权限过滤。这里体现出一个关键思想：不是 agent 被定义出来就一定可用，而是 agent 必须在“当前环境真实拥有的能力”和“当前权限规则允许的边界”之内才可被选择。Claude Code 把“能不能被模型看到”本身做成了一道运行时筛选。

第三层是调度阶段。`AgentTool.call()` 进入后，并不立刻执行子代理，而是先判断这次调用究竟属于哪一种路径。若同时给了 `team_name` 和 `name`，它会走 `spawnTeammate()`，这已经不是普通 subagent，而是 team member；若在 teammate 上又试图继续生成 teammate，源码会直接报错，因为 team roster 是扁平的；若 in-process teammate 试图启动后台 agent，也会被拒绝，因为其生命周期绑在 leader 进程上。然后，普通 subagent 路径还要继续判定是否使用 fork path、是否启用后台执行、是否进入 worktree isolation、是否带 `cwd` 覆盖。这里说明 `AgentTool` 的作用不是“转发 prompt”，而是“把不同代理运行形态分流到不同执行轨道”。

第四层是上下文切边。`runAgent()` 会根据 agent 定义和当前会话状态计算 `agentGetAppState()`：它可能覆盖权限模式，也可能为异步 agent 设置 `shouldAvoidPermissionPrompts`，还可能把 `allowedTools` 写成 session allow rules，防止父会话已批准的工具权限无意泄漏给子代理。随后，`createSubagentContext()` 真正创建新 context。它默认克隆 `readFileState`，重新生成 `agentId`，把 `queryTracking.depth` 加一，把 `setAppState` 在异步场景下变成 no-op，却又保留 `setAppStateForTasks` 以保证后台 bash 任务还能注册和回收；它还单独创建 `localDenialTracking`，以免异步子代理的拒绝计数失真。这里最重要的不是某一项字段怎么处理，而是 Claude Code 清楚地区分了“共享哪些能力是安全的”与“共享哪些能力会产生污染”。

第五层是启动阶段。`runAgent()` 在进入 `query()` 前做了大量装配工作。它要选择最终模型，构造 system prompt，处理 `omitClaudeMd` 这类 token 成本优化策略，执行 `SubagentStart` hooks，把 hooks 追加上下文变成新的用户消息，预加载 frontmatter 中声明的 skills，再通过 `initializeAgentMcpServers()` 启动 agent 专属 MCP server 并把这些工具与当前工具池去重合并。随后，它把这些全部编织进新的 `agentOptions` 和 `agentToolUseContext`，并用 `recordSidechainTranscript()` 与 `writeAgentMetadata()` 把初始状态落成 sidechain transcript 与 metadata。这些动作共同说明：Claude Code 理解的“启动子代理”是一整套运行时准备，而不是一句 API 调用。

第六层是执行与回收阶段。`runAgent()` 的主循环会逐条转发 `query()` 产生的消息，同时把可记录消息持续写入 sidechain transcript；遇到 `max_turns_reached` 这样的结构化 attachment 时，还会用专门的分支处理退出。真正体现工程成熟度的，是它的 `finally`。不论正常结束、报错还是 abort，都会执行 MCP cleanup、clearSessionHooks、cleanupAgentTracking、清空克隆出的文件缓存、释放 transcript subdir 映射、从 `AppState.todos` 删除该 agent 条目，并且杀掉这个 agent 产生的后台 shell 任务。源码注释写得很直白：如果不这样做，后台 bash 会变成 PPID=1 的孤儿，todos 也会在鲸鱼会话中累积成小泄漏。这里可以看到 Claude Code 的一个鲜明倾向：它把“代理结束”理解为资源回收事件，而不是“模型答完了”。

第七层是 fork 特例。`AgentTool` 在某些条件下会进入 fork path，并调用 `buildForkedMessages()` 构造特殊消息前缀，目的是让子代理保留足够多的父上下文字节级结构，从而命中 prompt cache，同时又通过查询源和 boilerplate tag 防止 fork child 继续 fork。`createSubagentContext()` 对 `contentReplacementState` 的默认策略也是“克隆而不是重建”，原因同样与 cache-safe prefix 有关。这说明 Claude Code 连“上下文复制”都不是朴素复制，而是围绕缓存一致性和边界约束做过专门设计。

## 设计取舍

Claude Code 在子代理设计上做了一个很重的选择：宁可把系统做复杂，也不把子代理当作“父代理能力的简单拷贝”。这个选择直接带来了实现代价。源码里因此出现了 agent 定义加载、过滤、权限覆盖、上下文克隆、系统提示词拼装、sidechain transcript、hooks 生命周期、MCP 生命周期、后台任务清理等多个层面。如果只追求“更快把任务再交给模型一次”，这些部件几乎都可以省掉。

但省掉之后，代价会转移到更隐蔽、更危险的地方。没有工具裁剪，父会话的批准规则就会无意泄漏；没有 `shouldAvoidPermissionPrompts`，异步 agent 可能在错误时机抢占 UI；没有 context cloning，子代理对文件状态和内容替换的改动就会污染父代理；没有 transcript 和 metadata，后台 agent 就无法恢复；没有 `finally` 清理，系统会持续积累 hooks、todos、shell 任务和缓存状态。Claude Code 选择的不是“最轻的代理模型”，而是“最可控的代理模型”。

这里的 trade-off 可以概括为一句话：把子代理做成真正的运行时，前期工程成本高，但后续的权限边界、恢复能力、缓存稳定性和资源治理都更可预测。对一个准备继续长出 Team Agent、Task List、remote runtime 的系统而言，这种重设计几乎是必需品。

## 工程启示

第一，设计子代理系统时，要先把“定义”从“调用”中拆出来。只有当能力、权限、隔离、背景执行这些特征先成为声明式结构，运行时才有可能稳定筛选和治理。

第二，父子代理之间不要只有“继承”一个动作，而要有“继承、克隆、重建、禁止”四种关系。真正可靠的多代理系统，核心不在于共享多少，而在于哪些边界必须重新建立。

第三，异步代理一定要有显式的 UI 与权限策略。能否显示 permission prompt、是否允许使用父会话批准过的工具、后台任务如何登记和回收，都必须单独设计。

第四，把 transcript、metadata 和 cleanup 当成运行时协议的一部分，而不是日志细节。一个会生成后台工作和跨回合状态的代理系统，如果没有稳定的回收路径，问题只会从功能错误变成资源泄漏。

第五，如果系统要做 fork、resume、background summarization 一类能力，就必须把 prompt cache 稳定性当成一等约束，而不是事后优化项。

## 思考题

1. 一个子代理到底应该继承父代理的哪些状态，哪些状态必须克隆，哪些状态必须禁用？
2. 如果取消 `allowedTools` 覆盖机制，只继承父代理已批准的工具规则，会出现哪些越权风险？
3. 为什么异步子代理不能简单复用父代理的 `setAppState` 和交互能力？
4. 对一个长时间运行的代理系统而言，`finally` 中的资源回收和功能正确性哪个更重要？
5. 如果你也要实现 fork path，哪些上下文必须保持字节级稳定，哪些上下文反而必须重新生成？

## 本章不覆盖什么

本章只讨论单个子代理如何被定义、启动、隔离与回收，不展开 Team Agent 的 swarm 协作、不展开 mailbox 与 permission bridge，也不展开共享 Task List 的认领和阻塞传播。这些内容分别留给 CH08 和 CH09。
