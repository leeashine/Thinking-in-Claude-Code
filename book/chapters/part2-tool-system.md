# 第二部分：工具系统

> 工具系统是Claude Code的核心能力扩展机制，它将AI Agent从一个简单的对话系统转变为一个能够执行实际操作的强大助手。

---

## 第5章：工具抽象设计

### 5.1 为什么需要工具系统

AI模型本身只能生成文本，无法直接操作文件系统、执行命令或访问网络。工具系统通过提供一个标准化的接口，让模型能够"调用"各种能力：

```
用户请求 → 模型决策 → 工具调用 → 实际执行 → 结果返回
```

这种设计将**决策**与**执行**分离，模型负责判断"做什么"，工具负责"怎么做"。

### 5.2 Tool类型定义

工具的核心接口定义在`Tool.ts`中：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 标识
  name: string
  aliases?: string[]
  searchHint?: string
  
  // 核心方法
  call(args: z.infer<Input>, context: ToolUseContext, ...): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  
  // Schema定义
  readonly inputSchema: Input
  readonly outputSchema?: z.ZodType<unknown>
  
  // 行为判断
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  
  // 验证与权限
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
  
  // UI渲染
  renderToolUseMessage(input, options): React.ReactNode
  renderToolResultMessage?(content, ...): React.ReactNode
}
```

**设计要点**：
- **泛型约束**：`Input extends AnyObject`使用Zod Schema确保类型安全
- **方法链模式**：`validateInput → checkPermissions → call`形成验证链
- **渲染分离**：工具执行与UI渲染解耦

### 5.3 buildTool工厂函数

系统提供`buildTool`函数为常用方法提供安全默认值：

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,  // 默认不安全，保守策略
  isReadOnly: () => false,         // 默认写操作，保守策略
  isDestructive: () => false,
  checkPermissions: (input) => Promise.resolve({ behavior: 'allow' }),
  toAutoClassifierInput: () => '',
  userFacingName: () => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**设计哲学**：采用"**失败安全**"原则，默认值倾向于保守：
- `isConcurrencySafe`默认`false`：非安全工具独占执行
- `isReadOnly`默认`false`：写操作需要更严格的权限检查
- `checkPermissions`默认`allow`：但会被全局权限规则覆盖

### 5.4 lazySchema延迟加载

```typescript
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

延迟Schema构建有两个好处：
1. **避免循环依赖**：Schema可以引用尚未定义的类型
2. **减少启动开销**：复杂Schema只在首次使用时构建

### 5.5 ToolUseContext依赖注入

```typescript
export type ToolUseContext = {
  // 中断控制
  abortController: AbortController
  
  // 状态访问
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  
  // 工具配置
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    thinkingConfig: ThinkingConfig
  }
  
  // 文件状态
  readFileState: FileStateCache
  
  // UI回调
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif: Notification) => void
  
  // 会话状态
  messages: Message[]
}
```

这是一个典型的**依赖注入容器**，工具通过context获取所有需要的依赖，而不是直接导入。

---

## 第6章：文件工具

### 6.1 FileWriteTool设计

文件写入是最复杂的工具之一，因为它需要处理多种边缘情况：

```typescript
export const FileWriteTool = buildTool({
  name: 'Write',
  
  // 验证输入
  async validateInput({ file_path, content }, context) {
    // 1. 检查敏感信息
    const secretError = checkTeamMemSecrets(fullFilePath, content)
    if (secretError) return { result: false, message: secretError }
    
    // 2. 检查权限规则
    const denyRule = matchingRuleForInput(fullFilePath, permissionContext, 'edit', 'deny')
    if (denyRule) return { result: false, message: 'File is denied' }
    
    // 3. 检查文件是否已读取
    const readTimestamp = context.readFileState.get(fullFilePath)
    if (!readTimestamp) {
      return { result: false, message: 'Read file first' }
    }
    
    // 4. 检查文件是否被外部修改
    if (lastWriteTime > readTimestamp.timestamp) {
      return { result: false, message: 'File modified since read' }
    }
    
    return { result: true }
  },
  
  // 执行写入
  async call({ file_path, content }, context) {
    // 原子性写入操作...
  }
})
```

### 6.2 多阶段验证流程

```
validateInput阶段:
  │
  ├─► 敏感信息检查 → 失败则拒绝
  │
  ├─► 权限规则匹配 → 匹配deny规则则拒绝
  │
  ├─► 文件读取状态 → 未读取则拒绝
  │
  └─► 文件修改检测 → 被修改则拒绝

call阶段:
  │
  ├─► 技能目录发现（非阻塞）
  │
  ├─► LSP诊断追踪
  │
  ├─► 确保父目录存在
  │
  ├─► 文件历史备份
  │
  ├─► 原子性检查与写入
  │
  ├─► LSP服务器通知
  │
  └─► VSCode diff通知
```

### 6.3 生态系统集成

FileWriteTool不仅是写入文件，还集成了多个子系统：

```typescript
// 1. LSP集成
const lspManager = getLspServerManager()
if (lspManager) {
  clearDeliveredDiagnosticsForFile(`file://${fullFilePath}`)
  lspManager.changeFile(fullFilePath, content)
  lspManager.saveFile(fullFilePath)
}

// 2. VSCode集成
notifyVscodeFileUpdated(fullFilePath, oldContent, content)

// 3. 技能系统
const newSkillDirs = await discoverSkillDirsForPaths([fullFilePath], cwd)
if (newSkillDirs.length > 0) {
  addSkillDirectories(newSkillDirs).catch(() => {})
}
```

### 6.4 FileEditTool增量编辑

对于已有文件的修改，FileEditTool提供了更精确的编辑能力：

```typescript
const inputSchema = z.object({
  file_path: z.string(),
  old_string: z.string(),
  new_string: z.string(),
  replace_all: z.boolean().optional(),
})
```

**核心特性**：
- **精确匹配**：必须完全匹配要替换的内容
- **唯一性检查**：如果有多个匹配，拒绝执行
- **replace_all**：支持批量替换所有匹配项

---

## 第7章：交互工具

### 7.1 AskUserQuestionTool设计

当模型需要用户输入时，使用AskUserQuestionTool：

```typescript
const questionSchema = z.object({
  question: z.string(),
  header: z.string(),  // 显示标签，最多12字符
  options: z.array(questionOptionSchema()).min(2).max(4),
  multiSelect: z.boolean().default(false),
})

const questionOptionSchema = () => z.object({
  label: z.string(),
  description: z.string().optional(),
  preview: z.string().optional(),  // HTML预览
})
```

### 7.2 声明式问题定义

工具使用声明式风格定义问题：

```typescript
await askUserQuestion({
  question: '选择哪个测试框架？',
  header: '测试框架',
  options: [
    { label: 'Jest', description: 'Facebook出品，React首选' },
    { label: 'Vitest', description: 'Vite原生，速度更快' },
    { label: 'Mocha', description: '灵活配置，生态成熟' },
  ],
  multiSelect: false,
})
```

### 7.3 预览支持

对于需要视觉对比的场景，支持HTML预览：

```typescript
{
  label: '方案A',
  preview: `
    <div style="padding: 10px">
      <h3>方案A：集中式状态</h3>
      <pre>使用单一Store管理所有状态</pre>
    </div>
  `
}
```

### 7.4 权限模型

交互工具的权限设计：

```typescript
async checkPermissions(input) {
  return { 
    behavior: 'ask', 
    message: '需要回答问题',
    updatedInput: input 
  }
}
```

即使在高权限模式下，交互工具也需要用户确认，因为它需要用户的实际输入。

---

## 第8章：网络工具

### 8.1 WebFetchTool网页获取

WebFetchTool实现了智能的网页内容获取：

```typescript
export const WebFetchTool = buildTool({
  name: 'WebFetch',
  
  async checkPermissions({ url }, context) {
    const parsedUrl = new URL(url)
    
    // 预批准域名白名单
    if (isPreapprovedHost(parsedUrl.hostname)) {
      return { behavior: 'allow' }
    }
    
    return { behavior: 'ask', message: `访问 ${url}？` }
  },
  
  async call({ url, prompt }, context) {
    // 1. 获取网页内容
    const response = await fetch(url)
    const html = await response.text()
    
    // 2. 转换为Markdown
    const markdown = htmlToMarkdown(html)
    
    // 3. 提取相关信息
    const extracted = await extractWithPrompt(markdown, prompt)
    
    return { data: extracted }
  }
})
```

### 8.2 重定向处理

跨域重定向需要特殊处理：

```typescript
// 检测重定向
if (response.redirected && new URL(response.url).host !== originalHost) {
  return {
    data: {
      type: 'redirect',
      message: `重定向到不同域名: ${response.url}`,
      newUrl: response.url,
    }
  }
}
```

模型收到重定向信息后，可以选择是否跟随新的URL。

### 8.3 WebSearchTool网络搜索

WebSearchTool使用服务器端工具实现：

```typescript
function makeToolSchema(input: Input): BetaWebSearchTool20250305 {
  return {
    type: 'web_search_20250305',
    name: 'web_search',
    max_uses: 8,  // 最多8次搜索
  }
}
```

**流式进度报告**：
```typescript
onProgress({
  type: 'web_search_progress',
  message: '正在搜索...',
  results: partialResults,
})
```

### 8.4 MCPTool模板方法模式

MCPTool为MCP服务器提供的工具提供统一的模板：

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 实际名称在mcpClient.ts中覆盖
  
  // 占位实现
  async call() { return { data: '' } },
  async description() { return DESCRIPTION },
  async prompt() { return PROMPT },
})
```

当MCP客户端连接时，工具定义被动态替换为实际实现。

---

## 第9章：任务工具

### 9.1 TaskCreateTool任务创建

```typescript
export const TaskCreateTool = buildTool({
  name: 'TaskCreate',
  
  inputSchema: z.object({
    subject: z.string().describe('任务标题'),
    description: z.string().describe('详细描述'),
    activeForm: z.string().optional().describe('进行中状态文本'),
  }),
  
  async call({ subject, description, activeForm }, context) {
    const taskId = generateTaskId('local_task')
    
    // 创建任务状态
    const taskState = {
      id: taskId,
      status: 'pending',
      subject,
      description,
      startTime: Date.now(),
    }
    
    // 注册到AppState
    context.setAppState(prev => ({
      ...prev,
      tasks: { ...prev.tasks, [taskId]: taskState }
    }))
    
    return { data: { taskId } }
  }
})
```

### 9.2 TaskGetTool/TaskUpdateTool CRUD模式

任务工具遵循标准的CRUD模式：

```typescript
// Read
export const TaskGetTool = buildTool({
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
  shouldDefer: true,  // 延迟加载
  
  async call({ taskId }) {
    const task = await getTask(taskListId, taskId)
    return { data: { task } }
  }
})

// Update
export const TaskUpdateTool = buildTool({
  async call({ taskId, status, owner, ... }, context) {
    // 1. 状态转换验证
    if (!isValidTransition(oldStatus, newStatus)) {
      return { data: { error: 'Invalid transition' } }
    }
    
    // 2. 执行Hook
    if (status === 'completed') {
      await runTaskCompletedHooks(taskId)
    }
    
    // 3. 通知团队成员
    if (ownerChanged) {
      notifyTeamMembers(taskId, newOwner)
    }
    
    return { data: { success: true } }
  }
})
```

### 9.3 shouldDefer延迟加载

对于可能产生大量输出的工具：

```typescript
shouldDefer: true  // 大量工具时延迟加载节省prompt空间
```

这告诉系统：在发送给模型时，这个工具的定义可以延迟到实际需要时再加载。

### 9.4 任务工具与Agent系统集成

任务工具与Agent系统紧密协作：

```typescript
// 任务完成时通知Agent
async function runTaskCompletedHooks(taskId: string) {
  const task = getTask(taskId)
  
  // 如果是Agent任务，发送完成消息
  if (task.type === 'local_agent') {
    injectMessageToAgent(task.agentId, {
      type: 'task_completed',
      taskId,
    })
  }
}
```

---

## 设计模式总结

### 策略模式
每种工具实现统一的接口，系统通过类型分发获取具体实现。

### 工厂模式
`buildTool`函数填充默认值，生成完整的工具对象。

### 模板方法模式
工具骨架定义了验证→权限→执行的流程，具体工具填充实现。

### 观察者模式
进度回调（`onProgress`）实现实时状态通知。

### 命令模式
工具调用封装为`ToolUseBlock`，支持队列、撤销、重试。

---

## 最佳实践

1. **类型优先**：使用Zod Schema定义输入输出类型
2. **验证链**：`validateInput → checkPermissions → call`
3. **保守默认**：安全相关属性默认false
4. **依赖注入**：通过ToolUseContext获取依赖
5. **原子操作**：关键操作确保原子性
6. **进度报告**：长时间操作报告进度
7. **生态系统集成**：与其他系统（LSP、VSCode）协作