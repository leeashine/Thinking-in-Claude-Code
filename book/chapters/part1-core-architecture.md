# 第一部分：核心架构

> 核心架构是Claude Code的骨架，定义了系统的基本结构和交互模式。

---

## 第1章：入口与启动流程

### 1.1 main.tsx的设计哲学

main.tsx是整个应用的入口点，长达4683行。它的设计遵循**启动时序优化**原则：

```typescript
// 这些副作用必须在所有其他导入之前运行
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');  // 性能分析标记

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();  // 并行启动MDM子进程

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();  // 并行预取Keychain数据
```

**设计要点**：
- 性能分析从入口开始记录
- MDM配置读取和Keychain认证并行执行
- 减少启动延迟约135ms

### 1.2 CLI命令注册模式

使用Commander.js构建层次化命令结构：

```typescript
const program = new CommanderCommand();

// 根命令
program
  .name('claude')
  .description('Claude Code - AI programming assistant')
  .version(VERSION);

// 子命令
program.command('update')
  .alias('upgrade')
  .description('Update Claude Code to latest version')
  .action(async () => {
    const { update } = await import('src/cli/update.js');
    await update();
  });

// 嵌套命令
const pluginCmd = program.command('plugin');
pluginCmd.command('install <plugin>').action(installPlugin);
pluginCmd.command('uninstall <plugin>').action(uninstallPlugin);
pluginCmd.command('list').action(listPlugins);
```

**懒加载设计**：使用动态`import()`在action回调中加载实现，避免启动时加载所有命令处理器。

### 1.3 条件编译

通过环境变量和feature flag控制功能：

```typescript
// 内部版本特性
if (process.env.USER_TYPE === 'ant') {
  program.addCommand(configCommand);
  program.addCommand(tungstenCommand);
}

// 实验性特性
if (feature('COORDINATOR_MODE')) {
  program.addCommand(coordinatorCommand);
}

if (feature('KAIROS')) {
  program.addCommand(assistantCommand);
}
```

### 1.4 迁移系统

采用版本化迁移策略，确保用户设置平滑升级：

```typescript
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  const config = getGlobalConfig();
  
  if (config.migrationVersion !== CURRENT_MIGRATION_VERSION) {
    // 依次执行迁移
    if (!config.migrationVersion || config.migrationVersion < 2) {
      migrateAutoUpdatesToSettings();
    }
    if (!config.migrationVersion || config.migrationVersion < 5) {
      migrateBypassPermissionsAcceptedToSettings();
    }
    // ... 更多迁移
    
    saveGlobalConfig(prev => ({
      ...prev,
      migrationVersion: CURRENT_MIGRATION_VERSION
    }));
  }
}
```

### 1.5 预取优化

将非关键操作延迟到REPL渲染后：

```typescript
export function startDeferredPrefetches(): void {
  // 这些操作不阻塞首屏渲染
  queueMicrotask(() => {
    void initUser();
    void getUserContext();
    void prefetchSystemContextIfSafe();
    void getRelevantTips();
    void prefetchOfficialMcpUrls();
  });
}
```

---

## 第2章：核心抽象

### 2.1 Tool类型系统

工具是Agent能力扩展的核心抽象：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  name: string
  aliases?: string[]
  
  // Schema定义
  readonly inputSchema: Input
  readonly outputSchema?: z.ZodType<unknown>
  
  // 核心方法
  call(args, context, ...): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>
  
  // 行为判断
  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive?(input): boolean
  
  // 验证与权限
  validateInput?(input, context): Promise<ValidationResult>
  checkPermissions(input, context): Promise<PermissionResult>
}
```

**类型安全设计**：
- `Input extends AnyObject`：使用Zod Schema约束
- 泛型参数`Output`：确保返回类型正确
- `ToolProgressData`：进度回调类型

### 2.2 Task任务抽象

任务是异步操作的抽象：

```typescript
export type TaskType =
  | 'local_bash'      // 本地Shell命令
  | 'local_agent'     // 本地Agent
  | 'remote_agent'    // 远程Agent
  | 'in_process_teammate' // 进程内队友
  | 'dream'           // Dream任务

export type TaskStatus =
  | 'pending'    // 等待执行
  | 'running'    // 正在运行
  | 'completed'  // 成功完成
  | 'failed'     // 失败
  | 'killed'     // 被终止

export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

**状态机设计**：
```
pending → running → completed
                 ↘ failed
                 ↘ killed
```

### 2.3 任务ID生成

安全设计确保不可预测性：

```typescript
const TASK_ID_PREFIXES = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  dream: 'd',
}

const TASK_ID_ALPHABET = '0123456789abcdefghijklmnopqrstuvwxyz'

export function generateTaskId(type: TaskType): string {
  const prefix = TASK_ID_PREFIXES[type]
  const bytes = randomBytes(8)  // 加密安全随机数
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % 36]
  }
  return id  // 如: 'b3k9x2m1'
}
```

- 36^8 ≈ 2.8万亿种组合
- 小写字母避免大小写混淆
- 前缀便于调试识别

### 2.4 AppState状态定义

集中式状态管理：

```typescript
export type AppState = DeepImmutable<{
  // 核心配置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  
  // 权限上下文
  toolPermissionContext: ToolPermissionContext
  
  // MCP服务
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
  }
  
  // 插件系统
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    errors: PluginError[]
  }
  
  // 远程连接
  replBridgeEnabled: boolean
  replBridgeConnected: boolean
}>
```

`DeepImmutable`确保所有属性为只读，防止意外修改。

---

## 第3章：查询引擎

### 3.1 QueryEngine类设计

QueryEngine管理完整的会话生命周期：

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  
  constructor(config: QueryEngineConfig) {
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.readFileState = config.readFileCache
  }
}
```

### 3.2 submitMessage核心流程

```typescript
async *submitMessage(prompt, options): AsyncGenerator<SDKMessage> {
  // 1. 预处理用户输入
  const { messagesFromUserInput } = await processUserInput({ ... });
  
  // 2. 记录会话transcript
  if (persistSession) {
    await recordTranscript(messages);
  }
  
  // 3. 构建system prompt
  const { defaultSystemPrompt, userContext } = await fetchSystemPromptParts();
  const systemPrompt = asSystemPrompt([...]);
  
  // 4. 调用核心query循环
  for await (const message of query({ messages, systemPrompt, ... })) {
    switch (message.type) {
      case 'assistant':
        this.mutableMessages.push(message);
        yield* normalizeMessage(message);
        break;
      case 'progress':
        yield message;
        break;
    }
  }
  
  // 5. 生成最终结果
  yield { type: 'result', subtype: 'success', ... };
}
```

### 3.3 AsyncGenerator流式输出

使用Generator实现流式输出：

```typescript
export async function* query(params: QueryParams): AsyncGenerator<
  | StreamEvent
  | Message
  | ToolUseSummaryMessage,
  Terminal
> {
  for await (const apiResponse of callClaudeAPI(...)) {
    // 处理流事件
    yield apiResponse;
    
    // 工具调用
    if (apiResponse.type === 'tool_use') {
      const results = await runTools(...);
      yield* results;
    }
  }
}
```

**优势**：
- 实时进度反馈
- 支持中断（AbortController）
- 内存高效（不需要缓存全部结果）

### 3.4 权限追踪

```typescript
const wrappedCanUseTool: CanUseToolFn = async (tool, input, context) => {
  const result = await canUseTool(tool, input, context);
  
  // 追踪拒绝决策用于SDK报告
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    });
  }
  
  return result;
};
```

### 3.5 预算控制

```typescript
// USD预算检查
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield { type: 'result', subtype: 'error_max_budget_usd' };
  return;
}

// 结构化输出重试限制
if (callsThisQuery >= maxRetries) {
  yield { type: 'result', subtype: 'error_max_retries' };
  return;
}
```

---

## 第4章：命令系统

### 4.1 命令类型定义

支持三种命令类型：

```typescript
export type Command = CommandBase & (
  | PromptCommand    // 扩展为文本提示
  | LocalCommand     // 本地执行
  | LocalJSXCommand  // 渲染React UI
)

// PromptCommand - 发送给模型
type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  allowedTools?: string[]
  model?: string
  getPromptForCommand(args, context): Promise<ContentBlockParam[]>
}

// LocalCommand - 本地执行
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

// LocalJSXCommand - UI组件
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}
```

### 4.2 命令来源层次

| 来源 | 位置 | 描述 |
|------|------|------|
| `builtin` | commands.ts | 核心系统命令 |
| `bundled` | skills/bundled | 预打包技能 |
| `skills` | .claude/skills/ | 用户自定义技能 |
| `plugin` | 插件系统 | 来自插件的命令 |
| `mcp` | MCP服务器 | Protocol提供的命令 |

### 4.3 命令注册与加载

```typescript
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  agents,
  branch,
  // ... 60+命令
  
  // 条件命令
  ...(webCmd ? [webCmd] : []),
  ...(forkCmd ? [forkCmd] : []),
])

export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd);
  
  return allCommands.filter(
    cmd => meetsAvailabilityRequirement(cmd) && isCommandEnabled(cmd)
  );
}
```

### 4.4 懒加载设计

命令实现延迟加载：

```typescript
const insights: Command = {
  type: 'prompt',
  name: 'insights',
  description: 'View usage insights',
  async getPromptForCommand(args, context) {
    // 懒加载113KB大文件
    const real = (await import('./commands/insights.js')).default;
    return real.getPromptForCommand(args, context);
  },
}
```

### 4.5 命令执行流程

```
用户输入 "/command args"
    │
    ▼
parseSlashCommand() 解析
    │
    ▼
hasCommand() 检查存在
    │
    ▼
getCommand() 获取对象
    │
    ├─ local-jsx → load().call() + 渲染UI
    │
    ├─ local → load().call() + 返回文本
    │
    └─ prompt → getPromptForCommand() + 扩展为消息
```

### 4.6 远程模式安全命令

```typescript
export const REMOTE_SAFE_COMMANDS: Set<Command> = new Set([
  session, exit, clear, help, theme, cost, usage
]);

export function filterCommandsForRemoteMode(commands: Command[]): Command[] {
  return commands.filter(cmd => REMOTE_SAFE_COMMANDS.has(cmd));
}
```

---

## 架构设计总结

### 分层架构

```
┌─────────────────────────────────────────┐
│           main.tsx (入口层)              │
├─────────────────────────────────────────┤
│         QueryEngine.ts (引擎层)          │
├─────────────────────────────────────────┤
│     Tool.ts + commands.ts (抽象层)       │
├─────────────────────────────────────────┤
│          tools/ (实现层)                 │
├─────────────────────────────────────────┤
│          state/ (状态层)                 │
└─────────────────────────────────────────┘
```

### 设计模式应用

| 模式 | 应用位置 |
|------|---------|
| 工厂模式 | buildTool() |
| 策略模式 | Tool接口 |
| 发布订阅 | Store |
| 依赖注入 | ToolUseContext |
| AsyncGenerator | query() |
| 状态机 | Task |
| 懒加载 | commands |