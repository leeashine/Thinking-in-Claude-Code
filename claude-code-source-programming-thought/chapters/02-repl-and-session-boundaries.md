# CH02 一切从 REPL 开始：交互壳层与会话边界

状态：`completed`

## 一句话摘要

REPL 在 Claude Code 中不是一层薄 UI，而是承接输入、状态、权限、任务视图和多代理协作的前台运行环境。

## 问题引入

终端应用很容易被误判为“薄壳”。界面看起来很轻，输入一行文字，等模型回复，再把结果打印出来，似乎所有真正重要的东西都发生在后端。但 Claude Code 的 `REPL.tsx` 明显不是这种结构。它的体量和耦合度本身就在提醒读者：这里不是一个装饰层，而是系统运行时的前台。

要理解这一点，最有效的方法不是数组件，而是顺着一次提交往里看。用户在 `PromptInput` 中输入文本，`onSubmit` 先判断是否是即时 slash command、是否应该进入远程模式、是否需要恢复 stash、是否要进入队列、是否要触发 speculation accept，最后才把输入交给 `handlePromptSubmit()`。换句话说，“提交输入”在 REPL 里不是一个单一动作，而是一段要穿越状态、权限、模式、命令队列和会话边界的编排过程。

## 思想命题

### 命题一：REPL 是运行前台，不是聊天外壳

在 `screens/REPL.tsx` 里，REPL 负责的不只是消息列表和输入框。它持有 `teamContext`、`tasksV2`、`toolUseConfirmQueue`、`workerSandboxPermissions`、`viewedAgentTask`、`isLoading`、`userInputOnProcessing` 等一整套运行时前台状态。这意味着它是会话的可视化前台，而不是模型调用完成之后才来“渲染结果”的薄层。

### 命题二：输入通道同时承担“命令路由”和“会话调度”

`components/PromptInput/PromptInput.tsx` 管的不只是文本，它还负责 slash command、历史搜索、队列命令、teammate 视图切换、任务视图、权限模式切换、buddy/fast/thinking 等辅助入口。真正的输入提交则在 `utils/handlePromptSubmit.ts` 中继续分流：立即命令、普通 prompt、队列执行、远程消息、meta message 都可能走不同路径。

### 命题三：REPL 是多个运行子系统的汇合处

`state/AppState.tsx` 用稳定 store 暴露会话状态，`REPL.tsx` 再把 store、消息流、任务系统、权限对话框、teammate 视图和 query 调度编到一个地方。这里不是典型意义上的“分层得非常整齐”的 UI，而是一个刻意承担汇合职责的运行前台。

## 源码剖面

`components/PromptInput/PromptInput.tsx` 显示了输入通道的复杂度。它依赖 `useCommandQueue()`、`useMainLoopModel()`、`useAppState()`、`useArrowKeyHistory()`、`usePromptSuggestion()`、`useInputBuffer()` 等一系列 hook，同时还要处理不同模式的输入、粘贴内容、teammate 视图、后台任务对话框和各种提示/快捷键状态。这样的输入组件，已经不是“收文本”的职责边界，而是“用户如何进入系统”的第一层协议。

`screens/REPL.tsx` 中的 `onSubmit()` 则把这个协议推到完整形态。它先处理立即命令和 slash command，再处理 idle return、history、stash 恢复、remote mode、speculation accept，然后再调用 `handlePromptSubmit()`。同一个 `onSubmit()` 还负责把输入加入历史、清空输入框、恢复 stash、同步 `submitCount` 和 `userInputOnProcessing`，甚至在必要时直接触发 `onQuery()`。这说明 REPL 不是模型调用的外侧，而是模型调用的发车站。

`utils/handlePromptSubmit.ts` 是第二层证据。这个函数统一处理直接输入和队列命令两条路径，会把 prompt 展开成实际消息、处理 `/exit`、解析引用、执行即时 local-jsx 命令、必要时再进入 `executeUserInput()` 与 `processUserInput()`。于是，“输入”在 Claude Code 里并不等于一段字符串，而是待解析、待分流、待注入上下文的一段会话事件。

第三层证据来自状态管理。`state/AppState.tsx` 把 store 设计成稳定上下文提供者，`useAppState()` 允许组件只订阅状态切片，`useSetAppState()` 给非 UI 流程开放更新入口。`REPL.tsx` 因此能把任务列表、多代理上下文、tool permission、会话恢复和各种提示状态统一挂在一个前台里，而不必为每次交互重建一套临时上下文。

最后是任务与多代理视图。`REPL.tsx` 不只渲染消息，还内建 `TaskListV2`、`viewedAgentTask`、`viewedTeammateTask`、worker sandbox permission queue 和 teammate message 注入逻辑。也就是说，REPL 的职责边界天然覆盖“看自己”和“看代理”，这与传统聊天界面完全不同。

## 设计取舍

把这么多职责放进 `REPL.tsx` 和 `PromptInput.tsx`，代价非常明显：组件会大，局部认知负担会高，很多行为只能在整条链路里理解。这不是最容易维护的前端风格。但收益同样明显：用户与系统之间的所有重要状态都在前台被串成一根线，输入、权限、命令、代理、任务、恢复不会因为层层拆分而失去一致性。

换句话说，Claude Code 在这里做的取舍是：宁愿让前台看起来“重”，也不把运行时边界拆散。对于一个 Agent CLI，这个取舍通常是合理的，因为真正难的从来不是渲染文本，而是保证前台行为和后台协议始终同构。

## 工程启示

第一，终端 UI 也应该被当成运行时的一部分来设计。只要系统涉及命令、权限、会话恢复、后台任务或多代理视图，前台就不可能只是薄壳。

第二，输入通道必须承担路由职责。立即命令、队列命令、远程输入、普通 prompt 如果没有统一分流入口，系统会很快在边界条件上失控。

第三，稳定 store 比“把状态散在组件里”更适合 Agent CLI。因为这类系统的状态不是页面状态，而是会话状态，天然要被非 UI 流程读写。

第四，任务视图和代理视图应该和主输入前台共址。否则用户很难理解自己到底是在和主线程交互，还是在和某个 teammate 或后台任务交互。

## 思考题

1. 如果把 `REPL.tsx` 只保留消息渲染，其余逻辑全部下沉，会发生什么？
2. 为什么 `handlePromptSubmit()` 要同时处理 slash command、引用展开、即时命令和普通 prompt？
3. 你的终端产品里，输入通道只是采集文本，还是已经承担会话路由？
4. 当多代理加入后，前台为什么必须知道 `viewedAgentTask` 和 `teamContext`？

## 本章不覆盖什么

本章重点解释 REPL 作为运行前台的地位，不展开 `query` 的内部状态机，不细讲 tool execution、memory 注入、compact 策略，也不展开 Team Agent 的通信协议。这些内容会在后续章节继续拆开。
