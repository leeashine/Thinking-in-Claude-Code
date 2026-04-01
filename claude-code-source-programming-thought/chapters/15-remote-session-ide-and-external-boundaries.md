# CH15 远程会话、IDE 与边界延伸

状态：`completed`

## 一句话摘要

Claude Code 并不把自己限制为本地终端进程，而是通过 remote session、permission bridge、IDE 发现与消息桥接，把会话边界延伸到外部环境。

## 问题引入

如果把 Claude Code 只理解为一个跑在本地终端里的 CLI，会很难解释源码里为何会出现 remote session manager、IDE bridge、permission bridge、WebSocket 会话流和 UDS messaging 之类模块。它们的存在说明，Claude Code 并不是一个只在单进程内自洽的系统，而是一个会主动向外延伸边界的 Agent runtime。

这种延伸不是简单“支持远程调用”四个字可以概括的。真正的问题是：当 Claude Code 走出本地 REPL 后，哪些能力仍留在本地，哪些状态要同步到远端，哪些权限必须回流到本地 UI 批准，哪些 IDE 诊断和文件上下文需要重新接入主循环？一旦这些问题不能被系统化回答，所谓“远程支持”很快就会退化成一堆无法治理的桥接脚本。

CH15 因此关注的不是单个桥，而是 Claude Code 如何把这些桥组织成“边界编排器”。

## 思想命题

本章的核心命题是：Claude Code 的边界延伸，不是把本地能力简单暴露给外部，而是重新划分“会话、权限、上下文、诊断和消息流”在本地与远端之间的归属。

第一，remote session 让 Claude Code 拥有远程会话状态，而不只是远程命令执行。也就是说，远端不是调用一个函数，而是在参与同一会话的持续消息流。

第二，权限始终是本地边界的一部分。即使工具调用发生在远端，审批与风险控制仍可能被桥接回本地 UI，这说明权限所有权没有被轻易下放。

第三，IDE 集成不是前端点缀，而是边界延伸的重要一环。只要系统要在 IDE 中发现诊断、工作区与会话关联，IDE 就已经成为 Claude Code 运行时的一部分。

第四，远程消息桥接需要显式 adapter。Claude Code 不是直接把所有远端消息扔回本地，而是通过适配层把不同来源的事件转换为本地运行时能理解的形式。

## 源码剖面

CH15 的核心文件之一是 [remote/RemoteSessionManager.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/RemoteSessionManager.ts)。从这个模块可以看出，Claude Code 对远程会话的理解不是“HTTP 发个请求拿结果”，而是区分了写入用户事件与订阅远端消息两条路径。也就是说，会话在远端是持续存在的，用户输入和系统输出会在这个远程会话对象上持续演化。

[remote/sdkMessageAdapter.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/sdkMessageAdapter.ts) 则说明，远程消息并不会原样进入本地运行时。Claude Code 需要一个 adapter，把远端 SDK 或服务端事件转换成当前 query loop、REPL 和工具层能够理解的消息结构。这个适配动作非常重要，因为一旦边界跨出本地进程，消息格式就不可能天然一致。

权限回流由 [remote/remotePermissionBridge.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/remotePermissionBridge.ts) 体现得最直接。这里的关键思想不是“远端也可以请求权限”，而是 Claude Code 会用 synthetic assistant/tool stub 等方式，把远端的工具调用尝试映射回本地审批流程。换句话说，即便动作发生在远端，本地仍保持风险审批主权。

IDE 相关边界则体现在 [utils/ide.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/ide.ts)。这个文件不只是做简单端口连接，而是通过 lockfile、端口探测、workspace 匹配等方式发现当前可连接的 IDE，再通过 WS/SSE 等端点建立桥接。这里说明 Claude Code 对 IDE 的理解也不是“附加展示面板”，而是把 IDE 当成可提供工作区语义、诊断信息和连接状态的外部协作面。

[setup.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/setup.ts) 与 `systemInit` 周边逻辑则说明这些桥接并非临时外挂，而是在系统启动阶段就被纳入考虑。需要特别保持事实边界的是：`setup.ts` 里有 `import('./utils/udsMessaging.js')` 的调用点，但当前工作树里没有直接找到对应的 `utils/udsMessaging.ts` 实体文件。因此，正文只能据此说明 Claude Code 存在 UDS messaging 启动点与边界桥接意图，而不能把该模块的内部实现写成已验证事实。

如果把这些文件连起来看，就能得到 CH15 的主链：本地系统启动时准备外部桥接能力，`RemoteSessionManager` 维护远程会话状态与消息流，`sdkMessageAdapter.ts` 把远端事件转成本地消息，`remotePermissionBridge.ts` 让远端工具尝试回流到本地权限控制，`ide.ts` 则把 IDE 工作区与诊断能力接回本地运行时。Claude Code 因此不只是“支持远程”，而是在编排一套跨边界的连续会话系统。

## 设计取舍

Claude Code 在这里的第一项取舍，是边界延伸能力与系统复杂度之间的平衡。把自己限制为本地 CLI 会更简单，但也会失去远程会话、IDE 诊断与跨环境协作面。Claude Code 选择承担复杂性，换更宽的系统边界。

第二项取舍，是远端执行自由度与本地权限主权之间的平衡。让远端完全自治最轻松，但风险也最大；Claude Code 明显更在意本地审批仍能兜底，因此设计了 permission bridge。

第三项取舍，是统一消息模型与适配开销之间的平衡。跨进程、跨 IDE、跨远端服务的消息天然异构，Claude Code 选择引入 adapter 与 bridge 层，而不是假设一切格式天然统一。

## 工程启示

如果你要把 Agent 从本地终端扩展到远端或 IDE，Claude Code 有三点非常值得迁移。

第一，不要只桥接工具调用，要桥接会话状态。远程能力若不能进入持续消息流，就无法真正参与主运行时。

第二，把权限留在你真正信任的边界里。即使执行面扩散到远端，审批权也不应该默认跟着外移。

第三，引入显式 adapter 层。边界越多，消息模型越不可能天然一致；适配不是多余，而是稳定性的前提。

第四，把 IDE 视为运行时的一部分，而不只是展示壳。工作区、诊断和会话关联都是高价值上下文来源。

## 思考题

1. 如果远端工具调用不再回流本地权限桥，Claude Code 的风险模型会发生什么变化？
2. 为什么 remote session 必须是持续会话而不是单次 RPC？这种差异会影响哪些恢复与诊断能力？
3. 当前工作树里未直接看到 UDS messaging 的实现文件时，写作者应如何界定“源码事实”和“合理推断”的边界？

## 本章不覆盖什么

本章不展开远端服务端实现，不详写 IDE 协议细节，也不把当前工作树中未见实体文件的模块当成已核实实现。这里关注的是 Claude Code 如何通过 remote session、bridge 和 IDE 集成延伸系统边界。
