# 第四部分：任务系统

> 本部分深入分析Claude Code的任务系统设计，揭示异步任务管理的最佳实践。

---

## 概述

任务系统是Claude Code的异步执行引擎，管理着多种类型的后台任务。它采用了**策略模式**和**状态机模式**，提供了统一的生命周期管理和状态转换机制。

### 核心文件

| 文件 | 作用 |
|------|------|
| `Task.ts` | 基础类型定义 |
| `tasks.ts` | 任务注册表 |
| `utils/task/framework.ts` | 框架核心工具 |
| `tasks/` | 各任务实现目录 |

---

## 第15章：任务抽象

### 15.1 任务类型设计

系统定义了6种任务类型：

```typescript
type TaskType =
  | 'local_bash'      // 本地Shell命令
  | 'local_agent'     // 本地Agent
  | 'remote_agent'    // 远程Agent
  | 'in_process_teammate' // 进程内队友
  | 'local_workflow'  // 本地工作流
  | 'monitor_mcp'     // MCP监控
  | 'dream'           // Dream任务
```

**设计思想**：
- 每种类型对应不同的执行场景
- 类型前缀用于ID生成，便于识别
- 扩展新类型只需添加到枚举

### 15.2 任务状态机

```typescript
type TaskStatus =
  | 'pending'    // 等待执行
  | 'running'    // 正在运行
  | 'completed'  // 成功完成
  | 'failed'     // 失败
  | 'killed'     // 被终止
```

状态转换图：

```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

**核心特性**：
- 状态转换不可逆
- 三种终端状态：completed、failed、killed
- 终端状态判定函数：

```typescript
function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

### 15.3 任务基础状态

所有任务共享的基础字段：

```typescript
type TaskStateBase = {
  id: string              // 唯一标识
  type: TaskType          // 任务类型
  status: TaskStatus      // 当前状态
  description: string     // 描述信息
  toolUseId?: string      // 工具使用ID
  startTime: number       // 开始时间
  endTime?: number        // 结束时间（终端状态）
  totalPausedMs?: number  // 总暂停时间
  outputFile: string      // 输出文件路径
  outputOffset: number    // 输出偏移
  notified: boolean       // 是否已通知
}
```

### 15.4 任务接口抽象

```typescript
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

**策略模式体现**：
- 统一的`kill`接口
- 不同类型有不同的实现
- 通过`getTaskByType`动态获取实现

### 15.5 任务上下文

```typescript
type TaskContext = {
  abortController: AbortController  // 中止控制器
  getAppState: () => AppState       // 状态读取
  setAppState: SetAppState          // 状态更新
}
```

**设计意义**：
- `abortController`：控制任务生命周期
- `getAppState`：获取当前应用状态
- `setAppState`：不可变更新状态

### 15.6 任务ID生成

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}

function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id  // 例如: "b3k9x2m1"
}
```

**设计要点**：
- 前缀标识类型（便于调试）
- 8字节随机数（2.8万亿组合，防暴力攻击）
- 使用大小写不敏感字母表

---

## 第16章：本地任务

### 16.1 LocalShellTask：本地Shell执行

**核心职责**：执行本地Shell命令，管理进程生命周期。

**状态结构**：

```typescript
type LocalShellTaskState = TaskStateBase & {
  type: 'local_bash'
  command: string                    // 执行的命令
  result?: { 
    code: number        // 退出码
    interrupted: boolean // 是否中断
  }
  shellCommand: ShellCommand | null  // Shell进程对象
  isBackgrounded: boolean            // 前台/后台标志
  agentId?: AgentId                  // 关联Agent
  kind?: 'bash' | 'monitor'          // 显示类型
}
```

**关键特性**：

1. **前台/后台切换**

```typescript
function backgroundTask(taskId, getAppState, setAppState): boolean {
  const task = getTaskById(taskId)
  if (task?.isBackgrounded) return false
  
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(t => 
      t.id === taskId ? { ...t, isBackgrounded: true } : t
    )
  }))
  return true
}
```

2. **停滞监控机制**

检测交互式提示（如sudo密码提示）：

```typescript
// 监控输出停滞
const STALL_TIMEOUT_MS = 5000

if (outputStalled && looksLikeInteractivePrompt) {
  // 提示用户可能需要交互
}
```

3. **命令输出持久化**

```typescript
// 输出到磁盘文件
outputFile: getTaskOutputPath(id)
outputOffset: 0

// 实时追加
appendTaskOutput(taskId, content)
```

### 16.2 LocalAgentTask：本地Agent执行

**核心职责**：后台运行Agent，支持进度追踪和消息队列。

**状态结构**：

```typescript
type LocalAgentTaskState = TaskStateBase & {
  type: 'local_agent'
  agentId: string
  prompt: string
  abortController?: AbortController
  progress?: AgentProgress
  pendingMessages: string[]  // 排队消息
  retain: boolean           // UI持有标志
  isBackgrounded: boolean
  evictAfter?: number       // 驱逐截止时间
}
```

**关键特性**：

1. **AbortController层级**

```typescript
// 创建子控制器 - 父终止时自动终止
const abortController = parentAbortController
  ? createChildAbortController(parentAbortController)
  : createAbortController()
```

2. **消息队列机制**

```typescript
// 入队消息
pendingMessages: string[]

// 排空消息
function drainPendingMessages(taskId, getAppState, setAppState): string[] {
  const task = getTaskById(taskId)
  const messages = task.pendingMessages
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    pendingMessages: []
  }))
  return messages
}
```

3. **自动后台化**

```typescript
// 设置自动后台定时器
registerAgentForeground({
  autoBackgroundMs: number  // 超时自动后台
})
```

---

## 第17章：远程任务

### 17.1 RemoteAgentTask：远程Agent执行

**核心职责**：在远程Claude.ai服务器执行任务，轮询监控状态。

**状态结构**：

```typescript
type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: RemoteTaskType
  sessionId: string
  todoList: TodoList
  log: SDKMessage[]
  pollStartedAt: number      // 轮询开始时间
  reviewProgress?: {...}     // 代码审查进度
  ultraplanPhase?: UltraplanPhase
}
```

**关键特性**：

1. **轮询机制**

```typescript
const POLL_INTERVAL_MS = 1000

function startRemoteSessionPolling(
  taskId: string,
  context: TaskContext
): () => void {  // 返回清理函数
  const intervalId = setInterval(async () => {
    const status = await pollRemoteSession(sessionId)
    if (isComplete(status)) {
      clearInterval(intervalId)
      updateTaskState(taskId, context.setAppState, t => ({
        ...t,
        status: 'completed',
        endTime: Date.now()
      }))
    }
  }, POLL_INTERVAL_MS)
  
  return () => clearInterval(intervalId)
}
```

2. **元数据持久化**

支持会话恢复：

```typescript
// 保存状态到磁盘
saveTaskMetadata(taskId, {
  sessionId,
  remoteTaskType,
  todoList
})

// 恢复时加载
loadTaskMetadata(taskId)
```

### 17.2 InProcessTeammateTask：进程内队友

**核心职责**：同进程运行队友Agent，使用AsyncLocalStorage隔离。

**状态结构**：

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

**关键特性**：

1. **AsyncLocalStorage隔离**

```typescript
// 创建隔离的存储上下文
const store = new AsyncLocalStorage<TeammateContext>()

// 在隔离上下文中运行
store.run(context, async () => {
  await runTeammate(taskId, prompt)
})
```

2. **团队感知身份**

```typescript
type TeammateIdentity = {
  name: string       // 队友名称
  agentId: AgentId   // Agent标识
  teamName?: string  // 团队名称
}
```

3. **计划审批流程**

```typescript
awaitingPlanApproval: boolean

// 提交计划等待审批
function submitPlanForApproval(taskId, plan) {
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    awaitingPlanApproval: true
  }))
}

// 审批通过/拒绝
function approvePlan(taskId) {
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    awaitingPlanApproval: false
  }))
}
```

4. **空闲状态管理**

```typescript
isIdle: boolean
onIdleCallbacks?: Array<() => void>

// 注册空闲回调
function registerIdleCallback(taskId, callback) {
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    onIdleCallbacks: [...(t.onIdleCallbacks || []), callback]
  }))
}

// 触发空闲回调
function notifyIdle(taskId) {
  const task = getTaskById(taskId)
  task.onIdleCallbacks?.forEach(cb => cb())
}
```

---

## 第18章：并发任务管理

### 18.1 任务注册与更新

**注册任务**：

```typescript
function registerTask(task: TaskState, setAppState: SetAppState): void {
  setAppState(prev => ({
    ...prev,
    tasks: [...prev.tasks, task]
  }))
}
```

**更新任务状态**：

```typescript
function updateTaskState<T extends TaskState>(
  taskId: string,
  setAppState: SetAppState,
  updater: (task: T) => T
): void {
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.map(t => 
      t.id === taskId ? updater(t as T) : t
    )
  }))
}
```

**驱逐终端任务**：

```typescript
function evictTerminalTask(taskId: string, setAppState: SetAppState): void {
  setAppState(prev => ({
    ...prev,
    tasks: prev.tasks.filter(t => t.id !== taskId)
  }))
  // 清理磁盘输出
  evictTaskOutput(taskId)
}
```

### 18.2 AbortController层级管理

**创建子控制器**：

```typescript
function createChildAbortController(parent: AbortController): AbortController {
  const child = new AbortController()
  
  // 父终止时自动终止子
  parent.signal.addEventListener('abort', () => {
    child.abort()
  })
  
  return child
}
```

**设计意义**：
- 层级终止：父任务终止时所有子任务自动终止
- 避免孤儿任务：确保资源完全释放
- 简化清理逻辑：只需终止父控制器

### 18.3 清理注册表模式

```typescript
// 注册清理回调
const unregisterCleanup = registerCleanup(async () => {
  killTask(taskId, setAppState)
})

// 组件卸载时调用
useEffect(() => {
  return () => unregisterCleanup()
}, [])
```

### 18.4 任务输出管理

```typescript
// 初始化输出
async function initTaskOutput(taskId: string): Promise<void> {
  const path = getTaskOutputPath(taskId)
  await fs.mkdir(dirname(path), { recursive: true })
  await fs.writeFile(path, '')
}

// 追加输出
function appendTaskOutput(taskId: string, content: string): void {
  const path = getTaskOutputPath(taskId)
  fs.appendFile(path, content)
}

// 驱逐输出
async function evictTaskOutput(taskId: string): Promise<void> {
  const path = getTaskOutputPath(taskId)
  await fs.unlink(path).catch(() => {})
}
```

### 18.5 消息注入机制

```typescript
// 向队友注入用户消息
function injectUserMessageToTeammate(
  taskId: string,
  message: string,
  setAppState: SetAppState
): void {
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    pendingUserMessages: [...t.pendingUserMessages, message]
  }))
}
```

### 18.6 容错设计

**原子性通知标志**：

```typescript
notified: boolean  // 防止重复通知

function notifyTaskComplete(taskId) {
  const task = getTaskById(taskId)
  if (task.notified) return  // 防重复
  
  // 发送通知
  sendNotification(task)
  
  updateTaskState(taskId, setAppState, t => ({
    ...t,
    notified: true
  }))
}
```

**竞态条件防护**：

```typescript
// 使用时间戳检测文件变更
const lastWriteTime = getFileModificationTime(filePath)
const lastRead = readFileState.get(filePath)

if (lastWriteTime > lastRead.timestamp) {
  throw new Error('File unexpectedly modified')
}
```

---

## 设计模式总结

### 策略模式

每种任务类型实现统一的`Task`接口，通过类型分发获取具体实现。

### 状态机模式

五种明确状态，不可逆转换，终端状态判定函数。

### 不可变数据模式

所有状态更新使用展开运算符，保证状态安全。

### 观察者模式

`onIdleCallbacks`通知空闲状态，`enqueuePendingNotification`通知状态变更。

### 资源清理模式

注册/注销清理回调，AbortController层级管理，磁盘输出生命周期管理。

### 关注点分离

类型定义与实现分离，状态管理与UI逻辑分离，框架工具函数独立。

---

## 最佳实践

1. **使用AbortController管理生命周期**
2. **状态更新始终保持不可变性**
3. **前台/后台模式优化用户体验**
4. **输出持久化支持任务恢复**
5. **层级清理避免资源泄漏**
6. **原子标志防止重复操作**