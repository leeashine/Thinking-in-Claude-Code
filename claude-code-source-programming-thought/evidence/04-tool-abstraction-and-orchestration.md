# CH04 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Tool 是被运行时托管的能力协议，不是函数包装器。
- 读者收益：理解 Claude Code 如何把能力对象、工具池装配、并发调度与结果回写收束成一条主线。
- 本章排除内容：不展开权限规则细节、Plan Mode、Team Agent 和 MCP 传输层。
- 关键源码清单：`Tool.ts`、`tools.ts`、`services/tools/toolOrchestration.ts`、`services/tools/toolExecution.ts`。

## 源码锚点

- `Tool.ts`
  - 原因：定义 Tool 合同、`ToolUseContext`、名称匹配和 `buildTool()` 默认值。
  - 关注点：Tool 不是单一 `call()`；默认策略是保守的 fail-closed。
- `tools.ts`
  - 原因：内建工具全集、过滤与装配。
  - 关注点：`getAllBaseTools()`、deny rule 过滤、MCP 与 built-in 的统一工具池。
- `services/tools/toolOrchestration.ts`
  - 原因：tool_use 分批与执行顺序控制。
  - 关注点：`partitionToolCalls()`、并发批的上下文延迟提交、串行批的即时上下文更新。
- `services/tools/toolExecution.ts`
  - 原因：单个 Tool 的完整执行管线。
  - 关注点：输入校验、权限、hooks、`tool.call()`、结果块处理、消息回写。

## 主调用链

1. 模型在 assistant message 中产出 `tool_use`。
2. `query` 主循环把这些调用交给 `runTools()`。
3. `runTools()` 通过 `partitionToolCalls()` 按 `isConcurrencySafe()` 划分并发批与串行批。
4. 每个具体调用进入 `runToolUse()`，解析工具、校验输入、准备执行状态。
5. `checkPermissionsAndCallTool()` 负责 hooks、权限判断、classifier 快路径和正式 `tool.call()`。
6. 执行结果被映射为 `ToolResultBlockParam`，再经 `processPreMappedToolResultBlock()` 或 `processToolResultBlock()` 处理。
7. 结果消息与 `contextModifier` 回到主消息流，供后续 tool follow-up 或下一轮模型调用使用。

## 关键 trade-off

- 约束：工具越多，权限、并发、上下文一致性和结果存储越容易失控。
- 方案：把 Tool 设计为统一能力协议，再把装配、过滤、调度和执行纳入一套固定流水线。
- 代价：Tool 定义更重，工具作者必须实现更多运行时语义。
- 收益：系统能在能力增长时保持可治理，而不是积累大量特殊分支。

## 事实校验

- 直接来自实现：`Tool` 类型、`ToolUseContext`、`buildTool()` 默认值、`getAllBaseTools()`、`partitionToolCalls()`、`runToolUse()` 与结果块处理路径。
- 属于推断：将这套设计概括为“能力协议”和“受控并发”是对实现结构的抽象总结。
- 需要避免夸大：
  - 不能说 Tool 只是函数包装。
  - 不能说所有只读工具都会并发。
  - 不能说模型看到的工具集合等于所有已注册工具。
  - 不能说工具执行等于直接调用 `tool.call()`。
