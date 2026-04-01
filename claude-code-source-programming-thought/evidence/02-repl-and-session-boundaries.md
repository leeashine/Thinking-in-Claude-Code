# CH02 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：REPL 是 Claude Code 的运行前台，而不是薄 UI。
- 读者收益：理解输入提交、命令路由、任务视图、teammate 视图和状态管理为什么必须在前台汇合。
- 本章排除内容：不展开 query 内部状态机，不展开权限系统内部判定。
- 关键源码清单：`screens/REPL.tsx`、`components/PromptInput/PromptInput.tsx`、`utils/handlePromptSubmit.ts`、`state/AppState.tsx`。

## 源码锚点

- `screens/REPL.tsx`
  - 原因：REPL 前台主文件。
  - 关注点：输入、query、任务、权限、代理视图都在这里汇合。
- `components/PromptInput/PromptInput.tsx`
  - 原因：输入前端实现。
  - 关注点：输入模式、slash command、teammate view、background tasks、suggestion 都接入其中。
- `utils/handlePromptSubmit.ts`
  - 原因：统一处理输入提交。
  - 关注点：立即命令、队列命令、普通 prompt、远程输入都在这里分流。
- `state/AppState.tsx`
  - 原因：稳定的应用状态 store。
  - 关注点：前台不是局部 UI state，而是会话级状态容器。

## 主调用链

1. 用户在 `PromptInput` 中输入文本。
2. `PromptInput` 触发 `screens/REPL.tsx` 中的 `onSubmit()`。
3. `onSubmit()` 判断即时命令、stash、历史、remote mode、queue、speculation 等前台边界。
4. 真正的提交进入 `utils/handlePromptSubmit.ts`。
5. `handlePromptSubmit()` 再调用 `processUserInput()` 或执行 local-jsx command / queued command。
6. 如果需要模型查询，`REPL.tsx` 的 `onQuery()` 再进入 `query.ts`。

## 关键 trade-off

- 约束：前台既要接收输入，又要承接权限、后台任务、多代理视图和会话恢复。
- 方案：让 `REPL.tsx` 和 `PromptInput.tsx` 承担较重的运行时前台职责。
- 代价：组件大、状态多、理解成本高。
- 收益：用户前台与后台协议保持一致，不会出现“UI 在做一套，运行时在做另一套”的撕裂。

## 事实校验

- 直接来自实现：`REPL.tsx` 内部大量状态、任务视图、permission queue、agent view 和 `handlePromptSubmit()` 链路。
- 属于推断：把 REPL 称作“运行前台”是抽象判断，但有充分实现依据。
- 需要避免夸大：不能说所有 UI 逻辑都集中在 REPL，本章只强调它在运行时边界上的中心性。
