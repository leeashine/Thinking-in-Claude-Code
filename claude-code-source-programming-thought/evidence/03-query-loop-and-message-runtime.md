# CH03 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Claude Code 的运行时核心是一个回合驱动、消息驱动、可恢复的主循环。
- 读者收益：理解 query loop 怎样把模型调用、工具执行、上下文治理和恢复路径统一到一轮回合中。
- 本章排除内容：不展开 REPL 前台和 Team Agent 协作协议。
- 关键源码清单：`query.ts`、`services/tools/toolOrchestration.ts`、`services/tools/toolExecution.ts`、`query/config.ts`、`services/compact/compact.ts`、`services/compact/autoCompact.ts`。

## 源码锚点

- `query.ts`
  - 原因：回合执行主循环。
  - 关注点：State、continue site、tool follow-up、compact/recovery、turn transition。
- `services/tools/toolOrchestration.ts`
  - 原因：工具调度。
  - 关注点：只读并发、写操作串行、上下文修改回写。
- `services/tools/toolExecution.ts`
  - 原因：单个工具执行。
  - 关注点：权限、hook、进度、错误分类、结果映射。
- `query/config.ts`
  - 原因：回合级快照配置。
  - 关注点：避免运行时 gate 在一轮中途漂移。
- `services/compact/compact.ts`
  - 原因：核心 compact 策略。
  - 关注点：长回合的上下文治理。
- `services/compact/autoCompact.ts`
  - 原因：自动 compact 触发与跟踪。
  - 关注点：压缩不是补丁，而是主路径。

## 主调用链

1. `query()` 调用 `queryLoop()` 并维护命令生命周期收尾。
2. `queryLoop()` 初始化 `State`，记录 `messages`、`toolUseContext`、`transition` 等跨迭代状态。
3. 进入单轮前，依次执行 tool result budget、snip、microcompact、context collapse、autocompact。
4. 通过 `deps.callModel()` 发起模型流式调用，流式收集 assistant message 与 tool_use。
5. 若出现工具调用，交给 `runTools()`，由 `toolOrchestration.ts` 分批并发/串行执行。
6. 若出现 prompt-too-long、media error、max_output_tokens、stop hooks 或 token budget 等情况，则通过 `transition` 进入下一次迭代。
7. 只有当本轮不再需要 follow-up，主循环才终止这一回合。

## 关键 trade-off

- 约束：真实工作回合会遇到长上下文、工具风暴、模型回退和恢复需求。
- 方案：把 recovery 与 compact 直接放入 query 主路径，维护复杂但稳定的回合状态机。
- 代价：`query.ts` 体量大，理解门槛高。
- 收益：系统能处理长回合，而不是一次 API 出错就整体失败。

## 事实校验

- 直接来自实现：`State`、预调用治理、tool follow-up、`runTools()`、recovery 分支和 `transition` 字段。
- 属于推断：把这套主循环概括为“回合驱动、消息驱动的运行时”是抽象总结。
- 需要避免夸大：不能说所有恢复都完美统一，源码里仍保留了多种 feature gate 和历史兼容路径。
