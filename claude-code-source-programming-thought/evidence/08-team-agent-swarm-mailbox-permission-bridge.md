# CH08 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Team Agent 不是多开几个子代理，而是由 `team identity + mailbox protocol + leader-owned permission bridge + runner loop` 组成的协作运行时。
- 读者收益：理解 swarm 的最小内核是什么，为什么权限必须回到 leader，为什么 mailbox 是控制平面而不只是聊天记录。
- 本章排除内容：不展开单个子代理生命周期，不展开共享 Task List 的 claim/block 算法，不展开远程 session 边界。
- 关键源码清单：`tools/TeamCreateTool/TeamCreateTool.ts`、`utils/swarm/inProcessRunner.ts`、`utils/swarm/leaderPermissionBridge.ts`、`utils/swarm/permissionSync.ts`、`utils/teammateMailbox.ts`、`tools/SendMessageTool/SendMessageTool.ts`。

## 源码锚点

- `TeamCreateTool.ts`
  - 原因：建立 team identity 与协作命名空间。
  - 关注点：team file、`resetTaskList()`、`setLeaderTeamName()`、`teamContext`。
- `teammateMailbox.ts`
  - 原因：团队 IPC 与控制消息存储层。
  - 关注点：按 `team + agent` 路由的 inbox、文件锁、结构化消息类型、已读标记。
- `leaderPermissionBridge.ts`
  - 原因：把 leader REPL 的权限 UI 能力桥接给 in-process teammate。
  - 关注点：`registerLeaderToolUseConfirmQueue()`、`registerLeaderSetToolPermissionContext()`、模块级 setter 存取。
- `inProcessRunner.ts`
  - 原因：teammate 常驻循环与权限 ask 的核心运行时。
  - 关注点：`createInProcessCanUseTool()`、classifier 快路径、leader queue 注入、mailbox fallback、idle/shutdown/message 优先级。
- `permissionSync.ts`
  - 原因：bridge 不可用时的权限同步协议。
  - 关注点：`SwarmPermissionRequest`、`createPermissionRequest()`、worker 发 leader、leader 回 worker 的异步握手。
- `SendMessageTool.ts`
  - 原因：把 mailbox 暴露成可用的控制面 API。
  - 关注点：普通消息、广播、shutdown、plan approval 等结构化消息入口。

## 主调用链

1. `TeamCreateTool.call()` 建立 team file、共享命名空间和 leader `teamContext`。
2. teammate 被拉起后进入 `inProcessRunner`，在 `runWithTeammateContext()` 中复用 `runAgent()` 执行。
3. teammate 与 leader、peer 之间的协作消息通过 `teammateMailbox` 的文件 inbox 落盘和轮询。
4. worker 遇到 `ask` 权限时，`createInProcessCanUseTool()` 优先把 `ToolUseConfirm` 注入 leader 队列，并附加 worker badge。
5. leader 批准后，通过 bridge 把 permission updates 写回 leader 的权限上下文，再直接 resolve worker 的等待。
6. 若 bridge 不可用，worker 使用 `permissionSync.ts` 生成请求并经 mailbox 异步等待 `permission_response`。
7. 一轮执行结束后，runner 发送 `idle_notification`，进入下一轮等待、继续接收 leader/peer/task 触发。

## 关键 trade-off

- 约束：多代理系统若只靠 prompt 协作，会在权限、消息路由和长期成员状态上迅速失控。
- 方案：建立 team identity、文件邮箱、leader permission bridge 和常驻 runner。
- 代价：实现更重，需要额外的消息协议、轮询器、状态同步和 UI 桥接。
- 收益：权限权威集中、协作可落盘、运行形态可退化、teammate 可持续被调度而不是一次性调用。

## 事实校验

- 直接来自实现：team file 与 `teamContext` 初始化、文件邮箱路径与锁、结构化 mailbox 消息、leader queue/context bridge、bridge 快路径与 mailbox fallback、idle 通知与等待循环。
- 属于推断：把这套机制概括为“协作运行时”与“控制平面”是对多个模块共同职责的结构性总结。
- 需要避免夸大：
  - 不能把 mailbox 写成分布式消息总线；实现本质上是文件锁加轮询的轻量 IPC。
  - 不能说 worker 自己拥有权限裁决权；最终批准仍回到 leader UI 和 leader 上下文。
  - 不能把 permission bridge 说成全部权限同步机制；bridge 只是快路径，mailbox fallback 仍是正式协议的一部分。
