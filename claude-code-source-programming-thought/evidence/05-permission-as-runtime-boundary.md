# CH05 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：权限系统是 Claude Code 的运行边界，不是附属弹窗。
- 读者收益：理解策略判定、交互编排与模式迁移如何共同构成权限协议。
- 本章排除内容：不展开单个工具的业务安全实现，不完整展开 Plan Mode 和 Team Agent 全流程。
- 关键源码清单：`hooks/useCanUseTool.tsx`、`utils/permissions/permissions.ts`、`utils/permissions/permissionSetup.ts`、`components/permissions/PermissionRequest.tsx`、`tools/EnterPlanModeTool/EnterPlanModeTool.ts`、`tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`。

## 源码锚点

- `utils/permissions/permissions.ts`
  - 原因：权限策略核心。
  - 关注点：`createPermissionRequestMessage()`、`hasPermissionsToUseTool()`、auto mode classifier、tool-specific `checkPermissions()`。
- `hooks/useCanUseTool.tsx`
  - 原因：策略到交互的编排层。
  - 关注点：allow/deny/ask 的分流，coordinator、swarm worker、interactive handler 的顺序。
- `components/permissions/PermissionRequest.tsx`
  - 原因：不同工具的审批路由。
  - 关注点：计划模式工具拥有专门审批组件，权限 UI 并不是单一模板。
- `utils/permissions/permissionSetup.ts`
  - 原因：权限模式状态迁移。
  - 关注点：dangerous rules 的剥离/恢复、plan/auto 模式联动。
- `tools/EnterPlanModeTool/EnterPlanModeTool.ts`
  - 原因：进入 plan mode 时直接改写权限上下文。
  - 关注点：`prepareContextForPlanMode()` 和 setMode 联合触发。
- `tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
  - 原因：退出 plan mode 的特殊权限协议。
  - 关注点：普通会话 ask、teammate 差异路径、plan file、恢复模式与 gate fallback。

## 主调用链

1. 工具调用进入 `useCanUseTool()`。
2. `useCanUseTool()` 调用 `hasPermissionsToUseTool()` 获取 `allow / deny / ask`。
3. 若是 `allow`，直接构造通过结果；若是 `deny`，记录决策并终止；若是 `ask`，继续走 coordinator、swarm worker、classifier 快路径与最终 interactive 路径。
4. 真正需要人工介入时，`PermissionRequest.tsx` 根据工具类型选择具体审批组件。
5. 对 `EnterPlanModeTool` 和 `ExitPlanModeV2Tool`，审批结果不只影响单次工具调用，还会通过 permission update 或直接 setAppState 改写会话权限模式。
6. 模式切换过程中，`permissionSetup.ts` 负责剥离或恢复危险规则，并维护 `prePlanMode` 等跨阶段状态。

## 关键 trade-off

- 约束：既要自动化推进，又不能让工具在复杂场景里失控。
- 方案：把权限拆成策略层、交互层、模式层，并允许 auto/plan/swarm 等路径重用同一套决策框架。
- 代价：实现复杂，阅读成本高，边界条件很多。
- 收益：权限不再依赖单个 UI，而能成为整个 Agent 运行时的稳定边界。

## 事实校验

- 直接来自实现：`hasPermissionsToUseTool()` 三态决策、`useCanUseTool()` 的 ask 分流、`PermissionRequest.tsx` 的组件路由、`permissionSetup.ts` 的模式迁移、Enter/Exit Plan Mode 对权限上下文的直接修改。
- 属于推断：将其概括为“策略层、交互层、模式层三层叠加”是对实现结构的抽象总结。
- 需要避免夸大：
  - 不能说权限系统只是 UI 弹窗。
  - 不能说 auto mode 完全等于 classifier，也不能说所有 ask 都会走人工审批。
  - 不能说 plan mode 只是提示词切换；源码里确实发生了权限上下文迁移。
