# CH04 一切皆能力：Tool 抽象、工具池与调度

状态：`completed`

## 一句话摘要

Claude Code 的 Tool 不是函数包装器，而是带着权限、上下文、并发语义和结果回写协议的能力单元；工具池与调度层则把这些能力变成可控运行时。

## 问题引入

很多人理解 Agent 系统时，会把“有没有工具”当成第一问题：能不能读文件，能不能执行命令，能不能联网，能不能开子代理。Claude Code 的源码给出的顺序完全不同。它先问的不是“能力列表有多长”，而是“能力应该以什么形状进入运行时，才能被统一调度、统一授权、统一回写消息流”。如果这个问题没有先回答，工具越多，系统越不稳定。

因此，Claude Code 并没有把 Tool 设计成“模型调用的一段函数”。在 `Tool.ts` 里，Tool 同时暴露 `inputSchema`、`description`、`prompt`、`checkPermissions`、`isConcurrencySafe`、`isReadOnly`、`renderToolUseMessage`、`mapToolResultToToolResultBlockParam` 等能力接口。换句话说，Tool 从一开始就不是业务逻辑，而是一份运行协议。

## 思想命题

### 命题一：能力不是接口数量，而是被运行时托管的对象

`ToolUseContext` 收进了 AppState、消息数组、MCP 客户端、通知、compact 进度、文件历史、权限上下文和 in-progress tool 状态。一个 Tool 是否成立，不只看它能不能 `call()`，还要看它能否在这套上下文里安全运行。Claude Code 真正抽象的不是“函数”，而是“能力对象”。

### 命题二：工具池不是静态目录，而是被环境与策略裁剪的集合

`tools.ts` 里的 `getAllBaseTools()` 是内建能力全集，但并不等于模型实际能看到的工具集合。feature gate、环境变量、用户类型、REPL 模式、MCP 拼装和 deny rules 都会在装配阶段改变结果。也就是说，Claude Code 把“能力是否存在”和“能力是否可见”拆成了两个层次。

### 命题三：调度器的价值不是并发，而是把并发限制在可证明的边界内

`toolOrchestration.ts` 并不按“读工具”“写工具”做粗糙分类，而是要求每个 Tool 自己声明 `isConcurrencySafe()`。可并发批次先收集 `contextModifier`，批次结束后再顺序回放；非并发批次则串行推进并即时更新上下文。Claude Code 追求的不是极限吞吐，而是受控并发。

## 源码剖面

第一层是 Tool 合同本身。`Tool.ts` 中的 `Tool` 类型把能力定义为一组稳定接口：输入 schema 决定参数边界，`checkPermissions()` 决定工具特有的授权规则，`isConcurrencySafe()` 和 `isReadOnly()` 暴露执行语义，`prompt()` 与 `description()` 负责模型可见面，`renderToolUseMessage()` 与结果映射函数负责 UI 和 transcript。`buildTool()` 又给这些接口补上保守默认值，例如默认不并发、默认非只读、默认不声明破坏性。这是典型的 fail-closed 思路。

第二层是工具池装配。`getAllBaseTools()` 集中列出系统在当前构建里可能拥有的所有内建工具，再用 gate 和运行条件做裁剪。随后，工具还会经历 deny-rule 过滤、MCP 工具拼装、同名去重和排序。这里的关键结论是：Claude Code 不把工具注册当成框架边角，而是把它视为系统能力边界的第一道门。

第三层是调度。`runTools()` 先调用 `partitionToolCalls()`，根据 `isConcurrencySafe()` 把 tool_use 分成并发批和串行批。并发批用 `runToolsConcurrently()` 执行，但不会立刻改共享上下文，而是暂存每个工具的 `contextModifier`，待整批结束后按原顺序回放。串行批则每次执行后马上更新 `currentContext`。这说明 Claude Code 不是简单 `Promise.all`，而是显式处理“执行可并发，但上下文提交必须有序”。

第四层是执行流水线。`toolExecution.ts` 中的 `runToolUse()` 会先解析工具、校验输入、处理 fallback tool，再进入 `checkPermissionsAndCallTool()`。在这个函数里，工具调用要经过 PreToolUse hooks、权限判断、classifier 快路径、正式 `tool.call()`、telemetry、错误分类、结果块加工与消息回写。`tool.call()` 只是流水线中的一步，而不是全部。

第五层是结果回写。工具结果并不会直接变成一段字符串塞回对话，而是先被映射为 `ToolResultBlockParam`，再通过 `processPreMappedToolResultBlock()` 或 `processToolResultBlock()` 处理为可存储、可截断、可附带上下文修改器的结果块。Tool 的输出因此不是“返回值”，而是消息流中的一类结构化事件。

## 设计取舍

Claude Code 在 Tool 层选择的是“运行协议统一化”，不是“工具实现最短路径”。代价很明显：单个 Tool 的定义比传统函数式插件复杂得多，工具作者必须考虑 schema、权限、并发、安全、UI 和结果映射，调度层也必须处理上下文一致性。

但收益同样明显。如果没有这种统一协议，权限系统会散落在工具内部，工具池无法在 prompt 前过滤，调度器也很难知道哪些调用可以并发、哪些结果会影响后续上下文。随着工具数量增长，系统会迅速退化成一组彼此不兼容的特例。Claude Code 用更重的抽象，换来了更强的可治理性。

## 工程启示

第一，不要把“给 Agent 加能力”理解成继续堆函数。真正应该设计的是能力对象：输入如何约束，权限如何判断，是否可并发，结果如何写回系统。

第二，工具池应该是装配层，而不是 import 清单。只有把环境、策略、构建差异和外部能力统一纳入装配阶段，模型才能看到正确而稳定的能力边界。

第三，并发控制最好由能力自身声明，而不是由调度器硬编码白名单。因为安全性来自能力语义，不来自工具类别名。

第四，工具执行必须被放进消息流水线。只要结果会影响 transcript、compact、resume 或后续上下文，它就已经不是简单返回值了。

## 思考题

1. 如果 Tool 只有 `call()` 而没有 `checkPermissions()`、`isConcurrencySafe()` 和结果映射，系统复杂度会转移到哪里？
2. 为什么并发批不能一边执行一边直接改共享上下文？
3. 你的 Agent 系统里，模型能看到的能力集合，是否与“系统理论上注册过的能力集合”严格区分？
4. 哪些能力应该默认 fail-closed，哪些能力可以默认开放？

## 本章不覆盖什么

本章只讨论 Tool 抽象、工具池装配和调度执行主线，不展开权限规则匹配细节、Plan Mode 协议、Team Agent 协作、memory 策略和 MCP 远程能力总线。后续章节会分别拆开这些主题。
