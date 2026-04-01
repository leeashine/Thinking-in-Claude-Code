# CH06 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Plan Mode 是“探索-计划-审批-执行”闭环的运行时协议，不是单一命令。
- 读者收益：理解进入、计划文件、退出审批和模式恢复如何共同构成 Plan Mode。
- 本章排除内容：不展开 Team Agent 全貌，不展开 memory/compact 主线，不讨论具体实现任务拆分。
- 关键源码清单：`tools/EnterPlanModeTool/EnterPlanModeTool.ts`、`tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`、`components/permissions/EnterPlanModePermissionRequest/EnterPlanModePermissionRequest.tsx`、`components/permissions/ExitPlanModePermissionRequest/ExitPlanModePermissionRequest.tsx`、`utils/permissions/permissionSetup.ts`、`utils/plans.ts`、`utils/planModeV2.ts`。

## 源码锚点

- `EnterPlanModeTool.ts`
  - 原因：进入计划阶段。
  - 关注点：setMode 为 `plan`、`prepareContextForPlanMode()`、面向模型的计划阶段约束说明。
- `permissionSetup.ts`
  - 原因：计划模式与 auto/permissions 的状态迁移。
  - 关注点：`prepareContextForPlanMode()`、`transitionPlanAutoMode()`、危险规则剥离与恢复。
- `plans.ts`
  - 原因：计划文件持久化与恢复。
  - 关注点：`getPlanFilePath()`、`getPlan()`、`copyPlanForResume()`、`copyPlanForFork()`。
- `ExitPlanModeV2Tool.ts`
  - 原因：退出计划阶段的核心协议实现。
  - 关注点：普通会话 vs teammate、plan file 同步、模式恢复、gate fallback。
- `ExitPlanModePermissionRequest.tsx`
  - 原因：退出计划阶段的审批提交器。
  - 关注点：`buildPermissionUpdates()`、keep-context / clear-context / resume-auto 的差异。
- `planModeV2.ts`
  - 原因：计划模式运行参数。
  - 关注点：agent 数、interview phase、plan structure variant。

## 主调用链

1. 模型调用 `EnterPlanModeTool`。
2. `EnterPlanModeTool.call()` 把权限上下文推进到 `plan`，并通过 `prepareContextForPlanMode()` 记录 `prePlanMode` 与相关 auto 语义。
3. 计划阶段围绕 plan file 展开，`getPlanFilePath()` / `getPlan()` 负责读写载体。
4. 模型调用 `ExitPlanModeV2Tool` 时，普通会话先走 ask 审批，teammate 可能转成 leader approval 或特殊本地路径。
5. `ExitPlanModePermissionRequest` 根据用户选择生成 permission updates，决定 clear context、keep-context 或恢复 auto/default 等模式。
6. `ExitPlanModeV2Tool.call()` 同步 plan file、恢复 `prePlanMode`、处理 gate fallback 与危险规则，再把系统带回实现阶段。
7. 后续若会话 resume 或 fork，`copyPlanForResume()` / `copyPlanForFork()` 继续维护 plan 作为协议载体的连续性。

## 关键 trade-off

- 约束：如果计划阶段只是 prompt 约束，就无法审计、恢复或协作。
- 方案：把计划阶段做成显式协议，包含工具、文件、审批和状态迁移。
- 代价：实现更重，需要更多状态与恢复逻辑。
- 收益：计划不再是脆弱提示词，而是可追踪、可分叉、可恢复的工程资产。

## 事实校验

- 直接来自实现：进入/退出工具、plan file 读写与恢复、keep-context/clear-context 审批分支、teammate 差异路径、`prepareContextForPlanMode()` 和 `prePlanMode`。
- 属于推断：将其概括为“运行时协议”是对这些实现共同结构的抽象总结。
- 需要避免夸大：
  - 不能说 Plan Mode 完全禁止任何写操作；源码明确允许围绕 plan file 工作。
  - 不能说 Plan Mode 只是 UI 流程；进入和退出都会改写权限上下文。
  - 不能说退出审批只有一种路径；普通会话与 teammate 存在差异分支。
