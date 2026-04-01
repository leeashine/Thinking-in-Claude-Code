# CH16 观测、诊断与故障恢复

状态：`completed`

## 一句话摘要

Claude Code 把观测、诊断与恢复串成了一条工程控制面：埋点留下结构化信号，诊断定位编辑副作用，恢复逻辑把失败转换成状态迁移，会话存储则保证这些状态在中断与 `resume` 后仍可重建。

## 问题引入

一个只在“成功路径”上工作良好的 Agent 系统，通常很难真正进入生产环境。真实世界里的失败并不稀有：模型流式中断、413、媒体超限、工具写坏文件、IDE 诊断飙升、网络抖动、会话中途退出、重启后恢复失败。很多工程会把这些问题零散地分给日志系统、错误处理和持久化模块，结果是每处都做了一点，但整体上仍然没有形成可靠的恢复链。

Claude Code 在这方面体现出非常鲜明的工程成熟度。它不是“顺手加一些日志”，也不是“出了错就再试一次”，而是把观测、诊断和恢复写进主运行时。埋点不只是为了后端看报表，API logging 不只是为了调试，diagnostic tracking 不只是 IDE 贴心功能，session storage 也不只是 transcript 归档。它们共同回答的是同一个问题：当系统偏离理想路径时，如何把偏离本身变成可观察、可定位、可恢复、可继续的状态。

因此，CH16 的重点不是模块列表，而是 Claude Code 如何构造一条“失败之后仍然能工作”的控制面。

## 思想命题

本章的核心命题是：Claude Code 的工程化底座，不在于“记录了很多信息”，而在于它把观测、诊断和恢复组织成相互闭合的回路。

第一，观测是默认存在的。`services/analytics/index.ts` 并不是高层产品逻辑附带的外壳，而是被做成无依赖公共入口，允许系统在 sink 未挂载时先排队、后回放。也就是说，系统默认认为“先留下信号”比“等所有基础设施都准备好再说”更重要。

第二，诊断不是全量噪声，而是增量差异。Claude Code 并不把 IDE 当前所有报错一股脑喂给模型，而是记录编辑前基线，再只暴露新增 diagnostics。这样模型面对的是“这次改动造成了什么后果”，而不是无穷背景噪声。

第三，恢复不是异常分支，而是主循环状态机的一部分。`query.ts` 会对不同失败类型做不同迁移：上下文溢出、输出 token 不足、模型 fallback、孤儿消息清理，都是下一次继续执行的准备动作。

第四，持久化不是事后归档，而是恢复协议的底座。只要系统要支持 compact 后继续、sidechain transcript、退出后 resume 和内容替换回放，session storage 就必须保存的不只是消息文本，而是消息链、替换记录和元数据关系。

## 源码剖面

CH16 的第一个关键文件是 [services/analytics/index.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/analytics/index.ts)。这里可以看到 Claude Code 对埋点的判断非常工程化：埋点入口保持低依赖，`eventQueue` 在 sink 未就绪时临时缓存，`attachAnalyticsSink()` 再异步回放历史事件。同时，metadata 还要经过显式验证和 proto 字段清洗。这些细节说明系统不把 analytics 当成“想记就记的日志字符串”，而是把它看成既要可靠又要受数据边界约束的结构化管道。

[services/api/logging.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/logging.ts) 与 [services/api/claude.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/api/claude.ts) 则把 API 生命周期观测接入真正的模型调用。Claude Code 会在请求前记录 query 事件，失败时分类错误类型、识别网关并上报 tracing，成功后记录 token、耗时、缓存与时间戳。这里最重要的并不是记了哪些字段，而是 API 调用已经被当成主运行时可观察的状态转移，而不是一个黑箱函数。

恢复链的中心仍在 [query.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/query.ts)。从观测角度看，它会在关键节点写入 `queryTracking` 和若干事件；从恢复角度看，它会针对不同故障走不同路径。例如出现 413 时，不会直接失败退出，而会优先尝试 context collapse drain，再尝试 reactive compact；若 `max_output_tokens` 不足，则先提高预算或注入恢复信息继续；若触发模型 fallback，则会清理孤儿消息并切换模型。也就是说，失败在 Claude Code 里是状态机输入，而不是简单异常。

诊断链则体现在 [services/diagnosticTracking.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/services/diagnosticTracking.ts) 及其与文件工具、附件系统的协同。查询开始时 tracker 进入工作态；文件修改前，相关工具先记录 baseline；下一轮查询装配附件时，只把新增 diagnostics 提取出来再转成提示。这种增量式设计很关键，因为它把“编辑引发的新问题”独立出来了。模型因此可以围绕当前变更做修复，而不是被历史诊断噪声淹没。

最后是恢复能否跨进程持续的问题。这要看 [utils/sessionStorage.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/sessionStorage.ts) 与 [utils/conversationRecovery.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/conversationRecovery.ts)。Claude Code 存储的不只是 transcript 文本，还包括 parent-child 消息链、content replacement、collapse commit、snapshot 和 sidechain transcript。`loadTranscriptFile()` 与一致性检查逻辑再把这些内容恢复回当前会话。这说明 Claude Code 的 `resume` 不是“从磁盘读一点历史”，而是试图恢复可继续工作的会话结构。

因此，本章可以还原出一条完整链路：analytics 负责留下结构化信号，API logging 观测模型调用生命周期，`query.ts` 把失败转换成恢复迁移，diagnostic tracking 把编辑后果变成增量上下文，session storage 和 conversation recovery 负责在退出、压缩和 resume 后重建会话结构。

## 设计取舍

Claude Code 在这一章的第一项取舍，是观测成本与调试能力之间的平衡。更完整的埋点和结构化 logging 意味着实现复杂度上升，但它换来的是对真实故障路径的可见性。对于长会话 Agent，这种代价是值得的。

第二项取舍，是诊断精度与实现复杂度之间的平衡。全量报错最简单，增量 diagnostics 需要 baseline、文件映射和状态维护；但只有后者才能让模型聚焦当前变更后果。

第三项取舍，是快速失败与可恢复执行之间的平衡。Claude Code 在很多场景下不选择立即终止，而是尝试 context collapse、reactive compact、fallback 和预算调整。这样做会让主循环更复杂，却能显著提高会话连续性。

第四项取舍，是持久化体量与恢复能力之间的平衡。保存更多链路元数据会增加存储与实现负担，但没有这些结构，`resume` 最终只能做到“读历史”，做不到“恢复工作面”。

## 工程启示

如果你要做可长期运行的 Agent，CH16 至少给出四条直接可迁移的方法。

第一，把 analytics 做成低依赖公共入口，让关键信号可以先留住，再异步送出。

第二，不要把诊断做成全量噪声投喂。用 baseline 与增量比较，把“本次修改新增的问题”单独暴露给模型。

第三，让错误恢复进入主循环状态机，而不是散落在外围异常处理里。

第四，持久化时保存结构，而不只是文本。消息链、替换记录、压缩边界和子 agent transcript 都是恢复工作面的关键。

## 思考题

1. 如果没有增量 diagnostics，Claude Code 在文件修复回路里会面临什么噪声问题？
2. 为什么 `resume` 必须恢复消息链和替换记录，而不能只把历史消息简单重放？
3. 如果把 413、fallback 和 `max_output_tokens` 都统一处理成“直接失败”，主循环会失去哪些能力？

## 本章不覆盖什么

本章不展开后端观测平台细节，不逐项分析 tracing 字段，也不重述 compact 算法本身。这里关注的是 Claude Code 如何把观测、诊断与恢复编进同一条工程控制面。
