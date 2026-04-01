# CH16 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：观测、诊断与恢复不是三套孤立功能，而是一条工程控制面。
- 读者收益：理解为什么 analytics、API logging、diagnostic tracking、query 恢复状态机和 session storage 必须协同出现。
- 本章排除内容：不展开观测后端平台细节，不复述 compact 算法，也不写成 SDK 教程。
- 关键源码清单：[services/analytics/index.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/analytics/index.ts)、[services/diagnosticTracking.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/diagnosticTracking.ts)、[utils/sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts)、[services/api/logging.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/logging.ts)、[services/api/claude.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/claude.ts)、[query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)、[utils/conversationRecovery.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/conversationRecovery.ts)。

## 源码锚点

- [analytics/index.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/analytics/index.ts)：无依赖埋点入口与事件排队回放。
- [api/logging.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/logging.ts)：API 生命周期结构化观测。
- [claude.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/claude.ts)：把 API logging 接进真实模型调用。
- [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)：故障恢复状态机。
- [diagnosticTracking.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/diagnosticTracking.ts)：编辑前后增量 diagnostics。
- [sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts)：消息链、替换记录与 sidechain transcript 持久化。
- [conversationRecovery.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/conversationRecovery.ts)：`resume` 一致性恢复。

## 主调用链

1. 查询开始时，系统通过 analytics 与 query tracking 留下结构化信号。
2. [claude.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/claude.ts) 在请求前后调用 [api/logging.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/logging.ts) 记录 API 生命周期。
3. [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts) 对 413、fallback、输出不足等故障做恢复迁移。
4. 文件编辑前后，[diagnosticTracking.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/diagnosticTracking.ts) 只提取新增 diagnostics。
5. [sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts) 保存 transcript、replacement 与 sidechain 结构。
6. [conversationRecovery.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/conversationRecovery.ts) 在 `resume` 时重建会话结构并做一致性检查。

## 关键 trade-off

Claude Code 选择更重的结构化观测与恢复链，换取长期会话的可追踪、可续跑和可诊断。代价是实现更复杂；收益是失败不会立刻把系统打回“重新开始”的状态。

## 事实校验

- “analytics 是低依赖公共入口”来自 `analytics/index.ts` 的事件队列与 sink 绑定方式。
- “故障恢复是主循环的一部分”来自 `query.ts` 对 413、fallback、输出不足等路径的特殊处理。
- “诊断是增量差异而非全量噪声”来自 `diagnosticTracking.ts` 的 baseline 设计。
- “resume 恢复的是结构而不只是文本”来自 `sessionStorage.ts` 与 `conversationRecovery.ts`。
