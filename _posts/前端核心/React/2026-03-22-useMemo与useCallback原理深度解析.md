---
layout: post
title: "useMemo与useCallback原理"
date: 2026-03-22
categories: ["前端核心", "React"]
tags: ["React", "Hooks", "useMemo", "useCallback", "性能优化", "面试"]
---

## 一句话概括

`useMemo` 缓存计算结果，`useCallback` 缓存函数引用——两者底层完全相同（Hook 链表中存 `[value, deps]`，每次渲染用 `Object.is` 浅比较依赖），区别只在一个自动执行工厂函数、一个直接存传入的函数。`useCallback(fn, deps)` 本质是 `useMemo(() => fn, deps)` 的语法糖。

## 核心知识点

### 1. 为什么需要这两个 Hook？

```jsx
// ❌ 每次渲染都重新排序、创建新函数
function TodoList({ todos }) {
  const sorted = todos.sort((a, b) => b.priority - a.priority);
  const handleClick = () => fetch(`/api/todo/${id}`);

  return <ExpensiveTable data={sorted} onClick={handleClick} />;
  // handleClick 每次都是新引用 → React.memo 的 ExpensiveTable 白 memo 了
}
```

两个问题：计算浪费 CPU + 新引用导致子组件重渲染白做。`useMemo` 管第一个，`useCallback` 管第二个。

### 2. Fiber 上的存储结构

```typescript
// 每个 Hook 在 Fiber.memoizedState 链表中有一席之地
type Hook = {
  memoizedState: [cachedValue, depsArray],  // 缓存值和依赖
  next: Hook | null,                        // 下一个 Hook
  queue: UpdateQueue | null,
};

// useMemo 的 mount 阶段
function mountMemo(factory, deps) {
  const hook = mountWorkInProgressHook();
  const value = factory();
  hook.memoizedState = [value, deps];  // 执行工厂，存结果
  return value;
}

// useCallback 的 mount 阶段
function mountCallback(callback, deps) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = [callback, deps];  // 不执行，直接存函数
  return callback;
}
```

### 3. 更新阶段的比较逻辑

```typescript
function updateMemo(factory, deps) {
  const hook = updateWorkInProgressHook();
  const [prevValue, prevDeps] = hook.memoizedState;

  if (areHookInputsEqual(deps, prevDeps)) {
    return prevValue;  // 依赖没变 → 返回缓存
  }

  const nextValue = factory();            // 依赖变了 → 重新计算
  hook.memoizedState = [nextValue, deps];
  return nextValue;
}

// 依赖比较：Object.is 的浅比较
function areHookInputsEqual(next, prev) {
  for (let i = 0; i < prev.length; i++) {
    if (!Object.is(next[i], prev[i])) return false;
  }
  return true;
}
```

记住：`Object.is({}, {})` 永远是 `false`。所以依赖项是引用类型时，每次渲染都"变了"。

### 4. useMemo vs useCallback 的选择

```
用 useMemo 当你想缓存一个计算结果 → useMemo(() => compute(data), [data])
用 useCallback 当你想缓存一个函数引用 → useCallback(() => doSomething(id), [id])
什么都不用 当计算结果廉价且不传给子组件 → 直接写
```

**黄金准则**：只有两种场景值得用——(1) 计算确实贵（大数据排序/过滤）(2) 函数要传给 `React.memo` 的子组件。其他场景反而增加开销。

### 5. React.memo + useCallback 的经典配合

```jsx
const Child = React.memo(({ onClick, label }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>{label}</button>;
});

function Parent() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // ❌ 不用 useCallback → 每次 Parent 渲染，Child 也渲染
  // const handleClick = () => setCount(c => c + 1);

  // ✅ 用 useCallback → Child 只在首次渲染
  const handleClick = useCallback(() => setCount(c => c + 1), []);
  // setCount 是稳定引用，不需要加进 deps

  return (
    <>
      <input value={text} onChange={...} />
      <Child onClick={handleClick} label="+1" />
    </>
  );
}
```

## 其实你每天都在用

- **表格排序**：1000 条数据按薪资排序，`useMemo(() => data.sort(...), [data, sortKey])` 只在数据或排序键变化时才重新算
- **筛选过滤**：`useMemo(() => items.filter(...), [items, query])` 只在输入框打字变化时重新过滤
- **固定回调传给子组件**：`useCallback(() => toggleItem(id), [id])` 保证 `React.memo` 的列表项不会因为其他 item 的变化而重渲染
- **useEffect 的依赖**：`useEffect(() => { fetch(id) }, [fetchWithId])` 如果不用 useCallback 包，fetchWithId 每次变 → effect 死循环

## 常见误解

- **❌ 误区：「所有函数都应该用 useCallback 包一层」** 错。useCallback 本身也有开销（定位 Hook 链表 + 比较依赖数组），对于不传给子组件的普通函数反而拖慢性能。

- **❌ 误区：「useMemo 能替代 React.memo」** 不能。useMemo 缓存的是值（计算结果），React.memo 缓存的是组件渲染结果（跳过子树），两者在不同层面起作用。

- **❌ 误区：「deps 空数组意味着永远不重新计算」** 是的——但这是一个危险信号。如果你在 factory 里用了外部变量却没写进 deps，你拿到的永远是闭包中的旧值。这就是"Hooks 闭包陷阱"的根源。

- **❌ 误区：「useCallback(fn, []) 是最佳实践」** 不是。空 deps 意味着函数闭包永远持有初始 props/state。如果你需要读到最新值，要么写对依赖，要么用 `useRef` + 函数式 setState。

## 一句话总结

useMemo 和 useCallback 不是让代码"更快"的银弹，而是用缓存换子组件稳定性——只在传给 `React.memo` 子组件、或在 `useEffect` 中作为依赖时真正值得。React Compiler（Forget）将来会帮我们自动做这件事。
