# CH05 权限不是附属功能，而是运行时边界

状态：`completed`

## 一句话摘要

Claude Code 的权限系统不是一个“要不要弹窗”的附属模块，而是一套先做策略判定、再编排交互、再改写模式状态的运行边界协议。

## 问题引入

很多工程师第一次看 Agent 权限时，会把它想成一个确认框：执行危险动作前问一下用户，用户点允许就继续，点拒绝就结束。这样的理解在小系统里还能成立，但 Claude Code 的源码显示，真实的 Agent 运行时根本不是这么工作的。因为真正的问题并不是“要不要问”，而是“哪些动作可以直接放行，哪些动作必须中断，哪些动作在当前模式下应该被自动改写，哪些动作只能交给特定审批路径处理”。

只要工具、模式和协作一多，权限就不再是 UI 问题，而是系统边界问题。Claude Code 正是按这个方向设计的：策略层先做 allow / deny / ask 三态决策，交互层再把 ask 变成具体审批流，模式层则在 plan、auto、acceptEdits、bypassPermissions 等运行状态之间迁移权限上下文。用户看到的是一个对话框，源码里发生的却是一整套边界协议。

## 思想命题

### 命题一：权限决定的不是“能不能点继续”，而是工具是否有资格进入执行链

`hasPermissionsToUseTool()` 并不等到工具执行后才兜底拦截，而是在执行前就返回 `allow`、`deny` 或 `ask`。这意味着权限系统本身就参与了工具调用语义的建模。工具不是“先运行再补确认”，而是“先过边界，再进入运行时”。

### 命题二：交互界面只是权限协议的表现层

真正的权限编排发生在 `useCanUseTool.tsx`。它先接住策略层给出的决策，再按顺序处理自动放行、自动拒绝、coordinator 权限、swarm worker 权限、classifier 快路径和最终人工审批。`PermissionRequest.tsx` 只是把不同工具路由到合适的审批组件，并不负责决定规则本身。

### 命题三：模式切换比单次授权更重要

`permissionSetup.ts` 把 default、dontAsk、acceptEdits、auto、plan、bypassPermissions 视为不同运行边界，而不是 UI 标签。进入 plan mode 会保存 `prePlanMode`，auto mode 会剥离危险 allow 规则，退出 plan mode 时又要决定是否恢复这些规则。Claude Code 的权限真正是状态机，而不是单次弹窗。

## 源码剖面

第一层是策略判定。`permissions.ts` 中的 `createPermissionRequestMessage()` 会把 decision reason 翻译成面向人的解释，而 `hasPermissionsToUseTool()` 则负责把规则、模式、classifier、tool-specific `checkPermissions()` 和 headless/async 条件汇总成最终决策。这里最关键的一点是三态结果：allow、deny、ask。只要有了这三态，权限就不再只是“成功或失败”，而是能把“需要进一步审批”建模为系统内的合法状态。

第二层是交互编排。`useCanUseTool.tsx` 拿到权限结果后，不会直接弹一个统一对话框。allow 直接 resolve，deny 会记录日志与通知，ask 则继续走多条特殊路径：如果当前上下文要求等待自动检查，就先走 coordinator handler；如果是 swarm worker，就走 worker 专属桥接；如果是 Bash classifier 可抢先命中高置信规则，还会在真正弹窗前自动放行。只有这些路径都不能解决时，才会进入 `handleInteractivePermission()`。

第三层是表现层路由。`PermissionRequest.tsx` 通过 `permissionComponentForTool()` 给不同工具选择不同审批组件。普通文件与命令工具走通用流程，`EnterPlanModeTool` 和 `ExitPlanModeV2Tool` 则拥有专门的 permission request 组件。这意味着 Claude Code 并不相信一种通用弹窗可以表达所有边界；某些工具本身就带着特殊协议，审批界面必须和工具语义一起变化。

第四层是模式状态迁移。`permissionSetup.ts` 的 `transitionPermissionMode()`、`prepareContextForPlanMode()`、`stripDangerousPermissionsForAutoMode()` 和 `restoreDangerousPermissions()` 明确表明：权限不是静态配置，而是会随着运行阶段切换。进入 plan mode 时要记录原模式；如果计划阶段启用了 auto 语义，还要额外剥离危险规则，防止前置 allow 规则绕过 classifier；退出时又要根据 gate 与 `prePlanMode` 决定恢复到什么状态。

第五层是计划模式的特殊权限协议。`EnterPlanModeTool` 在 `call()` 里直接调用 `prepareContextForPlanMode()` 并把 mode 设为 `plan`。`ExitPlanModeV2Tool` 更复杂：普通会话需要 ask，teammate 则可能绕过本地 UI，转为 leader approval 或本地直接退出；退出时还会读写 plan file、恢复模式、处理 auto gate fallback，并把审批结果映射成新的 permission update。这里可以清楚看到，权限系统已经深入运行时协议，而不是停留在界面层。

## 设计取舍

Claude Code 在权限系统上选择的是“先策略化，再界面化”，而不是“所有风险动作一律弹窗”。代价是实现明显更复杂：策略规则、decision reason、自动审批、swarm 路径、mode transition、dangerous rule strip/restore 都要一起维护，阅读门槛也很高。

但如果不这样做，系统要么过度保守，导致一切动作都依赖人工确认；要么过度开放，只能把安全性寄托在工具作者自觉上。Claude Code 把复杂度前置到了权限运行时，换来的好处是：自动化和可控性可以同时存在，且能随着模式切换与协作场景变化动态调整。

## 工程启示

第一，权限系统至少要区分 allow、deny、ask 三态。否则你无法把“需要进一步审批”建成系统内的中间状态。

第二，策略层与 UI 层要分离。规则判定、自动化快路径和交互展示混在一起，最终只会得到既难测又难扩展的代码。

第三，运行模式应该显式建模。default、plan、auto、acceptEdits 这类模式不是按钮文案，而是会改写权限边界的状态。

第四，危险 allow 规则需要和模式联动。只要系统允许 auto/classifier 一类自动化路径，就必须考虑现有 allow 规则是否会直接绕过它们。

## 思考题

1. 你的 Agent 系统里，`ask` 是一等状态，还是 `allow/deny` 之外的临时补丁？
2. 哪些权限规则应该在模式切换时被剥离或恢复？
3. 为什么 `PermissionRequest` 适合做路由层，而不适合做策略层？
4. 如果权限系统只存在于 UI，而不进入执行主路径，会首先在哪些场景失效？

## 本章不覆盖什么

本章聚焦权限策略、交互编排和模式状态迁移，不展开单个工具内部的业务安全检查，也不完整展开 Plan Mode 全流程、Team Agent mailbox 协议和更细的 classifier 提示词实现。后续章节会继续拆开这些主题。
