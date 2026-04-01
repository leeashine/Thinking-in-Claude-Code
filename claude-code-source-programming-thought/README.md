# 《Claude Code源码编程思想》

副标题：从 REPL、Tool、Plan Mode 到 Team Agent 的系统设计方法

## 项目定位

这是一部面向架构型工程师的中文技术书，素材严格来自当前 `src/` 目录快照，目标不是做功能手册，也不是逐目录导游，而是借 Claude Code 的真实源码回答三个问题：

1. 一个现代 Agent CLI 到底由哪些抽象组成。
2. 这些抽象为什么要长成现在这个样子。
3. 哪些设计思想可以迁移到你自己的 Agent 系统中。

全书写法参考《JAVA编程思想》的组织节奏，但不复制其表达方式。每章统一采用“问题引入 -> 思想命题 -> 源码剖面 -> 设计取舍 -> 工程启示 -> 思考题”的结构。

## 当前范围

- 仅使用 `/Users/lizixuan/Documents/IdeaProjects/cc-source/src` 下源码。
- 不追溯 git 历史，不引用未核实的外部资料。
- 每章至少锚定 3 个源码文件，至少还原 1 条主调用链，至少讲清 1 个 trade-off。

## 目录总览

### 卷一 重新理解 Claude Code

1. `CH01` 为什么 Claude Code 值得用“编程思想”来阅读
2. `CH02` 一切从 REPL 开始：交互壳层与会话边界
3. `CH03` 一切皆回合：`query` 主循环与消息驱动运行时

### 卷二 约束塑造智能

4. `CH04` 一切皆能力：Tool 抽象、工具池与调度
5. `CH05` 权限不是附属功能，而是运行时边界
6. `CH06` Plan Mode：先探索、后实现的工程协议

### 卷三 从单代理到团队智能

7. `CH07` AgentTool：子代理如何被定义、启动与回收
8. `CH08` Team Agent：swarm、mailbox、permission bridge
9. `CH09` 共享 Task List：多代理如何分工、认领、推进与收敛

### 卷四 上下文就是系统设计

10. `CH10` System Prompt 的分层装配
11. `CH11` Memory 体系：用户、项目、团队三层记忆
12. `CH12` Compact、Token Budget 与上下文折叠

### 卷五 可扩展的 Claude Code

13. `CH13` 命令、技能、插件：统一扩展模型
14. `CH14` MCP：远程能力总线，而不只是工具接线
15. `CH15` 远程会话、IDE 与边界延伸

### 卷六 工程化与可复制方法

16. `CH16` 观测、诊断与故障恢复
17. `CH17` 面向约束的工程哲学：依赖环、权限、预算、策略
18. `CH18` 如何设计你自己的 Claude Code 型 Agent 系统

## 目录结构

```text
docs/claude-code-source-programming-thought/
├── PREFACE.md
├── FINAL-REVIEW.md
├── README.md
├── TASKS.md
├── STYLE.md
├── GLOSSARY.md
├── SOURCE-INDEX.md
├── templates/
│   ├── chapter-template.md
│   └── evidence-template.md
├── chapters/
└── evidence/
```

## 协作协议

每章都必须经过多 agent 协同，不允许单 agent 直接给出终稿。固定角色如下：

- `chief-editor`：负责收敛、定调、交叉链接、更新任务状态。
- `source-cartographer`：负责源码锚点、关键文件、模块边界。
- `runtime-tracer`：负责调用链、时序、状态迁移。
- `concept-writer`：负责把实现升格为思想命题和工程方法。
- `fact-checker`：负责逐条回指源码，清理空泛结论。

每章固定任务模板：

- `CHxx-A brainstorm`
- `CHxx-B evidence harvest`
- `CHxx-C call-chain tracing`
- `CHxx-D chapter draft`
- `CHxx-E fact check`
- `CHxx-F polish and cross-link`

## 阅读顺序

- 第一次阅读建议按卷顺序走，先建立运行时骨架，再进入扩展与工程化。
- 如果你只关心多代理协作，可以直接看 `CH07`、`CH08`、`CH09`。
- 如果你只关心上下文治理，可以先读 `CH10`、`CH11`、`CH12`。
- 如果你要复刻类似系统，优先读 `CH03`、`CH04`、`CH05`、`CH17`、`CH18`。

## 进度约定

- `completed`：正文、evidence、引用、边界说明均已完成。
- `in_progress`：已冻结命题并进入写作，但尚未完成校验。
- `pending`：尚未进入该章的正式写作。

当前进度以 `TASKS.md` 为准。当前总稿状态：`completed`。
