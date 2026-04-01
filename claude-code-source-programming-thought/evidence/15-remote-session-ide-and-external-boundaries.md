# CH15 evidence

状态：`completed`

## Brainstorm 冻结

- 核心命题：Claude Code 通过 remote session、permission bridge、IDE 发现与消息 adapter，把自身从本地 CLI 延伸成跨边界会话系统。
- 读者收益：理解为什么远程支持不是单次 RPC，而是会话、权限和上下文归属的重新分配。
- 本章排除内容：不展开远端服务实现，不细讲 IDE 协议，不把未在工作树中直接见到的模块写成既成事实。
- 关键源码清单：[remote/RemoteSessionManager.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/RemoteSessionManager.ts)、[remote/sdkMessageAdapter.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/sdkMessageAdapter.ts)、[remote/remotePermissionBridge.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/remotePermissionBridge.ts)、[utils/ide.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/ide.ts)、[setup.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/setup.ts)。

## 源码锚点

- [RemoteSessionManager.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/RemoteSessionManager.ts)：远程会话状态与消息订阅。
- [sdkMessageAdapter.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/sdkMessageAdapter.ts)：远端事件到本地消息模型的适配层。
- [remotePermissionBridge.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/remotePermissionBridge.ts)：把远端工具调用桥接回本地权限审批。
- [ide.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/ide.ts)：IDE 发现、工作区匹配与连接。
- [setup.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/setup.ts)：外部桥接能力的启动接点。

## 主调用链

1. [setup.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/setup.ts) 在系统启动阶段准备外部桥接能力。
2. [RemoteSessionManager.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/RemoteSessionManager.ts) 维护远程会话并处理用户事件写入与远端消息订阅。
3. [sdkMessageAdapter.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/sdkMessageAdapter.ts) 将远端事件转换为本地运行时可理解的消息。
4. [remotePermissionBridge.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/remote/remotePermissionBridge.ts) 把远端工具尝试回流到本地审批。
5. [ide.ts](/Users/lizixuan/Documents/IdeaProjects/cc-source/src/utils/ide.ts) 将 IDE 工作区和诊断通道接入 Claude Code。

## 关键 trade-off

Claude Code 选择承担跨边界消息适配和本地审批回流的复杂性，换来持续远程会话、IDE 诊断接入和边界可控。它没有为了“远程更自由”而轻易放弃本地权限主权。

## 事实校验

- “remote session 是持续会话”来自 `RemoteSessionManager.ts` 中写事件与订阅消息的双路径职责。
- “权限回流到本地”来自 `remotePermissionBridge.ts` 的 synthetic bridge 思路。
- “IDE 是运行时的一部分”来自 `ide.ts` 的发现与连接逻辑。
- “当前工作树未直接看到 UDS messaging 实现文件”来自实际文件检索，因此正文只写调用点与职责推断。
