# 第三部分：终端UI框架（Ink）

> Ink框架将React的声明式组件模型引入终端，实现了现代化的CLI应用开发体验。

---

## 第10章：Ink设计哲学

### 10.1 为什么选择React式终端UI

传统CLI开发面临挑战：
- 手动处理ANSI转义序列
- 难以管理复杂的UI状态
- 缺乏组件复用机制
- 布局计算繁琐

Ink的解决方案：
- **声明式UI**：描述"想要什么"，而不是"怎么做"
- **组件模型**：可复用的UI构建块
- **状态驱动**：UI = f(state)
- **Flexbox布局**：熟悉的CSS布局概念

### 10.2 核心架构

```
React组件 → Reconciler → DOM树 → Yoga布局 → Screen缓冲区 → ANSI输出
```

**渲染流程**：
1. React组件通过reconciler渲染到虚拟DOM
2. Yoga引擎计算布局
3. 输出到Screen缓冲区
4. 差分计算生成最小ANSI更新

### 10.3 与React的异同

**相同点**：
- 组件、Props、State
- 生命周期方法
- Hooks API
- Context API

**不同点**：
- 渲染目标：字符网格而非像素
- 布局引擎：Yoga而非浏览器布局
- 事件模型：键盘为主而非鼠标
- 性能考量：终端刷新频率限制

---

## 第11章：组件系统

### 11.1 自定义Reconciler

Ink使用react-reconciler创建自定义渲染器：

```typescript
// reconciler.ts
const reconciler = createReconciler<{
  ElementNames, Props, DOMElement, ...
}>({
  getRootHostContext: () => ({ isInsideText: false }),
  
  createInstance(type, props, root, hostContext): DOMElement {
    const node = createNode(type);
    
    // 应用属性到DOM节点
    for (const [key, value] of Object.entries(props)) {
      applyProp(node, key, value);
    }
    
    return node;
  },
  
  createTextInstance(text, root, hostContext): TextNode {
    return createTextNode(text);
  },
  
  appendChild(parent, child): void {
    parent.appendChild(child);
  },
  
  // ...其他生命周期方法
});
```

### 11.2 DOM节点类型

```typescript
export type ElementNames =
  | 'ink-root'    // 根节点
  | 'ink-box'     // 容器组件
  | 'ink-text'    // 文本组件
  | 'ink-virtual-text'  // 嵌套文本
  | 'ink-link'    // 链接
  | 'ink-raw-ansi' // 原始ANSI
```

### 11.3 Box组件

类似`<div style="display: flex">`：

```tsx
function Box({ 
  children, 
  flexDirection = 'row',
  flexGrow = 0,
  justifyContent = 'flex-start',
  alignItems = 'stretch',
  ...style 
}) {
  return (
    <ink-box
      style={{
        display: 'flex',
        flexWrap: 'nowrap',
        flexDirection,
        flexGrow,
        justifyContent,
        alignItems,
        ...style
      }}
    >
      {children}
    </ink-box>
  );
}
```

### 11.4 Text组件

支持丰富的文本样式：

```tsx
<Text 
  color="green"
  backgroundColor="black"
  bold
  italic
  underline
>
  Hello, World!
</Text>
```

**样式系统**：
- 16色：`'red'`, `'blue'`, ...
- 256色：`color={27}`
- RGB：`color="#ff5500"`

### 11.5 组件生命周期

```tsx
function MyComponent() {
  // 挂载
  useEffect(() => {
    console.log('Component mounted');
    return () => console.log('Component unmounted');
  }, []);
  
  // 更新
  useEffect(() => {
    console.log('Count changed:', count);
  }, [count]);
  
  return <Text>Count: {count}</Text>;
}
```

---

## 第12章：布局引擎

### 12.1 Yoga布局引擎

Ink集成Facebook的Yoga布局引擎：

```typescript
// layout/node.ts
export type LayoutNode = {
  // 树结构
  insertChild(child: LayoutNode, index: number): void
  removeChild(child: LayoutNode): void
  
  // 布局计算
  calculateLayout(width?: number, height?: number): void
  setMeasureFunc(fn: LayoutMeasureFunc): void
  
  // 样式设置
  setWidth(value: number): void
  setFlexDirection(dir: LayoutFlexDirection): void
  setAlignItems(align: LayoutAlign): void
  setJustifyContent(justify: LayoutJustify): void
}
```

### 12.2 Yoga适配器

```typescript
// layout/yoga.ts
export class YogaLayoutNode implements LayoutNode {
  readonly yoga: YogaNode;
  
  insertChild(child: LayoutNode, index: number): void {
    this.yoga.insertChild((child as YogaLayoutNode).yoga, index);
  }
  
  calculateLayout(width?: number): void {
    this.yoga.calculateLayout(width, undefined, Direction.LTR);
  }
  
  getComputedLayout(): { left: number; top: number; width: number; height: number } {
    return {
      left: this.yoga.getComputedLeft(),
      top: this.yoga.getComputedTop(),
      width: this.yoga.getComputedWidth(),
      height: this.yoga.getComputedHeight(),
    };
  }
}
```

### 12.3 Flexbox布局示例

```tsx
<Box flexDirection="column" padding={1}>
  <Box flexDirection="row" justifyContent="space-between">
    <Text>Name:</Text>
    <Text>Claude</Text>
  </Box>
  <Box flexDirection="row" justifyContent="space-between">
    <Text>Status:</Text>
    <Text color="green">Active</Text>
  </Box>
</Box>
```

渲染结果：
```
┌──────────────────┐
│ Name:    Claude  │
│ Status:  Active  │
└──────────────────┘
```

### 12.4 文本测量

动态测量文本尺寸：

```typescript
const measureTextNode = function (
  node: DOMNode,
  width: number,
  widthMode: LayoutMeasureMode,
): { width: number; height: number } {
  const text = expandTabs(node.textContent);
  const dimensions = measureText(text, width);
  
  // 处理换行
  if (dimensions.width > width) {
    const wrappedText = wrapText(text, width, node.style?.textWrap);
    return measureText(wrappedText, width);
  }
  
  return dimensions;
}
```

### 12.5 宽字符处理

处理中文字符、emoji等宽字符：

```typescript
export const enum CellWidth {
  Narrow = 0,      // 普通字符（宽度1）
  Wide = 1,        // 宽字符（宽度2）
  SpacerTail = 2,  // 宽字符的第二列
  SpacerHead = 3,  // 软换行延续
}
```

---

## 第13章：渲染系统

### 13.1 渲染流水线

```
scheduleRender → renderer() → renderNodeToOutput() → Output → Screen
```

### 13.2 主渲染器

```typescript
// renderer.ts
export default function createRenderer(node: DOMElement): Renderer {
  return options => {
    // 1. 设置终端尺寸
    node.yogaNode.setWidth(terminalWidth);
    
    // 2. 计算布局
    node.yogaNode.calculateLayout(terminalWidth);
    
    // 3. 渲染到Output
    const output = new Output();
    renderNodeToOutput(node, output, { prevScreen });
    
    // 4. 返回帧数据
    return {
      screen: output.get(),
      viewport: { width, height },
      cursor: { x: 0, y: screen.height, visible: !isTTY }
    };
  };
}
```

### 13.3 节点渲染

递归渲染DOM树：

```typescript
function renderNodeToOutput(
  node: DOMElement,
  output: Output,
  { offsetX = 0, offsetY = 0, prevScreen }
) {
  // 1. 检查是否可以blit（复用上一帧）
  if (prevScreen && canBlit(node, prevScreen)) {
    output.blit(prevScreen, x, y, width, height);
    return;
  }
  
  // 2. 处理overflow: scroll
  if (node.style.overflowY === 'scroll') {
    // 虚拟滚动处理
  }
  
  // 3. 裁剪到边界
  output.pushClipRect(x, y, width, height);
  
  // 4. 递归渲染子节点
  for (const child of node.childNodes) {
    renderNodeToOutput(child, output, { offsetX, offsetY });
  }
  
  output.popClipRect();
}
```

### 13.4 Screen缓冲区

```typescript
type Screen = {
  width: number
  height: number
  cells: Uint32Array  // 打包的单元格数据
  charPool: CharPool
  stylePool: StylePool
  hyperlinkPool: HyperlinkPool
}
```

**池化设计**：
- 字符串共享：相同文本只存储一次
- 样式共享：相同样式只生成一次ANSI序列
- 超链接共享：URL复用

### 13.5 增量渲染

```typescript
class LogUpdate {
  render(prev: Frame, next: Frame): Diff {
    // 差分计算
    const patches: string[] = [];
    
    diffEach(prev.screen, next.screen, (x, y, prevCell, nextCell) => {
      if (prevCell !== nextCell) {
        patches.push(moveCursor(x, y));
        patches.push(nextCell.toAnsi());
      }
    });
    
    return patches.join('');
  }
}
```

---

## 第14章：事件处理

### 14.1 事件调度器

支持完整的DOM事件模型：

```typescript
// dispatcher.ts
export class Dispatcher {
  dispatch(target: EventTarget, event: TerminalEvent): boolean {
    // 1. 收集监听器（捕获→目标→冒泡）
    const listeners = collectListeners(target, event);
    
    // 2. 按顺序执行
    for (const listener of listeners) {
      listener.call(event);
      if (event.propagationStopped) break;
    }
    
    return !event.defaultPrevented;
  }
}
```

**事件传播顺序**：
```
[root-capture] → [parent-capture] → [target] → [parent-bubble] → [root-bubble]
```

### 14.2 键盘事件

```typescript
export class KeyboardEvent extends TerminalEvent {
  readonly key: string      // 字符或键名
  readonly ctrl: boolean
  readonly shift: boolean
  readonly meta: boolean
  readonly superKey: boolean  // Cmd/Win
  
  constructor(parsedKey: ParsedKey) {
    super('keydown', { bubbles: true, cancelable: true });
    this.key = keyFromParsed(parsedKey);
  }
}
```

### 14.3 useInput Hook

```typescript
const useInput = (handler: InputHandler, options = {}) => {
  const { stdin, setRawMode } = useStdin();
  
  useEffect(() => {
    if (options.isActive !== false) {
      setRawMode(true);
    }
    
    const onInput = (data: Buffer) => {
      const key = parseKeypress(data);
      handler(key.toString(), key);
    };
    
    stdin.on('data', onInput);
    return () => {
      stdin.off('data', onInput);
      setRawMode(false);
    };
  }, [options.isActive]);
};
```

### 14.4 焦点管理

```typescript
export class FocusManager {
  activeElement: DOMElement | null = null;
  
  focus(node: DOMElement): void {
    if (node === this.activeElement) return;
    
    // 派发blur/focus事件
    if (this.activeElement) {
      this.dispatchEvent(new FocusEvent('blur', node));
    }
    
    this.activeElement = node;
    this.dispatchEvent(new FocusEvent('focus', previous));
  }
  
  focusNext(): void {
    const tabbable = collectTabbable(root);
    const index = tabbable.indexOf(this.activeElement);
    const next = tabbable[(index + 1) % tabbable.length];
    this.focus(next);
  }
}
```

### 14.5 useFocus Hook

```typescript
const useFocus = (options = {}) => {
  const { isActive } = options;
  const [isFocused, setFocused] = useState(false);
  
  useEffect(() => {
    if (!isActive) return;
    
    const onFocus = () => setFocused(true);
    const onBlur = () => setFocused(false);
    
    node.addEventListener('focus', onFocus);
    node.addEventListener('blur', onBlur);
    
    return () => {
      node.removeEventListener('focus', onFocus);
      node.removeEventListener('blur', onBlur);
    };
  }, [isActive]);
  
  return { isFocused, focus: () => manager.focus(node) };
};
```

---

## 设计思想总结

### 声明式UI
组件描述UI状态，Ink处理ANSI细节。

### 布局即代码
使用Flexbox API，布局逻辑清晰可读。

### 增量渲染
双层缓冲+差异diff，高效更新终端。

### 事件抽象
完整的DOM事件模型，键盘优先设计。

### 性能优化
池化、缓存、虚拟滚动、硬件滚动。