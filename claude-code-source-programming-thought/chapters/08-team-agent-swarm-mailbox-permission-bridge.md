# CH08 Team Agent：swarm、mailbox、permission bridge

状态：`completed`

## 一句话摘要

本章将解释 team agent 为什么需要 swarm、邮箱和权限桥三件套才能真正协作。

## 问题引入

如果只从功能表面理解 Team Agent，它看起来像是“给 Claude Code 多开几个代理，再让它们互相发消息”。这种理解会把卷三写薄。因为从源码看，Team Agent 并不是几个子代理的堆叠，而是一套新的协作运行时：它先定义团队身份边界，再把代理之间的通信降成结构化 mailbox 协议，然后把权限裁决重新收拢到 leader 的 UI，最后再由常驻 runner 把 teammate 变成可持续调度的执行主体。

这套设计回答的是一个多代理系统里非常硬的问题：当多个执行主体同时存在时，谁拥有团队身份？消息走什么传输层？权限由谁审批？worker 空闲后如何再次被唤醒？如果没有这些答案，多代理只会退化成多个互不相干的会话窗口，协作只能依赖 prompt 中的道德自觉。Claude Code 没有这样处理。它把团队协作写成了 runtime protocol。

因此，本章的目标不是介绍“怎么创建 team”，而是解释 Claude Code 为什么需要 `TeamCreateTool`、`teammateMailbox`、`permissionSync`、`leaderPermissionBridge` 和 `inProcessRunner` 这些看起来很重的部件。只有把这几层拼起来，Team Agent 才从“多个 agent”变成“一个可协调的 swarm”。

## 思想命题

### 命题一：Team Agent 不是多个代理的集合，而是先由 team identity 建立出的协作边界

`TeamCreateTool` 做的第一件事不是启动智能，而是写 team file、重置 task list、调用 `setLeaderTeamName()`，并把 `teamContext` 放进 leader 的 AppState。也就是说，所谓 swarm 并不是一开始就体现在消息交互里，而是先体现在“大家属于同一个命名空间、同一个协作目录、同一个 leader 上下文”这件事上。

### 命题二：邮箱不是聊天附属物，而是团队控制平面

`teammateMailbox.ts` 里维护的不是一个简单收件箱，而是一套结构化团队协议。`permission_request`、`permission_response`、`plan_approval_request`、`shutdown_request`、`idle_notification` 这些消息说明 mailbox 承载的是运行时控制流，不只是普通文本对话。Claude Code 把“协作”降成可落盘、可轮询、可重放的文件协议。

### 命题三：Team Agent 的权限不分布式地下放，而是重新收束到 leader

`leaderPermissionBridge.ts` 的桥非常薄，但意义非常重。它没有创造新的权限系统，只是把 leader REPL 持有的 `ToolUseConfirm` 队列和 `toolPermissionContext` setter 暴露给 in-process teammate。于是 worker 可以发起 ask，但最终裁决和权限更新仍回到 leader UI。这说明 Team Agent 追求的不是“人人都有独立权限中心”，而是“多执行主体共享一个权限权威”。

## 源码剖面

第一层是团队建立。`TeamCreateTool.call()` 在创建 team 时会生成 team file、准备协作目录、重置对应 task list，并通过 `setLeaderTeamName()` 把 leader 的后续任务解析绑定到这个团队名。它还会把 `teamContext` 放进 AppState，让 leader 明确知道当前正处于哪个 team 之中。这里可以看出一个很重要的设计取向：Claude Code 先建立团队身份边界，再讨论具体成员如何运行。没有这一步，后面的 mailbox、task list、permission routing 都找不到统一命名空间。

第二层是 teammate 的运行形态。创建 team 本身不等于 teammate 已经开始协作，后续还要由 teammate spawn 路径把成员拉起。对于 in-process teammate，`inProcessRunner.ts` 并没有重新发明一套执行器，而是把 `runAgent()` 包进 `runWithTeammateContext()` 中运行，并在外围附加进度追踪、idle 通知、plan approval 支持和完成清理。这里体现出 Claude Code 的一个稳定倾向：Team Agent 不是抛弃普通 agent runtime，而是在其外面加一层 team-aware 的上下文和生命周期。

第三层是邮箱协议。`teammateMailbox.ts` 把 mailbox 落在 `.claude/teams/{team}/inboxes/{agent}.json` 这样的文件路径上，并通过读写锁保证并发追加和已读标记的串行化。更关键的是，这个邮箱承载的消息不只是自然语言。源码里显式存在 `permission_request/response`、`plan_approval_request/response`、`shutdown_*`、`mode_set_request`、`idle_notification` 等结构化消息。也就是说，Claude Code 的 mailbox 更像一个轻量 IPC 控制面，而不是聊天记录容器。

第四层是 leader permission bridge。`leaderPermissionBridge.ts` 很短，只做注册和读取两个 setter：一个用于把新的 `ToolUseConfirm` 项注入 leader REPL 的确认队列，一个用于把批准后产生的 permission updates 写回 leader 的 `toolPermissionContext`。桥本身没有业务逻辑，却把 React UI 世界和 in-process runner 连接起来。它的意义在于：worker 不需要各自渲染一份权限 UI，也不能各自维护一套分裂的批准状态，team 内 ask 权限被统一收编到 leader 的确认流程中。

第五层是 in-process 权限处理。`createInProcessCanUseTool()` 遇到 `allow` 和 `deny` 时会直接透传，只有 `ask` 才进入 team 协作路径。对于 Bash，它会先尝试 classifier 自动放行；若仍需要人工批准，就优先取 `getLeaderToolUseConfirmQueue()`，把标准 `ToolUseConfirm` 对象塞进 leader 的队列，并附带 worker badge，让 leader 在同一套 UI 中看到是哪个 teammate 发起了权限请求。用户批准后，`persistPermissionUpdates()` 会落盘，随后 bridge 还会把更新后的 permission context 写回 leader，但显式保留 leader 的 mode，防止 worker 的变换反向污染 coordinator。这里说明 bridge 并不是“帮 worker 点一下按钮”，而是把 worker 的 ask 权限纳入 leader 的真实权限上下文。

第六层是 mailbox fallback。Claude Code 并没有假定 bridge 永远存在。若 `getLeaderToolUseConfirmQueue()` 取不到，`inProcessRunner` 就会退到 `permissionSync.ts` 提供的协议：worker 调 `createPermissionRequest()` 生成 `SwarmPermissionRequest`，经 `sendPermissionRequestViaMailbox()` 发给 leader inbox，再轮询自己的 mailbox 等 `permission_response`。leader 侧的 inbox poller 会把请求重新提升为 `ToolUseConfirm`，审批完成后再写回 worker mailbox。这里可以看到一个很清楚的分层：UI 体验尽量统一，但传输层允许直连桥和邮箱握手两条路径共存。

第七层是常驻 teammate 状态机。`inProcessRunner` 不是一次跑完就销毁的函数。它在一轮执行结束后会把 teammate 标成空闲，向 leader 发送 `idle_notification`，然后进入等待下一个 prompt 或 shutdown 的轮询循环。源码里连消息优先级都被写死了：`shutdown` 高于 leader message，高于 peer message，再高于 task prompt。与此同时，teammate 默认拿到一组 team-essential tools，但其输出并不会自动回流给 leader，必须显式通过消息工具发送。这意味着 Team Agent 不是广播式心灵感应，而是一个带身份、带优先级、带显式唤醒机制的协作状态机。

## 设计取舍

Claude Code 在 Team Agent 上的核心选择，是不用一个“超强 coordinator prompt”去软性管理多个 worker，而是建立 team identity、mailbox 协议、leader permission bridge 和常驻 runner 这几层硬边界。代价很明显：实现更重，文件协议更多，状态也更分散，光读源码就能看到 team file、inboxes、permission requests、REPL queue、runner loop、poller 之间的多处耦合。

但收益同样清楚。首先，权限不会被分发到每个 worker 自行裁决，安全边界保持集中。其次，协作不依赖单进程内存；就算 bridge 不可用，mailbox fallback 仍能维持同一套语义。再次，teammate 可以作为长期存在的团队成员被唤醒、暂停、请求审批或关闭，而不是一次性工具调用。Claude Code 用更重的运行时结构，换来了更稳的团队一致性。

从工程角度看，这个 trade-off 很合理。只要系统准备继续支持 in-process teammate、外部 pane teammate、plan approval、task coordination 等多条路径，就必须有一个比“多个 prompt”更硬的协作底座。

## 工程启示

第一，做多代理协作时，先定义团队边界，再定义成员行为。没有 team identity，后续所有共享目录、消息路由和任务命名空间都会飘。

第二，把代理间协作降成结构化协议，而不是任由模型自由聊天。文件邮箱、消息类型、已读语义和轮询器虽然朴素，却比隐式上下文共享更可靠。

第三，权限系统最好保持单一权威。worker 可以发起 ask，但批准、拒绝和规则写回应该落在同一个中心，否则团队很快会出现权限漂移。

第四，尽量让团队协作复用单代理运行时，而不是完全另起炉灶。这样 team 只是上下文和协议的扩展层，不会变成第二套彼此分裂的系统。

第五，如果系统同时存在直连桥和消息回退路径，就要明确两者的职责边界：UI 可以统一，传输层不必统一。

## 思考题

1. 如果没有 `TeamCreateTool` 建立出的统一 team identity，mailbox 和 task list 会遇到哪些路由歧义？
2. 为什么 Team Agent 的 mailbox 更适合被理解为控制平面，而不是聊天通道？
3. 在多代理系统里，权限为什么应该尽量收束到 leader，而不是分散到每个 worker？
4. 如果去掉 mailbox fallback，只保留 in-process permission bridge，这套协作语义还能跨运行形态成立吗？
5. 一个 teammate 执行结束后为什么不能直接消失，而要进入 idle、等待下一次唤醒？

## 本章不覆盖什么

本章聚焦 Team Agent 的协作内核，只讨论 team identity、mailbox、permission bridge 和 runner loop，不展开单个子代理生命周期，不展开共享 Task List 的 claim/block/owner 规则，也不展开远程会话与跨 session 边界。这些内容分别留给 CH07、CH09 和后续章节。
