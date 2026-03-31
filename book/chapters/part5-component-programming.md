# 第五部分：组件化编程

> Claude Code的UI层展示了如何在终端环境中应用React式的组件化编程思想。

---

## 第19章：UI组件设计

### 19.1 组件目录组织

Claude Code的组件按功能分类：

```
components/
├── design-system/    # 设计系统（可复用）
│   ├── Dialog.tsx
│   ├── FuzzyPicker.tsx
│   ├── Pane.tsx
│   └── ListItem.tsx
├── agents/           # Agent相关组件
├── AutoUpdater.tsx   # 自动更新
├── BaseTextInput.tsx # 文本输入
└── ...
```

### 19.2 对话框组件模式

基于Promise的对话框设计：

```typescript
// interactiveHelpers.tsx
export function showDialog<T>(
  root: Root,
  renderer: (done: (result: T) => void) => React.ReactNode
): Promise<T> {
  return new Promise<T>(resolve => {
    const done = (result: T): void => {
      resolve(result);
      root.render(null);  // 清理
    };
    root.render(renderer(done));
  });
}
```

使用示例：

```typescript
const result = await showDialog(root, done => (
  <ConfirmDialog
    message="是否继续？"
    onConfirm={() => done(true)}
    onCancel={() => done(false)}
  />
));
```

### 19.3 FuzzyPicker虚拟列表

高效的列表选择器：

```tsx
function FuzzyPicker<T>({ items, onSelect, renderItem, ... }) {
  const [query, setQuery] = useState('');
  const [selectedIndex, setSelectedIndex] = useState(0);
  const { rows, columns } = useTerminalSize();
  
  // 响应式布局
  const isCompact = columns < 120;
  const visibleCount = Math.min(
    items.length,
    rows - CHROME_ROWS
  );
  
  // 虚拟滚动
  const startOffset = Math.max(0, selectedIndex - Math.floor(visibleCount / 2));
  const visibleItems = items.slice(startOffset, startOffset + visibleCount);
  
  return (
    <Box flexDirection="column">
      <TextInput value={query} onChange={setQuery} />
      <Box flexDirection="column">
        {visibleItems.map((item, i) => (
          <ListItem
            key={getKey(item)}
            isSelected={startOffset + i === selectedIndex}
          >
            {renderItem(item)}
          </ListItem>
        ))}
      </Box>
    </Box>
  );
}
```

### 19.4 复合组件模式

Dialog组合多个子组件：

```tsx
function Dialog({ title, children, onClose }) {
  return (
    <Pane borderStyle="round">
      <Box flexDirection="column">
        <Text bold>{title}</Text>
        <Box marginTop={1}>{children}</Box>
        <KeyboardShortcutHint shortcut="Esc" action="Close" />
      </Box>
    </Pane>
  );
}
```

---

## 第20章：状态管理

### 20.1 Store设计

发布订阅模式的Store：

```typescript
// state/store.ts
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(initialState: T): Store<T> {
  let state = initialState;
  const listeners = new Set<Listener>();

  return {
    getState: () => state,
    
    setState: updater => {
      const prev = state;
      const next = updater(prev);
      if (Object.is(next, prev)) return;  // 无变化跳过
      state = next;
      listeners.forEach(l => l());
    },
    
    subscribe: listener => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}
```

### 20.2 不可变性模式

DeepImmutable类型保护：

```typescript
type DeepImmutable<T> = {
  readonly [K in keyof T]: DeepImmutable<T[K]>
}

export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  // ...
}>
```

更新使用spread操作：

```typescript
context.setAppState(prev => ({
  ...prev,
  settings: {
    ...prev.settings,
    theme: 'dark',
  },
}));
```

### 20.3 选择性订阅

只在选定值变化时重渲染：

```typescript
// 推荐：独立调用
const verbose = useAppState(s => s.verbose);
const model = useAppState(s => s.mainLoopModel);

// 不推荐：返回新对象
const { text, id } = useAppState(s => ({ text: s.text, id: s.id }));

// 推荐：选择子对象
const { text, id } = useAppState(s => s.promptSuggestion);
```

### 20.4 Selector模式

纯函数selector无副作用：

```typescript
// state/selectors.ts
export function getViewedTeammateTask(state: AppState) {
  const { viewingAgentTaskId, tasks } = state;
  if (!viewingAgentTaskId) return undefined;
  
  const task = tasks[viewingAgentTaskId];
  if (!isInProcessTeammateTask(task)) return undefined;
  
  return task;
}
```

---

## 第21章：交互处理

### 21.1 键盘优先设计

所有交互都支持键盘操作：

```typescript
// keybindings/useKeybinding.ts
export function useKeybinding(
  key: string,
  callback: () => void,
  options: KeybindingOptions = {}
) {
  const { isActive = true, when } = options;
  
  useInput((input, key) => {
    if (!isActive) return;
    if (when && !when()) return;
    
    if (matchesKeybinding(input, key)) {
      callback();
    }
  });
}
```

### 21.2 退出状态机

处理Ctrl+C/D的复杂逻辑：

```typescript
export function useExitOnCtrlCDWithKeybindings() {
  const [pendingExit, setPendingExit] = useState(false);
  
  useInput((input, key) => {
    if (key.ctrl && (input === 'c' || input === 'd')) {
      if (hasActiveTask()) {
        // 有活动任务，提示确认
        setPendingExit(true);
      } else {
        // 无活动任务，直接退出
        process.exit(0);
      }
    }
  });
  
  return { pendingExit, cancelExit: () => setPendingExit(false) };
}
```

### 21.3 模式状态机

复杂交互使用状态机：

```typescript
type ModeState = 
  | { mode: 'select-event' }
  | { mode: 'select-matcher'; event: HookEvent }
  | { mode: 'select-hook'; event: HookEvent; matcher: string }
  | { mode: 'view-hook'; event: HookEvent; hook: HookConfig };

function HooksConfigMenu() {
  const [state, setState] = useState<ModeState>({ mode: 'select-event' });
  
  switch (state.mode) {
    case 'select-event':
      return <EventList onSelect={e => setState({ mode: 'select-matcher', event: e })} />;
    case 'select-matcher':
      return <MatcherList onSelect={m => setState({ mode: 'select-hook', ... })} />;
    // ...
  }
}
```

### 21.4 用户输入处理流程

```
用户输入
    │
    ▼
processUserInput()
    │
    ├─► 命令？→ processSlashCommand()
    │
    ├─► URL？→ handleUrlInput()
    │
    ├─► 文件路径？→ handleFileInput()
    │
    └─► 普通文本？→ handleTextInput()
```

---

## 第22章：Hooks应用

### 22.1 自定义Hooks

### useTerminalSize

```typescript
export function useTerminalSize() {
  const [size, setSize] = useState({
    rows: process.stdout.rows || 24,
    columns: process.stdout.columns || 80,
  });
  
  useEffect(() => {
    const onResize = () => {
      setSize({
        rows: process.stdout.rows,
        columns: process.stdout.columns,
      });
    };
    
    process.stdout.on('resize', onResize);
    return () => process.stdout.off('resize', onResize);
  }, []);
  
  return size;
}
```

### useDebounce

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);
  
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);
  
  return debouncedValue;
}
```

### useThrottle

```typescript
export function useThrottle<T extends (...args: any[]) => void>(
  callback: T,
  delay: number
): T {
  const lastCall = useRef(0);
  
  return useCallback((...args) => {
    const now = Date.now();
    if (now - lastCall.current >= delay) {
      lastCall.current = now;
      callback(...args);
    }
  }, [callback, delay]) as T;
}
```

### 22.2 上下文Hooks

### useApp

```typescript
const { exit, waitUntilExit } = useApp();

// 退出应用
exit();

// 等待退出
await waitUntilExit();
```

### useStdin / useStdout

```typescript
const { stdin, setRawMode } = useStdin();
const { stdout, write } = useStdout();

// 启用原始模式
setRawMode(true);

// 写入输出
write('\x1b[?25h');  // 显示光标
```

### 22.3 组合Hooks

构建复杂功能：

```typescript
function useSearchableList<T>(items: T[], filterFn: (item: T, query: string) => boolean) {
  const [query, setQuery] = useState('');
  const [selectedIndex, setSelectedIndex] = useState(0);
  
  const filteredItems = useMemo(
    () => items.filter(item => filterFn(item, query)),
    [items, query]
  );
  
  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(i => Math.max(0, i - 1));
    } else if (key.downArrow) {
      setSelectedIndex(i => Math.min(filteredItems.length - 1, i + 1));
    } else if (key.return) {
      return filteredItems[selectedIndex];
    }
  });
  
  return { query, setQuery, filteredItems, selectedIndex };
}
```

---

## 设计思想总结

### Provider层次
分层Provider管理状态、主题、键绑定。

### 状态不可变
DeepImmutable + spread更新保证安全。

### 键盘优先
所有交互支持键盘，模式状态机管理复杂流程。

### Hook复用
自定义Hooks封装通用逻辑。

### 响应式设计
终端尺寸变化自适应布局。