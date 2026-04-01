# CH01 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Claude Code 值得被当作一套 Agent CLI 的系统设计样本来阅读，而不只是一个“会调工具的产品源码”。
- 读者收益：建立全书观察框架，知道后续 17 章为什么按运行时、约束、协作、上下文、扩展、工程化六条线展开。
- 本章排除内容：不进入具体实现细节，不展开某一条链路的完整代码级分析。
- 关键源码清单：`query.ts`、`screens/REPL.tsx`、`utils/swarm/inProcessRunner.ts`、`utils/tasks.ts`、`commands.ts`、`skills/loadSkillsDir.ts`、`context.ts`、`services/mcp/client.ts`。

## 源码锚点

- `query.ts`
  - 原因：定义主循环和回合推进。
  - 关注点：系统的核心不是工具集合，而是回合协议。
- `screens/REPL.tsx`
  - 原因：定义交互前台。
  - 关注点：前台承接输入、权限、任务、多代理与 query 启动。
- `utils/swarm/inProcessRunner.ts`
  - 原因：定义 in-process team agent 运行内核。
  - 关注点：多代理需要显式协作协议。
- `utils/tasks.ts`
  - 原因：定义共享任务台账。
  - 关注点：多代理协作通过任务协议显式化。
- `commands.ts`
  - 原因：汇总命令与扩展面。
  - 关注点：扩展机制最终要统一装配。
- `skills/loadSkillsDir.ts`
  - 原因：把技能 Markdown 转成运行时对象。
  - 关注点：技能本质上是可调度 prompt 程序。
- `context.ts`
  - 原因：装配系统/用户上下文。
  - 关注点：上下文是运行资源，而不是静态附录。
- `services/mcp/client.ts`
  - 原因：并入远程能力。
  - 关注点：MCP 是能力总线，而不只是工具桥。

## 主调用链

1. 用户在 `screens/REPL.tsx` 的前台提交输入。
2. 输入进入 `utils/handlePromptSubmit.ts`，被解析为命令、prompt 或队列事件。
3. 一轮真正执行进入 `query.ts` 的 `queryLoop()`。
4. `queryLoop()` 在模型调用前执行上下文治理和预算检查。
5. 工具调用进入 `services/tools/toolOrchestration.ts` 与 `services/tools/toolExecution.ts`。
6. 多代理或团队协作则进一步落到 `utils/swarm/inProcessRunner.ts` 与 `utils/tasks.ts`。
7. 扩展能力则由 `commands.ts`、`skills/loadSkillsDir.ts`、`services/mcp/client.ts` 汇入会话可见能力面。

## 关键 trade-off

- 约束：同一套系统要同时支持交互式 REPL、SDK、plan mode、多代理、MCP、上下文治理和恢复路径。
- 方案：接受更高的运行时复杂度，围绕回合协议、任务协议、权限协议和扩展协议建立稳定骨架。
- 代价：局部代码复杂、模块边界看起来不总是最纯净。
- 收益：系统能承受真实世界的长会话、恢复、扩展和协作压力。

## 事实校验

- 直接来自实现：主循环、前台、任务台账、技能加载、上下文装配和 MCP 并入都能在源码里直接定位。
- 属于推断：把这些实现提升为“Agent CLI 方法论”是作者抽象，不是源码显式声明。
- 需要避免夸大：不能声称 Claude Code 已经有单一完美架构，它更像一套在现实约束中演化中的系统。
