# CH06 Plan Mode：先探索、后实现的工程协议

状态：`completed`

## 一句话摘要

Plan Mode 的本质不是一条提示词或一个命令，而是一套把探索、计划文件、审批和恢复执行串成闭环的运行时协议。

## 问题引入

“先设计、后实现”这句话几乎所有 Agent 都会说，但大多数系统只是把它写进 prompt，然后指望模型自觉遵守。Claude Code 没有停在这个层面。它把这句话落实成了一个可执行协议：何时进入计划阶段，计划写到哪里，谁来批准退出，退出后恢复到什么权限状态，计划如何在 resume、fork 或 clear context 后继续存在。只有这些问题都被系统化回答之后，“先探索、后实现”才不再是一句道德劝告。

所以，Plan Mode 不能被理解成“让模型先多想一会儿”。从源码看，它至少包含四条链路：进入链、计划文件链、退出审批链和权限恢复链。进入时改写权限上下文，计划阶段围绕 plan file 展开，退出时要走专门审批组件，真正恢复执行时还要决定是否恢复 auto 语义与危险规则。Claude Code 设计的不是一种工作习惯，而是一套运行时协议。

## 思想命题

### 命题一：Plan Mode 不是命令，而是状态迁移协议

`EnterPlanModeTool` 的作用并不只是“宣布进入计划模式”。它会调用 `prepareContextForPlanMode()`，记录 `prePlanMode`，并在需要时处理 auto 语义和危险规则。进入 Plan Mode 的瞬间，系统边界已经被改写了。

### 命题二：计划文件不是副产品，而是协议载体

`getPlanFilePath()`、`getPlan()`、`copyPlanForResume()`、`copyPlanForFork()` 表明计划不是模型脑中的短期文本，而是一个可落盘、可恢复、可分叉的工程产物。没有 plan file，就很难让计划阶段具备可追踪性和可延续性。

### 命题三：退出 Plan Mode 才是协议真正闭环的地方

`ExitPlanModeV2Tool` 和 `ExitPlanModePermissionRequest` 共同处理 plan 写盘、审批、上下文保留或清空、模式恢复、teammate 差异路径和 auto gate fallback。所谓“批准开始实现”，本质上是一次新的边界提交。

## 源码剖面

第一层是进入阶段。`EnterPlanModeTool` 本身是只读且可并发安全的工具，但它的 `call()` 会直接把 `toolPermissionContext` 更新为 `plan`，并通过 `prepareContextForPlanMode()` 处理进入前的权限状态。其 `mapToolResultToToolResultBlockParam()` 还会向模型明确交代计划阶段应做什么，并限制“除 plan file 外不要写文件”。这里已经能看出一个重要特征：Plan Mode 既是权限模式，也是对模型工作流的显式约束。

第二层是计划文件载体。`plans.ts` 负责生成 plan slug、决定计划文件路径、读取计划内容，并在 resume/fork 时恢复或复制计划文件。`copyPlanForResume()` 甚至会在 plan file 缺失时尝试从 file snapshot 或历史消息中恢复内容。这说明计划并不是一次性消息，而是被当成持久化资产来对待。

第三层是退出工具。`ExitPlanModeV2Tool` 的复杂度远高于进入工具。它既要判断当前是否真的处于 plan mode，又要区分普通会话和 teammate：普通会话在 `checkPermissions()` 中返回 `ask`，需要本地审批；teammate 则可能绕过本地 UI，转成 leader approval 或在某些条件下本地直接退出。真正执行 `call()` 时，它还要读取或同步 plan file、处理 gate fallback、恢复 `prePlanMode`、决定是否恢复被剥离的危险规则，并最终返回后续执行所需的状态。

第四层是审批界面。`ExitPlanModePermissionRequest` 并不是一个简单确认框，而是一个协议提交器。它允许用户在退出时选择清空上下文、保留上下文、恢复 auto、退回 default 或 acceptEdits，并通过 `buildPermissionUpdates()` 生成新的 permission update。如果用户选择 clear context，系统会把 plan 内容转成新的执行起点；如果选择 keep-context，则保留当前消息流继续推进。这里的审批不是“同意不同意”，而是“用哪种边界进入实现阶段”。

第五层是模式联动。`planModeV2.ts` 提供 plan agent 数、explore agent 数、interview phase 和 plan structure variant 等运行参数，说明计划模式本身也在持续演化。`permissionSetup.ts` 中 `prepareContextForPlanMode()` 与 `transitionPlanAutoMode()` 则进一步证明：Plan Mode 不是隔离世界，它和 auto mode、classifier、dangerous rules 有显式耦合关系。

## 设计取舍

Claude Code 选择把计划阶段做成显式协议，而不是隐性约定。代价是系统实现明显更重：需要额外的工具、计划文件、审批组件、状态迁移函数和恢复逻辑，理解门槛也更高。

但收益同样明确。如果没有 plan file，计划无法跨 resume/fork 持续存在；如果没有显式进入/退出协议，计划阶段就只是一段脆弱 prompt；如果没有退出审批与模式恢复，系统很容易在“看似已经批准执行”时仍处于错误的权限状态。Claude Code 宁可增加运行时复杂度，也要换来计划阶段的可审计、可恢复与可分叉。

## 工程启示

第一，任何“先探索、后实现”的流程都应该有显式状态迁移，而不是只靠提示词。

第二，设计成果最好落成文件或等价持久化对象。否则计划阶段无法真正跨会话、跨恢复、跨协作存在。

第三，退出计划阶段不应该只是一个确认按钮，而应当是一次新的边界提交：保留多少上下文，恢复哪种模式，是否附带新的规则，都应该显式决定。

第四，计划协议和权限协议最好共用同一套状态模型。这样进入、审批、恢复才能保持一致。

## 思考题

1. 如果没有 plan file，计划阶段最先失去的能力是恢复、审计还是协作？
2. 为什么 `ExitPlanModeV2Tool` 的复杂度远高于 `EnterPlanModeTool`？
3. 你的系统里，计划阶段退出后恢复到什么状态，是显式建模还是隐式假设？
4. 哪些场景下应该保留上下文继续实现，哪些场景下更适合 clear context 后重新开始？

## 本章不覆盖什么

本章聚焦 Plan Mode 协议本身，不展开 Team Agent 的完整协作内核、不展开 memory/compact 主线，也不展开具体实现任务如何拆分为 task list。后续章节会继续处理这些问题。
