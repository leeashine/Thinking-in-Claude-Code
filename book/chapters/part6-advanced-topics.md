# 第六部分：高级主题

> 本章涵盖Claude Code的高级特性，展示系统的扩展性和深度集成能力。

---

## 第23章：Vim集成

### 23.1 Vim模式设计

Claude Code提供完整的Vim模式支持：

```typescript
// vim/motions.ts
export type Motion = 
  | { type: 'h' }  // 左
  | { type: 'j' }  // 下
  | { type: 'k' }  // 上
  | { type: 'l' }  // 右
  | { type: 'w' }  // 词首
  | { type: 'b' }  // 词尾
  | { type: 'e' }  // 词末
  | { type: '0' }  // 行首
  | { type: '$' }  // 行尾
  | { type: 'gg' } // 文件首
  | { type: 'G' }  // 文件尾
```

### 23.2 操作符

```typescript
// vim/operators.ts
export type Operator =
  | 'd'  // 删除
  | 'y'  // 复制
  | 'c'  // 修改
  | 'p'  // 粘贴
  | 'u'  // 撤销
  | 'r'  // 重做
```

### 23.3 文本对象

```typescript
// vim/textObjects.ts
export type TextObject =
  | 'iw'  // 内部词
  | 'aw'  // 一个词
  | 'i"'  // 内部引号
  | 'a"'  // 包含引号
  | 'i('  // 内部括号
  | 'a('  // 包含括号
```

### 23.4 状态转换

```typescript
// vim/transitions.ts
type VimMode = 'normal' | 'insert' | 'visual' | 'command';

function handleKeyPress(state: VimState, key: string): VimState {
  switch (state.mode) {
    case 'normal':
      if (key === 'i') return { ...state, mode: 'insert' };
      if (key === 'v') return { ...state, mode: 'visual' };
      if (key === ':') return { ...state, mode: 'command' };
      // ...
      break;
    case 'insert':
      if (key === 'Escape') return { ...state, mode: 'normal' };
      // 插入字符
      break;
  }
}
```

---

## 第24章：历史管理

### 24.1 会话历史

```typescript
// history.ts
export function addToHistory(message: Message): void {
  const history = loadHistory();
  history.push({
    ...message,
    timestamp: Date.now(),
  });
  saveHistory(history);
}

export function getHistory(limit?: number): Message[] {
  const history = loadHistory();
  return limit ? history.slice(-limit) : history;
}
```

### 24.2 会话持久化

```typescript
// assistant/sessionHistory.ts
type SessionRecord = {
  id: string
  messages: Message[]
  createdAt: number
  updatedAt: number
  metadata: SessionMetadata
}

export async function persistSession(session: SessionRecord): Promise<void> {
  const path = getSessionPath(session.id);
  await writeFile(path, JSON.stringify(session));
}

export async function loadSession(id: string): Promise<SessionRecord | null> {
  const path = getSessionPath(id);
  try {
    const content = await readFile(path, 'utf-8');
    return JSON.parse(content);
  } catch {
    return null;
  }
}
```

### 24.3 历史压缩

长会话需要压缩以适应上下文窗口：

```typescript
export function compactHistory(messages: Message[]): Message[] {
  const result: Message[] = [];
  let summary = '';
  
  for (const message of messages) {
    if (shouldCompact(message)) {
      summary += extractKeyPoints(message);
    } else {
      if (summary) {
        result.push(createSummaryMessage(summary));
        summary = '';
      }
      result.push(message);
    }
  }
  
  return result;
}
```

---

## 第25章：成本追踪

### 25.1 成本计算

```typescript
// cost-tracker.ts
type Usage = {
  inputTokens: number
  outputTokens: number
  cacheCreationInputTokens?: number
  cacheReadInputTokens?: number
}

type ModelPricing = {
  inputCostPerToken: number
  outputCostPerToken: number
  cacheCreationCostPerToken?: number
  cacheReadCostPerToken?: number
}

export function calculateCost(usage: Usage, pricing: ModelPricing): number {
  let cost = 0;
  cost += usage.inputTokens * pricing.inputCostPerToken;
  cost += usage.outputTokens * pricing.outputCostPerToken;
  
  if (usage.cacheCreationInputTokens && pricing.cacheCreationCostPerToken) {
    cost += usage.cacheCreationInputTokens * pricing.cacheCreationCostPerToken;
  }
  
  if (usage.cacheReadInputTokens && pricing.cacheReadCostPerToken) {
    cost += usage.cacheReadInputTokens * pricing.cacheReadCostPerToken;
  }
  
  return cost;
}
```

### 25.2 预算控制

```typescript
export function checkBudget(maxBudgetUsd: number): boolean {
  const totalCost = getTotalCost();
  return totalCost < maxBudgetUsd;
}

export function getRemainingBudget(maxBudgetUsd: number): number {
  return Math.max(0, maxBudgetUsd - getTotalCost());
}
```

### 25.3 成本报告

```typescript
export function generateCostReport(session: Session): CostReport {
  return {
    totalCost: session.totalCost,
    byModel: groupBy(session.apiCalls, 'model').map(([model, calls]) => ({
      model,
      calls: calls.length,
      inputTokens: sum(calls, 'inputTokens'),
      outputTokens: sum(calls, 'outputTokens'),
      cost: sum(calls, 'cost'),
    })),
    cacheSavings: calculateCacheSavings(session),
  };
}
```

---

## 第26章：权限系统

### 26.1 权限模式

```typescript
type PermissionMode =
  | 'default'      // 默认：敏感操作需要确认
  | 'acceptEdits'  // 自动接受编辑
  | 'plan'         // 计划模式：先规划后执行
  | 'bypass'       // 绕过：自动执行所有操作（危险）
```

### 26.2 权限规则

```typescript
type PermissionRule = {
  rule: string
  behavior: 'allow' | 'deny' | 'ask'
  source: 'user' | 'managed' | 'default'
}

// 规则示例
const rules: PermissionRule[] = [
  { rule: 'Bash(git status:*)', behavior: 'allow', source: 'user' },
  { rule: 'Bash(rm -rf:*)', behavior: 'deny', source: 'user' },
  { rule: 'Edit(.env:*)', behavior: 'ask', source: 'default' },
];
```

### 26.3 权限检查流程

```typescript
async function checkPermissions(tool: Tool, input: unknown, context: ToolUseContext): Promise<PermissionResult> {
  const permissionContext = context.getAppState().toolPermissionContext;
  
  // 1. 检查全局deny规则
  if (matchesDenyRule(tool.name, input, permissionContext)) {
    return { behavior: 'deny', message: 'Blocked by deny rule' };
  }
  
  // 2. 检查全局allow规则
  if (matchesAllowRule(tool.name, input, permissionContext)) {
    return { behavior: 'allow' };
  }
  
  // 3. 检查工具特定规则
  const toolResult = await tool.checkPermissions?.(input, context);
  if (toolResult) return toolResult;
  
  // 4. 应用权限模式
  switch (permissionContext.mode) {
    case 'acceptEdits':
      if (tool.isReadOnly?.(input)) return { behavior: 'allow' };
      break;
    case 'bypass':
      return { behavior: 'allow' };
  }
  
  // 5. 默认询问用户
  return { behavior: 'ask', message: `允许执行 ${tool.name}?` };
}
```

### 26.4 敏感操作检测

```typescript
const SENSITIVE_PATTERNS = [
  /rm\s+-rf/,
  /sudo/,
  />\s*\/dev\/sd/,
  /DROP\s+TABLE/i,
];

function isSensitiveBashCommand(command: string): boolean {
  return SENSITIVE_PATTERNS.some(p => p.test(command));
}
```

---

## 第27章：技能系统

### 27.1 技能定义

```yaml
# .claude/skills/my-skill/skill.md
---
name: my-skill
description: My custom skill
whenToUse: Use this skill when...
allowedTools:
  - Bash(npm:*)
  - Read
  - Write
---

# My Skill

This skill does X, Y, Z...

## Instructions

1. First, do X
2. Then, do Y
3. Finally, do Z
```

### 27.2 技能加载

```typescript
// skills/loadSkillsDir.ts
export async function loadSkillFromDir(dir: string): Promise<Command | null> {
  const skillFile = path.join(dir, 'skill.md');
  if (!existsSync(skillFile)) return null;
  
  const content = await readFile(skillFile, 'utf-8');
  const { frontmatter, body } = parseFrontmatter(content);
  
  return {
    type: 'prompt',
    name: frontmatter.name,
    description: frontmatter.description,
    allowedTools: frontmatter.allowedTools,
    async getPromptForCommand(args, context) {
      return substituteArguments(body, args);
    },
  };
}
```

### 27.3 技能发现

```typescript
export async function discoverSkillDirs(paths: string[], cwd: string): Promise<string[]> {
  const skillDirs: string[] = [];
  
  for (const filePath of paths) {
    const dir = findSkillDir(filePath, cwd);
    if (dir && !skillDirs.includes(dir)) {
      skillDirs.push(dir);
    }
  }
  
  return skillDirs;
}

function findSkillDir(filePath: string, cwd: string): string | null {
  // 向上查找包含.claude/skills的目录
  let dir = path.dirname(filePath);
  while (dir !== cwd) {
    const skillsDir = path.join(dir, '.claude', 'skills');
    if (existsSync(skillsDir)) {
      return skillsDir;
    }
    dir = path.dirname(dir);
  }
  return null;
}
```

---

## 第28章：MCP集成

### 28.1 MCP协议概述

Model Context Protocol (MCP) 是Anthropic定义的标准协议，用于连接AI模型与外部工具：

```
Claude Code <--MCP Protocol--> MCP Server <---> Tools/Resources
```

### 28.2 MCP服务器配置

```json
// .claude/settings.json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/dir"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

### 28.3 MCP客户端实现

```typescript
// services/mcp/client.ts
export async function connectMCPServer(
  config: McpServerConfig
): Promise<MCPServerConnection> {
  // 启动子进程
  const process = spawn(config.command, config.args, {
    env: { ...process.env, ...config.env },
  });
  
  // 创建JSON-RPC连接
  const transport = new StdioClientTransport(process.stdin, process.stdout);
  const client = new Client({ name: 'claude-code', version: VERSION }, {
    capabilities: { tools: {}, resources: {} },
  });
  
  await client.connect(transport);
  
  // 获取工具列表
  const { tools } = await client.request(ListToolsRequestSchema, {});
  
  // 获取资源列表
  const { resources } = await client.request(ListResourcesRequestSchema, {});
  
  return {
    process,
    client,
    tools: tools.map(convertMCPTool),
    resources,
  };
}
```

### 28.4 MCP工具调用

```typescript
export async function callMCPTool(
  connection: MCPServerConnection,
  toolName: string,
  args: Record<string, unknown>
): Promise<ToolResult> {
  const response = await connection.client.request(CallToolRequestSchema, {
    name: toolName,
    arguments: args,
  });
  
  return {
    content: response.content,
    isError: response.isError,
  };
}
```

### 28.5 MCP资源访问

```typescript
export async function readMCPResource(
  connection: MCPServerConnection,
  uri: string
): Promise<ResourceContent> {
  const response = await connection.client.request(ReadResourceRequestSchema, {
    uri,
  });
  
  return response.contents[0];
}
```

---

## 扩展性设计总结

### Vim模式
完整的状态机实现，支持扩展motions和operators。

### 历史管理
会话持久化、压缩、恢复。

### 成本追踪
精确计算、预算控制、报告生成。

### 权限系统
多层检查、规则配置、模式切换。

### 技能系统
声明式定义、动态加载、参数替换。

### MCP集成
标准协议、多服务器、工具与资源统一管理。