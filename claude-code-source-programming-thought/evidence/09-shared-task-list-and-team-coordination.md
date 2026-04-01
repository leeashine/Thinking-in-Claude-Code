# CH09 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：多代理协作不是靠口头协商维持秩序，而是把分工、认领、依赖、推进和回收压到一份共享 Task List 上。
- 读者收益：看清为什么 team 创建时就要统一任务命名空间，为什么状态被压缩成三态，为什么 owner、blocker 和 UI 投影共同组成协作协议。
- 本章排除内容：不展开 Team Agent mailbox 协议，不展开子代理生命周期，不展开权限桥和完整 verification agent 机制。
- 关键源码清单：`utils/tasks.ts`、`tools/TaskCreateTool/TaskCreateTool.ts`、`tools/TaskUpdateTool/TaskUpdateTool.ts`、`tools/TaskListTool/TaskListTool.ts`、`components/TaskListV2.tsx`、`tools/TeamCreateTool/TeamCreateTool.ts`。

## 源码锚点

- `tasks.ts`
  - 原因：共享账本、任务模型、锁、认领与恢复逻辑的核心实现。
  - 关注点：`getTaskListId()`、三态 `TaskStatus`、high water mark、`createTask()`、`blockTask()`、`claimTaskWithBusyCheck()`、agent busy/idle 推导。
- `TeamCreateTool.ts`
  - 原因：团队建立时同步初始化 task list 命名空间。
  - 关注点：`resetTaskList()`、`ensureTasksDir()`、`setLeaderTeamName()`。
- `TaskCreateTool.ts`
  - 原因：工作项进入共享系统的入口。
  - 关注点：默认 `pending`、无 owner、空依赖图，创建与认领刻意拆分。
- `TaskUpdateTool.ts`
  - 原因：任务推进、依赖维护与收束提醒。
  - 关注点：`in_progress` 自动补 owner、completed hooks、`addBlocks/addBlockedBy`、完成后的下一步 nudge。
- `TaskListTool.ts`
  - 原因：模型与用户看到的任务投影。
  - 关注点：内部任务过滤、已解决 blocker 剔除、紧凑文本视图。
- `TaskListV2.tsx`
  - 原因：任务协议如何在前台转化为可执行视图。
  - 关注点：recent-completed / in-progress / pending 排序、owner 颜色、activity、open blockers。

## 主调用链

1. `TeamCreateTool.call()` 初始化团队级 task list，并通过 `setLeaderTeamName()` 把 leader 后续的任务访问绑定到团队名。
2. `getTaskListId()` 让 leader、in-process teammate 和进程外 teammate 都解析到同一份任务账本。
3. `TaskCreateTool.call()` 经 `createTask()` 在 list 锁内生成稳定 ID，创建默认 `pending` 的共享工作项。
4. `claimTask()` / `claimTaskWithBusyCheck()` 在认领时检查已占用、已完成、未解决 blocker 和 agent busy 状态。
5. `TaskUpdateTool.call()` 推进状态、自动补 owner、维护依赖图，并在完成时触发 hooks 与下一步提示。
6. `TaskListTool.call()` 过滤内部任务与已解决 blocker，输出当前仍相关的阻塞关系。
7. `TaskListV2` 把任务按优先级、owner、activity 和 blocker 投影成团队收敛视图；成员退出时未完成任务还会被重新变回公共待办。

## 关键 trade-off

- 约束：多代理共享任务时，状态模型越复杂，越容易在不同 agent 与不同运行形态之间产生语义漂移。
- 方案：采用 `pending / in_progress / completed` 三态，加上 owner、dependency graph、activeForm 和 UI/activity 补足细节。
- 代价：无法仅靠状态枚举表达更细粒度流程阶段。
- 收益：共享语义更稳，锁与认领规则更简单，阻塞判断和收束视图更容易保持一致。

## 事实校验

- 直接来自实现：team 级 task list 初始化、`getTaskListId()` 的多来源解析、三态任务模型、high water mark、防并发认领、双向 blocker 维护、`in_progress` 自动 owner、完成 hooks 与 `TaskList` 视图过滤、UI 排序与 owner/activity 展示。
- 属于推断：把共享 Task List 概括为“团队协调协议”与“收敛内核”是对这些机制共同作用的抽象总结。
- 需要避免夸大：
  - 不能把 Task List 写成纯 UI；它的核心在 `tasks.ts` 的锁、认领、阻塞和恢复逻辑。
  - 不能说任务阶段很细；源码明确收敛到三态，细节更多靠补充字段和视图表达。
  - 不能把 CH09 写成 mailbox 或权限桥章节；这里讨论的是共享账本与任务协议，而不是团队消息传输。
