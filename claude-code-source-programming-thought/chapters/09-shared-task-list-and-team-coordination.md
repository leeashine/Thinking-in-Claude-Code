# CH09 共享 Task List：多代理如何分工、认领、推进与收敛

状态：`completed`

## 一句话摘要

本章将解释共享任务台账如何成为多代理协作的显式协议，而不是附属提醒板。

## 问题引入

走到 Team Agent 这一步，系统最容易掉进的陷阱不是“不会发消息”，而是“虽然能发消息，却没有共享的工作事实”。如果多个 agent 只能互相聊天，却没有统一的任务账本，那么谁负责什么、谁已经接手、什么还被阻塞、谁做完后应该轮到谁，这些问题最终都会退化成 prompt 中的口头协商。协作看上去很热闹，实际上没有稳定秩序。

Claude Code 处理这个问题的方法非常直接：把分工和推进压到一份共享 Task List 上。任务不是 leader 脑中的待办，也不是某个 worker 私有的临时意图，而是落在统一目录中的一组 JSON 文件，并且带有 `status`、`owner`、`blocks`、`blockedBy`、`activeForm` 等显式字段。这样，多代理协作第一次从“彼此理解”变成了“共享事实”。

因此，本章真正要解释的不是 Task 工具的用法，而是 Claude Code 为什么要把任务系统做成一个可加锁、可认领、可阻塞、可投影到 UI 的共享运行时协议。只有理解这件事，才会明白卷三的核心并不只是“能启动多个 agent”，而是“多个 agent 终于开始围绕同一份工作状态协作”。

## 思想命题

### 命题一：共享 Task List 不是提醒板，而是团队协调协议

`getTaskListId()`、`setLeaderTeamName()` 和 `TeamCreateTool` 一起决定了所有 teammate 最终落到哪一份任务目录里。所谓“共享”，首先不是大家都能看见，而是 leader、in-process teammate 和进程外 teammate 都被路由到同一账本。没有这个前提，后面的分工、认领和阻塞都无从谈起。

### 命题二：多代理协作依赖最小一致语义，而不是复杂流程状态机

`tasks.ts` 把任务状态刻意压缩成 `pending / in_progress / completed` 三态，连旧的 `planning / implementing / reviewing / verifying` 都会被迁移为 `in_progress`。Claude Code 的选择很清晰：宁可让状态模型更少，也要保证所有代理都能稳定共享同一语义；更细的进行中信息由 `owner`、`activeForm`、activity 和 hooks 去补足。

### 命题三：收敛不是最后补一层 UI，而是任务协议的一部分

`TaskListTool` 会过滤掉已完成 blocker 和内部任务，`TaskListV2` 会按“刚完成、进行中、未被阻塞的待办、较早完成”排序，并附上 owner 颜色和活动描述。也就是说，Task List 的终点不是把数据存下来，而是把“团队现在最该看见什么”组织出来，帮助协作继续推进。

## 源码剖面

第一层是命名空间绑定。`TeamCreateTool.call()` 在 team 创建时，不只写 team file，也会调用 `resetTaskList()`、`ensureTasksDir()` 和 `setLeaderTeamName()`。这意味着团队一旦成立，共享任务账本就已经被初始化好了。随后，`getTaskListId()` 会按固定优先级解析当前要访问哪份任务表：显式环境变量优先，其次是 in-process teammate 上下文，再其次是进程外 teammate 的 team name、leader 记录下来的 team name，最后才回落到单会话的 session ID。这里体现的是一条很硬的工程原则：协作先统一账本，再统一行为。

第二层是任务模型本身。`tasks.ts` 把 `Task` 定义成一个很克制的数据结构：`id`、`subject`、`description`、`activeForm`、`owner`、`status`、`blocks`、`blockedBy`、`metadata`。其中最值得注意的是状态只有三种。Claude Code 并没有把每种工作阶段都编码成状态枚举，而是故意保持三态最小集，再让 `activeForm` 这种轻量字段承担“现在正在做什么”的细节表达。这个决策非常适合多代理协作，因为越简单的共享语义，越容易稳定地被多个执行主体同时理解和维护。

第三层是账本持久化与编号稳定性。`createTask()` 并不是随手写一个 JSON 文件。它会先获取 task-list 级别锁，在锁内读取当前最高 ID 与 high water mark，再分配新的单调递增编号，最后把任务落盘。`resetTaskList()` 和 `deleteTask()` 也会维护 high water mark，目的是在 reset 或删除后避免旧编号被复用。这个细节非常关键，因为在多代理系统里，`#17` 不是装饰性的序号，而是跨消息、跨回合、跨成员引用同一工作项的稳定句柄。

第四层是创建与认领的拆分。`TaskCreateTool.call()` 创建的新任务默认是 `pending`、`owner: undefined`、空依赖图。这说明 Claude Code 有意把“把工作项放入团队账本”与“谁来接手它”拆成两步。真正的接手逻辑由 `claimTask()` 和 `claimTaskWithBusyCheck()` 承担。它们会检查任务是否存在、是否已完成、是否已被他人认领、是否仍被未完成 blocker 卡住；启用 busy check 时，还会在同一个 list 锁内确认当前 agent 没有别的未完成任务。这里的重点不是锁本身，而是系统试图用原子性去替代“大家礼让一下”的软协商。

第五层是依赖图。`blockTask()` 会同时更新 `blocks` 和 `blockedBy` 两个方向，形成任务之间的显式图结构。随后，`claimTask()` 在判断任务能否被认领时，会把“所有未完成任务的 ID”计算成集合，并检查当前任务的 `blockedBy` 中是否仍有未完成项；`TaskListTool` 在对外返回任务时，也会把已经完成的 blocker 从 `blockedBy` 里滤掉，只保留当前仍真实存在的阻塞关系。这里说明 Claude Code 并不把依赖写进自然语言描述里，而是把它写成可被程序消费的约束图。

第六层是推进和收束。`TaskUpdateTool.call()` 负责真正的状态迁移。当 teammate 把任务标记为 `in_progress` 却没有显式提供 owner 时，工具会自动把当前 agent 名称写成 owner，这样共享列表和 UI 才能正确反映“谁在做”。当任务标记为 `completed` 时，系统会先跑完成 hooks；如果新增依赖，则通过 `addBlocks/addBlockedBy` 进入显式任务图。更重要的是，完成后工具还会主动 nudging：提醒调用 `TaskList` 看下一项，或者确认是否解除了他人的阻塞。可见，收束不是静默发生的，而是被编码成系统行为。

第七层是前台投影。`TaskListTool` 只把当前协作真正需要的信息暴露出来：忽略内部任务、过滤掉已解决 blocker，并输出 `#id [status] subject (owner)` 这样的紧凑视图。`TaskListV2` 则进一步做排序和认知组织：最近完成的任务优先展示，进行中任务其次，未被阻塞的 pending 再其次，较早完成项放在后面；owner 颜色、teammate activity、open blockers 也会一起显示。也就是说，Task List 的最后一公里不是“渲染一张表”，而是把团队最需要看到的局部现实摆到前台。

第八层是故障回收。`tasks.ts` 还定义了基于任务所有权推导 agent `idle/busy` 状态的逻辑，并在成员离开时把其未完成任务取消 owner、重置为 `pending`。这意味着共享 Task List 不只负责正常推进，还负责在团队成员退出、卡住或消失时把工作重新变回公共可接手状态。对于一个多代理系统来说，这其实就是最基础的恢复能力。

## 设计取舍

Claude Code 在任务系统上做了一个很鲜明的取舍：不用复杂流程状态机，而用极简三态加上 `owner`、阻塞关系、活动描述和 UI 排序来表达协作。这样做的代价是，很多细粒度阶段不再有独立枚举，像“规划中”“验证中”“审查中”这类语义，需要通过 `activeForm`、hooks、activity 文本和工具调用时机来补足。

但好处也正因此成立。状态越少，共享语义越稳；锁规则越简单，原子认领和阻塞判断越容易做对；不同运行形态下的 agent 越不容易在状态理解上漂移。对一个强调多代理协作的系统来说，这种“少而硬”的任务模型通常比“多而细”的流程模型更可用。Claude Code 把复杂性从状态枚举中拿走，转移到显式 owner、dependency graph 和前台收敛逻辑中，这是相当务实的工程选择。

## 工程启示

第一，想让多个 agent 真正协作，必须先把工作项落成共享事实，而不是只留在各自上下文里。

第二，任务系统最好先解决“同一份账本”的问题，再解决“任务长什么样”的问题。命名空间不一致时，所有状态设计都无意义。

第三，原子认领比礼貌协商更重要。只要有并发，就要假设两个 agent 会同时判断自己可以接同一项任务。

第四，依赖关系应当显式建模。把 blocker 写成结构化字段，比把“等我做完再做”埋在文本里可靠得多。

第五，前台 UI 不只是展示层。排序、过滤和活动提示会反过来塑造团队下一步的决策，因此它本身就是收敛协议的一部分。

## 思考题

1. 如果 leader 和 teammate 没有被 `getTaskListId()` 路由到同一份账本，会出现哪些“看似协作、实则各干各的”问题？
2. 为什么 `createTask()` 和 `claimTask()` 要被拆成两个阶段，而不是创建时就顺手写入 owner？
3. 三态任务模型牺牲了哪些表达力，又换来了哪些协作稳定性？
4. 如果不维护 `blocks`/`blockedBy` 的双向关系，任务阻塞判断会在哪些地方失真？
5. 一个共享 Task List 的 UI 应该只是展示现状，还是应该主动帮助团队决定下一步？

## 本章不覆盖什么

本章只讨论共享 Task List 如何承担分工、认领、推进和收束协议，不展开 Team Agent 的 mailbox 与权限桥，不回到单个子代理生命周期，也不深入 verification agent、远程 session 或更高层工作流编排。这些内容分别属于 CH07、CH08 和后续章节。
