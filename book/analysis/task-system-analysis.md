# 任务系统分析报告

## 一、系统概述

任务系统是一个统一的异步任务管理框架，用于处理多种类型的后台/并发任务。它采用了**策略模式**和**状态机模式**的设计思想，提供了统一的生命周期管理和状态转换机制。

### 核心文件位置

- `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/Task.ts` - 基础类型定义
- `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks.ts` - 任务注册表
- `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/utils/task/framework.ts` - 框架核心工具
- `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/` - 各任务实现目录

---

## 二、任务类型分析

### 2.1 LocalShellTask (本地Shell任务)

**文件**: `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/LocalShellTask/LocalShellTask.tsx`

**核心特性**:
- 执行本地Shell命令
- 支持前台/后台模式切换
- 提供"停滞监控"机制检测交互式提示
- 命令输出持久化到磁盘

**状态管理**:
```typescript
type LocalShellTaskState = TaskStateBase & {
  type: 'local_bash'
  command: string
  result?: { code: number; interrupted: boolean }
  shellCommand: ShellCommand | null
  isBackgrounded: boolean  // 前台/后台标志
  agentId?: AgentId        // 关联的Agent ID
  kind?: 'bash' | 'monitor'
}
```

### 2.2 LocalAgentTask (本地Agent任务)

**文件**: `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/LocalAgentTask/LocalAgentTask.tsx`

**核心特性**:
- 后台Agent执行，使用AbortController控制生命周期
- 支持进度追踪
- 消息队列机制支持任务间通信
- 支持父/子AbortController层级结构

**状态管理**:
```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  abortController?: AbortController
  progress?: AgentProgress
  pendingMessages: string[]      // 排队消息
  retain: boolean               // UI持有标志
  isBackgrounded: boolean
  evictAfter?: number          // 驱逐截止时间
}
```

### 2.3 RemoteAgentTask (远程Agent任务)

**文件**: `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

**核心特性**:
- 远程Claude.ai会话执行
- 轮询机制监控远程状态
- 支持多种远程任务类型
- 元数据持久化支持会话恢复

**状态管理**:
```typescript
type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: RemoteTaskType
  sessionId: string
  todoList: TodoList
  log: SDKMessage[]
  pollStartedAt: number        // 轮询开始时间
  reviewProgress?: {...}       // 代码审查进度
  ultraplanPhase?: UltraplanPhase
}
```

### 2.4 InProcessTeammateTask (进程内队友任务)

**文件**: `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`

**核心特性**:
- 同进程运行，使用AsyncLocalStorage隔离
- 团队感知身份
- 支持计划模式审批流程
- 空闲/活跃状态切换

**状态管理**:
```typescript
type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity
  permissionMode: PermissionMode
  awaitingPlanApproval: boolean
  isIdle: boolean
  shutdownRequested: boolean
  pendingUserMessages: string[]
  onIdleCallbacks?: Array<() => void>
}
```

### 2.5 DreamTask (Dream任务)

**文件**: `/Users/lizixuan/Documents/IdeaProjects/cc-source/src_glm/tasks/DreamTask/DreamTask.ts`

**核心特性**:
- 内存整合子代理
- 阶段追踪
- 文件变更追踪
- 锁回滚机制

**状态管理**:
```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  priorMtime: number   // 锁时间戳
}
```

---

## 三、生命周期管理

### 3.1 任务状态

```typescript
type TaskStatus =
  | 'pending'    // 等待执行
  | 'running'    // 正在运行
  | 'completed'  // 成功完成
  | 'failed'     // 失败
  | 'killed'     // 被终止
```

### 3.2 终端状态判定

```typescript
function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### 3.3 注册与更新

```typescript
// 注册任务到AppState
function registerTask(task: TaskState, setAppState: SetAppState): void

// 更新任务状态
function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T
): void

// 驱逐已完成的终端任务
function evictTerminalTask(taskId: string, setAppState: SetAppState): void
```

---

## 四、状态转换模式

### 4.1 状态转换图

```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

### 4.2 前台/后台切换

任务支持前台/后台模式切换，主要应用于Shell和Agent任务:

```typescript
// LocalShellTask
function backgroundTask(taskId, getAppState, setAppState): boolean
function backgroundAll(getAppState, setAppState): void

// LocalAgentTask  
function backgroundAgentTask(taskId, getAppState, setAppState): boolean
```

### 4.3 自动后台化

```typescript
// 设置自动后台定时器
registerAgentForeground({
  autoBackgroundMs: number  // 自动后台延迟
})
```

---

## 五、任务间通信机制

### 5.1 消息队列

```typescript
// 入队待处理通知
enqueuePendingNotification({
  value: string,
  mode: 'task-notification',
  priority: 'next' | 'later',
  agentId?: AgentId
})
```

### 5.2 进程内队友消息注入

```typescript
// 向队友注入用户消息
function injectUserMessageToTeammate(
  taskId: string,
  message: string,
  setAppState: SetAppState
): void

// 排空待处理消息
function drainPendingMessages(
  taskId: string,
  getAppState: () => AppState,
  setAppState: SetAppState
): string[]
```

### 5.3 空闲回调

```typescript
type InProcessTeammateTaskState = {
  onIdleCallbacks?: Array<() => void>  // 空闲时通知
}
```

---

## 六、并发任务处理模式

### 6.1 AbortController层级

```typescript
// 创建子AbortController - 父终止时自动终止
const abortController = parentAbortController
  ? createChildAbortController(parentAbortController)
  : createAbortController()
```

### 6.2 清理注册表

```typescript
// 注册清理回调
const unregisterCleanup = registerCleanup(async () => {
  killTask(taskId, setAppState)
})
```

### 6.3 轮询机制

```typescript
// 标准轮询间隔
const POLL_INTERVAL_MS = 1000

// 远程任务轮询
function startRemoteSessionPolling(
  taskId: string,
  context: TaskContext
): () => void  // 返回清理函数
```

### 6.4 任务输出管理

```typescript
// 初始化任务输出
initTaskOutput(taskId: string): Promise<void>

// 追加任务输出
appendTaskOutput(taskId: string, content: string): void

// 驱逐任务输出
evictTaskOutput(taskId: string): Promise<void>
```

---

## 七、设计模式和编程思想总结

### 7.1 策略模式

每种任务类型实现统一的`Task`接口:

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

通过`getTaskByType`动态获取任务实现:

```typescript
function getTaskByType(type: TaskType): Task | undefined {
  return getAllTasks().find(t => t.type === type)
}
```

### 7.2 状态机模式

- 五种明确状态: pending, running, completed, failed, killed
- 不可逆的状态转换
- 终端状态判定函数

### 7.3 不可变数据模式

所有状态更新使用展开运算符:

```typescript
updateTaskState(taskId, setAppState, task => ({
  ...task,
  status: 'completed',
  endTime: Date.now()
}))
```

### 7.4 资源清理模式

- 注册/注销清理回调
- AbortController层级管理
- 磬盘输出生命周期管理

### 7.5 观察者模式

- `onIdleCallbacks` 通知空闲状态
- `enqueuePendingNotification` 通知任务状态变更
- SDK事件队列

### 7.6 容错设计

- 原子性通知标志防止重复
- 竞态条件防护
- 超时机制

### 7.7 关注点分离

- 类型定义与实现分离
- 状态管理与UI逻辑分离
- 框架工具函数独立