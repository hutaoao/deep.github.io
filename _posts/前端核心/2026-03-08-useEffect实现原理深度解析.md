---
layout: post
title: "useEffect实现原理深度解析"
date: 2026-03-08
tags: [React, useEffect, Hooks, Fiber]
categories: [前端核心, React]
---

## 一句话概括

`useEffect` 是 React 中用于在函数组件中执行**副作用**操作的 Hook，它通过 Fiber 架构的 effect 链表挂载副作用，利用调度器（Scheduler）以异步优先级执行回调，并在下次渲染前通过 `Object.is` 比对依赖项来决定是否跳过执行，从而实现与组件生命周期解耦的声明式副作用管理。

## 背景与意义

### 从 Class 组件到 Hooks 的范式革命

在 React 16.8 之前，函数组件被称为"无状态组件"（Stateless Component），只能接收 `props` 渲染 UI，无法使用状态、生命周期等能力。开发者必须使用 Class 组件来管理副作用，常见模式如下：

```jsx
class UserProfile extends React.Component {
  componentDidMount() {
    // 获取用户数据
    fetchUser(this.props.userId);
  }

  componentDidUpdate(prevProps) {
    // userId 变化时重新获取
    if (prevProps.userId !== this.props.userId) {
      fetchUser(this.props.userId);
    }
  }

  componentWillUnmount() {
    // 清理：取消订阅
    this.unsubscribe?.();
  }

  render() {
    return <div>{this.state.user.name}</div>;
  }
}
```

这种模式有几大痛点：

1. **逻辑分散**：同一个副作用的挂载、更新、卸载代码被迫分布在三个不同的生命周期方法中。
2. **代码复用困难**：难以在多个组件间共享有状态逻辑（HOC 和 Render Props 带来了 wrapper hell）。
3. **`this` 绑定问题**：需要手动绑定事件处理器或使用箭头函数。
4. **心智负担**：开发者需要时刻关注组件挂载/更新/卸载的生命周期阶段。

`useEffect` 的诞生彻底改变了这一切。它将相关的副作用逻辑聚合在一起，让开发者可以**按关注点（concern）而非生命周期阶段（phase）来组织代码**：

```jsx
function UserProfile({ userId }) {
  useEffect(() => {
    fetchUser(userId);
    const sub = subscribe();
    return () => sub.unsubscribe(); // 清理
  }, [userId]);
}
```

### 为什么需要理解 useEffect 的实现原理？

`useEffect` 看起来像魔法——"在依赖变化时执行某些操作"。但在实际开发中，许多令人困惑的行为（例如 `useEffect` 的执行时机为什么比 `useLayoutEffect` 晚？为什么严格模式下 effect 会执行两次？为什么闭包会捕获"过期"的值？）只有理解其底层实现才能真正掌握。

本文将从 React 源码入手，带领读者深入 `useEffect` 的实现细节，涵盖 Fiber 架构中的 effect 链表、调度机制、依赖比对、清理函数执行时机等核心知识点。

## 概念与定义

### Effect（副作用）

在 React 的语境中，副作用指任何与**纯渲染无关**的操作，包括：
- 数据获取（API 调用）
- 订阅/事件监听
- 手动修改 DOM
- 日志记录
- 定时器的设置与清除

### useEffect vs useLayoutEffect

这是开发者最常混淆的一对概念：

| 特性 | useEffect | useLayoutEffect |
|------|-----------|-----------------|
| 执行时机 | 浏览器完成布局与绘制**之后**（异步） | DOM 变更**之后**、浏览器绘制**之前**（同步） |
| 阻塞渲染 | 否 | 是 |
| 适用场景 | 数据获取、日志、非关键操作 | DOM 测量、视觉修复、避免闪烁 |
| 底层标记 | `Passive`（被动效果） | `Layout`（布局效果） |

本质区别在于：`useEffect` 的回调被标记为 **passive effect**，其执行被延迟到浏览器完成绘制之后；而 `useLayoutEffect` 的回调在 commit 阶段同步执行，会阻塞浏览器绘制。

### Passive Effect（被动效果）

`useEffect` 创建的副作用在 React 源码中被称为 **passive effect**，这个"被动"的含义是：
- 它是"被动"的——不会阻塞浏览器对用户的视觉更新
- 它以较低的优先级被调度执行
- 当浏览器有空闲时或者在绘制完成后执行

### Fiber（纤维）

Fiber 是 React 16 引入的**新的协调引擎**，每一个组件对应一个 Fiber 节点。Fiber 架构使 React 能够：
- 将渲染工作拆分成小单元
- 暂停、恢复、终止工作
- 为不同类型的工作分配优先级
- 复用已完成的工作

`useEffect` 的 effect 对象就存储在 Fiber 节点的 `updateQueue` 链表中。

## 最小示例

```jsx
import { useEffect, useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // 最基本的使用：每次渲染后执行
  useEffect(() => {
    document.title = `点击了 ${count} 次`;
  });

  // 带依赖数组：仅在 count 变化时执行
  useEffect(() => {
    console.log('count 更新为：', count);
  }, [count]);

  // 带清理函数：在组件卸载或重新执行前清理
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('tick');
    }, 1000);
    return () => {
      clearInterval(timer);
      console.log('定时器已清理');
    };
  }, []);

  return <button onClick={() => setCount(c => c + 1)}>点击</button>;
}
```

这个例子涵盖了 `useEffect` 的三种典型使用模式：
1. **无依赖**：每次渲染后都执行
2. **有依赖**：仅依赖项变化时执行
3. **空依赖+清理函数**：挂载时执行，卸载时清理

## 核心知识点拆解

### 1. 依赖项比较逻辑：`Object.is`

React 使用 `Object.is` 进行依赖项的**浅比较**。这是决定 effect 是否需要重新执行的关键。

```javascript
// React 源码中依赖比对的逻辑（简化版）
function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
): boolean {
  if (prevDeps === null) return false; // 首次渲染，必然不同

  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      // 相同，继续比下一个
      continue;
    }
    // 有任何不同则返回 false，表示需要重新执行
    return false;
  }
  return true;
}
```

**关键特点**：
- 使用 `Object.is` 而非 `===` 或 `==`。`Object.is` 与 `===` 几乎相同，区别在于：`Object.is(NaN, NaN)` 为 `true`，而 `NaN === NaN` 为 `false`；`Object.is(-0, +0)` 为 `false`，而 `-0 === +0` 为 `true`。
- 这只是**浅比较**：如果依赖项是对象或数组，传入新的引用会导致重新执行，即使内容相同。

```javascript
// 常见陷阱1：每次渲染创建新对象
const options = { threshold: 0.5 };
useEffect(() => {
  observe(options);
}, [options]); // 每次渲染都执行！因为 options 是新的对象

// 正确做法：用 useMemo 或重构
const options = useMemo(() => ({ threshold: 0.5 }), []);

// 常见陷阱2：函数作为依赖
useEffect(() => {
  fetchData(fn);
}, [fn]); // 如果 fn 是内联函数，每次渲染都不同

// 正确做法：用 useCallback 或把函数放在 effect 内部
```

### 2. 清理函数执行时机

`useEffect` 返回的清理函数在以下时机执行：

1. **组件卸载时**：清理函数在组件从 DOM 中移除前执行
2. **重新执行 effect 前**：当依赖项变化导致 effect 重新执行时，先执行上一次的清理函数，再执行新的 effect
3. **开发环境严格模式**：React 18 Strict Mode 会故意双重调用 effect，以帮助开发者发现未正确清理的副作用

```jsx
useEffect(() => {
  console.log('Effect 执行');

  return () => {
    console.log('清理函数执行');
  };
}, [dependency]);
```

执行顺序是：渲染 → 清理上一次 effect → 执行新的 effect（在浏览器绘制之后）。

对于 `useLayoutEffect`，顺序是：渲染 → 清理上一次 effect → 执行新的 effect（在浏览器绘制之前，同步）。

### 3. effect 链表结构：`fiber.updateQueue`

在 Fiber 节点上，`useEffect` 创建的 effect 对象通过 `updateQueue` 链表串联起来。

```typescript
// 简化的 effect 结构
interface Effect {
  tag: number;           // HookFlags: Passive(被动) | Layout(布局) | HasEffect(有副作用)
  create: () => void;    // 用户传入的回调函数
  destroy: (() => void) | null;  // 清理函数，即 create 的返回值
  deps: Array<any> | null;       // 依赖数组
  next: Effect | null;           // 指向下一个 effect 的指针
}
```

每个使用了 Hooks 的 Fiber 节点维护一个 `updateQueue`，它是一个单向循环链表：

```
fiber.updateQueue → Effect1 → Effect2 → Effect3 → (指向Effect1形成循环)
```

```javascript
// React 内部创建 effect 的逻辑（伪代码）
function pushEffect(tag, create, destroy, deps) {
  const effect = {
    tag,
    create,
    destroy,
    deps,
    next: null,
  };

  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // 第一个 effect，指向自己形成循环
    effect.next = effect;
    fiber.updateQueue = effect;
  } else {
    // 插入到链表尾部
    effect.next = updateQueue.next;
    updateQueue.next = effect;
    // 更新 lastEffect 指针
    fiber.updateQueue = effect;
  }

  return effect;
}
```

可以看出，每一个 `useEffect` 或 `useLayoutEffect` 调用都会向这个链表**尾部追加**一个 effect 节点。后续 React 在 commit 阶段会遍历这个链表，根据 `tag` 标志决定如何处理每个 effect。

### 4. Passive Effects 的调度：`scheduleCallback`

这是 `useEffect` **异步执行**的核心机制。React 使用内置的 **Scheduler**（调度器）来安排 passive effects 的执行。

```javascript
// React 源码中的流程（简化）
// 1. commit 阶段：将所有 passive effects 标记并收集
function commitRoot(root) {
  // ... 执行同步的 DOM 操作（mutation）

  // 如果有 passive effects（即 useEffect），安排异步执行
  if (rootWithPendingPassiveEffects !== null) {
    scheduleCallback(NormalSchedulerPriority, () => {
      // 异步执行所有 passive effects
      flushPassiveEffects();
    });
  }

  // 同步执行 useLayoutEffect 的清理和回调
  // commitLayoutEffects 中会处理 Layout effects
}

// 2. 浏览器完成绘制后，以闲时或低优先级执行
```

`scheduleCallback` 是 Scheduler 模块的核心 API，它根据优先级将回调放入任务队列，由 Scheduler 决定何时执行。对于 passive effects，使用的是**普通优先级**（NormalSchedulerPriority），这意味着：
- 它们不会阻塞浏览器绘制
- 它们可能会被更高优先级的任务（如用户输入响应）打断
- 浏览器绘制完成后尽快执行

### 5. `flushPassiveEffects` 执行流程

`flushPassiveEffects` 是真正执行所有 pending passive effects 的函数，其完整流程如下：

```javascript
function flushPassiveEffects() {
  // 5.1 先执行所有待清理的 destroy 函数（上一次 effect 的清理）
  //     这会遍历 fiber.updateQueue 链表中 tag 包含 HookPassive | HookHasEffect 的节点
  //     调用其 destroy 函数（如果有）
  const pendingPassiveHookEffects = rootWithPendingPassiveEffects;
  for (let i = 0; i < pendingPassiveHookEffects.length; i++) {
    const fiber = pendingPassiveHookEffects[i];
    const effect = fiber.updateQueue;
    do {
      if (effect.tag & HookPassive && effect.tag & HookHasEffect) {
        if (effect.destroy !== undefined) {
          // ⚠️ 执行清理函数
          effect.destroy();
          effect.destroy = undefined;
        }
      }
      effect = effect.next;
    } while (effect !== fiber.updateQueue);
  }

  // 5.2 再执行所有新的 create 函数
  //     再次遍历，调用 create 函数并将返回的 destroy 存回 effect
  for (let i = 0; i < pendingPassiveHookEffects.length; i++) {
    const fiber = pendingPassiveHookEffects[i];
    const effect = fiber.updateQueue;
    do {
      if (effect.tag & HookPassive && effect.tag & HookHasEffect) {
        // ⚠️ 执行用户提供的 effect 回调
        const destroy = effect.create();
        effect.destroy = destroy;
      }
      effect = effect.next;
    } while (effect !== fiber.updateQueue);
  }
}
```

**关键细节**：

1. **先销毁（destroy）再创建（create）**：保证每次 effect 重新执行前，上一次的清理已完成。这也是为什么在依赖变化时，清理函数会在新 effect 之前执行。

2. **执行时机**：`flushPassiveEffects` 被调度到浏览器绘制完成后异步执行。但在 `commitRoot` 中还会检查是否有 pending passive effects，如果有会**同步立即执行**以防止无限循环（例如在 `useEffect` 中更新状态导致重新渲染）。

3. **双重遍历**：先遍历执行所有 destroy，再遍历执行所有 create。这样保证了清理顺序和创建顺序都是正确的。

### 6. 销毁阶段：destroy 函数的执行

destroy 函数（即 `useEffect` 返回的清理函数）的执行流程如下：

```
组件卸载 / 依赖变化
    ↓
commit 阶段标记 fiber（flags |= Passive）
    ↓
异步调用 flushPassiveEffects
    ↓
遍历 pendingPassiveHookEffects
    ↓
对每个 HookEffect（tag 包含 HookPassive | HookHasEffect）：
    1. 如果 destroy 存在 → 执行 destroy()
    2. 如果 destroy 不存在 → 跳过
    3. 清空 destroy（设为 null/undefined）
    4. 将 create 标记为待执行（留在一会儿的 create 阶段）
    ↓
重新遍历 pendingPassiveHookEffects
    ↓
对同一个 HookEffect：
    1. 执行 create() 获取新的 destroy
    2. 存入 effect.destroy
```

**组件卸载时的特殊处理**：
```javascript
// 卸载时，React 会标记 fiber 为 Deletion
// 然后调用 commitDeletion 递归处理所有子 fiber
function commitDeletion(fiber) {
  // 递归处理子节点
  // ...
  // 在递归过程中，对每个 fiber 的 effect 链表调用 destroy
  // 此时不再执行 create
}
```

实际在源码中，卸载时清理是通过 `commitHookEffectListUnmount` 函数完成的，它会遍历 effect 链表并只执行 destroy，不会触发 create。

## 实战案例

### 案例 1：防抖搜索

```jsx
function SearchBox() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isSearching, setIsSearching] = useState(false);

  // 使用 useEffect 实现防抖搜索
  useEffect(() => {
    if (!query.trim()) {
      setResults([]);
      return;
    }

    setIsSearching(true);

    // 🔑 利用清理函数取消上一次的请求
    const controller = new AbortController();
    const timeoutId = setTimeout(async () => {
      try {
        const res = await fetch(`/api/search?q=${query}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        setResults(data);
      } catch (err) {
        if (err.name !== 'AbortError') {
          console.error('搜索出错:', err);
        }
      } finally {
        setIsSearching(false);
      }
    }, 300);

    // 🔑 清理：取消上一次的定时器和请求
    return () => {
      clearTimeout(timeoutId);
      controller.abort();
    };
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="搜索..."
      />
      {isSearching && <div>搜索中...</div>}
      <ul>
        {results.map(item => (
          <li key={item.id}>{item.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

**关键设计**：
- 每次 `query` 变化时，清理函数会清除上一个定时器并中止前一次请求
- 清理函数不仅用于组件卸载时的资源释放，还用于**取消上一次未完成的副作用**
- 这是 `useEffect` + 清理函数最经典的生产实践

### 案例 2：事件监听的正确管理

```jsx
function WindowSize() {
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    // 🔑 每次渲染都创建新函数，但 useEffect 通过依赖控制来避免频繁操作
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }

    handleResize();
    window.addEventListener('resize', handleResize);

    // 🔑 清理时移除监听器，避免重复绑定
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // 空依赖：仅在挂载时绑定

  return (
    <div>
      窗口大小：{size.width} × {size.height}
    </div>
  );
}
```

**注意事项**：
- 空依赖 `[]` 确保事件监听只绑定一次，但也意味着 effect 内部捕获的永远是**初始闭包**
- 这里 `handleResize` 内部调用 `setSize` 使用**函数式更新**，才能避免过期闭包问题
- 如果 effect 中依赖了外部的状态值（非 setter），空依赖会导致过期闭包 bug

### 案例 3：useEffect 与 useLayoutEffect 的选择

{% raw %}
```jsx
function Tooltip({ targetRect, content }) {
  const [tooltipStyle, setTooltipStyle] = useState({ top: 0, left: 0 });
  const tooltipRef = useRef(null);

  // 需要 DOM 测量 → 用 useLayoutEffect 避免闪烁
  useLayoutEffect(() => {
    if (!tooltipRef.current) return;

    const tooltipRect = tooltipRef.current.getBoundingClientRect();

    // 根据 Tooltip 和 target 位置计算最佳显示位置
    let top = targetRect.bottom + 8;
    let left = targetRect.left;

    // 如果 Tooltip 超出视口底部，改为显示在 target 上方
    if (top + tooltipRect.height > window.innerHeight) {
      top = targetRect.top - tooltipRect.height - 8;
    }

    // 如果 Tooltip 超出视口右侧，改为右对齐
    if (left + tooltipRect.width > window.innerWidth) {
      left = targetRect.right - tooltipRect.width;
    }

    setTooltipStyle({ top, left });
  }, [targetRect]);

  // 数据获取 → 用 useEffect 避免阻塞渲染
  useEffect(() => {
    logAnalyticsEvent('tooltip_shown', { content });
  }, [content]);

  return (
    <div
      ref={tooltipRef}
      style={{
        position: 'fixed',
        ...tooltipStyle,
      }}
    >
      {content}
    </div>
  );
}
```
{% endraw %}

**设计原则**：
- 需要读取 DOM 布局/尺寸、需要"视觉同步"的操作 → useLayoutEffect
- 数据获取、事件订阅、日志记录等非视觉操作 → useEffect
- React 官方建议**优先使用 useEffect**，只有遇到闪烁问题时才换 useLayoutEffect

### 案例 4：竞态条件（Race Condition）处理

```jsx
function UserData({ userId }) {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchData() {
      try {
        setError(null);
        const response = await fetch(`/api/user/${userId}`);
        const result = await response.json();

        // 🔑 如果请求返回时组件已经 unmount 或者 userId 已经变化
        if (!cancelled) {
          setData(result);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
        }
      }
    }

    fetchData();

    // 🔑 清理函数将 cancelled 置为 true，防止状态在卸载后更新
    return () => {
      cancelled = true;
    };
  }, [userId]);

  if (error) return <div>错误：{error}</div>;
  if (!data) return <div>加载中...</div>;
  return <div>{data.name}</div>;
}
```

**核心思路**：虽然无法取消 async/await 操作，但可以用一个 `cancelled` 标志配合清理函数来防止**在已卸载或已过期的组件上更新状态**。

## 底层原理（结合 React 源码分析）

现在是整篇文章最深入的部分。我们从 React 源码（React 18.2+）中追踪 `useEffect` 从调用到执行的完整链路。

### 第一阶段：mountEffect — 首次挂载

当组件首次渲染，调用 `useEffect` 时：

```javascript
// ReactFiberHooks.old.js （简化重构）
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  // 1. 获取当前正在工作的 Fiber 节点（mount 阶段）
  // 2. 创建 Hook 对象并挂载到 fiber.memoizedState 链表上
  const hook = mountWorkInProgressHook();

  // 3. 解析依赖数组
  const nextDeps = deps === undefined ? null : deps;

  // 4. 调用 pushEffect 挂载 effect 对象
  //    Tag 为 HookPassive（被动） | HookHasEffect（有副作用）
  //    HookPassive 标记这是 useEffect（异步）
  //    HookHasEffect 标记这个 effect 需要被执行（首次挂载必执行）
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,  // tag
    create,                        // create 回调
    undefined,                     // destroy 初始为 undefined
    nextDeps,                      // 依赖数组
  );
}
```

**关键点**：
- `mountWorkInProgressHook()` 创建新的 `Hook` 对象，挂到 `currentlyRenderingFiber.memoizedState` 的链表尾部
- `pushEffect` 创建 `Effect` 对象并挂载到 `fiber.updateQueue` 链表尾部
- 首次渲染时 **始终** 设置 `HookHasEffect` 标志，因为第一次必须执行

### 第二阶段：updateEffect — 更新阶段

每次重新渲染时（状态更新或 props 变化），调用 `useEffect` 走的是 `updateEffect`：

```javascript
function updateEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null,
): void {
  // 1. 获取当前 Hook 对象（从 fiber.memoizedState 链表中取出当前 hook）
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevEffect = hook.memoizedState; // 上一次的 effect 对象
  const prevDeps = prevEffect.deps;

  // 2. 🔑 最关键的一步：依赖比对
  if (prevDeps !== null && nextDeps !== null) {
    // 使用 Object.is 逐项比较依赖数组
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // 🔑 依赖未变化 → 仍然 pushEffect 但去掉 HookHasEffect 标志
      pushEffect(HookPassive, create, prevEffect.destroy, nextDeps);
      return;
    }
  }

  // 3. 依赖变了（或没有依赖数组）→ 标记需要执行
  // 重新 pushEffect，tag 包含 HookPassive | HookHasEffect
  // ⚠️ 清理函数（prevEffect.destroy）在新旧 effect 间传递
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,
    create,
    prevEffect.destroy, // 🔑 保留了上一次的 destroy，在 commit 阶段执行
    nextDeps,
  );
}
```

**关键点**：
- `areHookInputsEqual` 用 `Object.is` 逐项比较
- 即使依赖没变，**仍然会 pushEffect**（保留 effect 在链表中），但**不带** `HookHasEffect` 标志，这样在 commit 阶段会跳过执行
- 如果依赖变了，新的 effect 对象会**保留旧的 destroy 函数**，供 commit 阶段执行清理

### 第三阶段：commit 阶段 — 标记与收集

当 Reconciler 完成 diff 后，进入 commit 阶段：

```javascript
// ReactFiberWorkLoop.js（简化）
function commitRoot(root) {
  // ... 获取渲染后的 finishedWork

  // 3.1 收集所有包含 passive effect 的 fiber
  // 这些 fiber 上标记了 Passive 标志（二进制标志）
  if (
    (finishedWork.subtreeFlags & Passive) !== NoFlags ||
    (finishedWork.flags & Passive) !== NoFlags
  ) {
    // 如果有 passive effects，记录到全局变量
    rootWithPendingPassiveEffects = root;

    // 3.2 安排 passive effects 的异步执行
    scheduleCallback(NormalSchedulerPriority, () => {
      // 异步执行
      flushPassiveEffects();
      return null;
    });
  }

  // 3.3 同步执行 Layout effects（useLayoutEffect）
  // 这里会遍历 fiber.updateQueue 中 tag 为 HookLayout | HookHasEffect 的 effect
  // 先执行 destroy，再执行 create
  commitLayoutEffects(root, root);
}
```

这里可以看到 `useEffect` 和 `useLayoutEffect` 的分岔口：
- `useEffect` → `scheduleCallback` 调度到异步
- `useLayoutEffect` → 在 `commitLayoutEffects` 中**同步执行**

### 第四阶段：flushPassiveEffects — 真正执行

```javascript
// ReactFiberWorkLoop.js（简化）
function flushPassiveEffects(root) {
  // 防御：防止递归调用
  if (rootWithPendingPassiveEffects === null) return;

  let root = rootWithPendingPassiveEffects;

  // 4.1 🔑 先执行所有 pending 的 destroy 函数（上一轮 effect 的清理）
  // 通过递归遍历 fiber 树，收集所有 tag 包含 HookPassive | HookHasEffect 的 effect
  // 按深度优先顺序执行 destroy
  const unmountEffects = [];
  collectPassiveEffects(unmountEffects, root, 'unmount');
  for (let i = 0; i < unmountEffects.length; i++) {
    const destroy = unmountEffects[i];
    destroy(); // 执行清理函数
  }

  // 4.2 🔑 再执行所有新的 create 函数
  const mountEffects = [];
  collectPassiveEffects(mountEffects, root, 'mount');
  for (let i = 0; i < mountEffects.length; i++) {
    const create = mountEffects[i];
    const destroy = create(); // 执行 effect 回调，获取清理函数
    // 将 destroy 保存到对应的 effect 对象上，供下次使用
  }
}
```

**执行顺序的深层原因**：

```jsx
useEffect(() => {
  subscribeA();           // create A
  return () => unsubscribeA(); // destroy A
}, [depA]);

useEffect(() => {
  subscribeB();           // create B
  return () => unsubscribeB(); // destroy B
}, [depB]);
```

当 `depA` 变化时，执行顺序是：
```
渲染 → destroy A → create A → (浏览器绘制) → ...
```

当 `depA` 和 `depB` 都变化时：
```
渲染 → destroy A → destroy B → create A → create B → (浏览器绘制)
```

这种"先全部销毁、再全部创建"的模式，保证了所有清理函数都在任何新的 create 之前执行，避免了状态不一致的问题。

### 为什么严格模式下 effect 执行两次？

React 18 的 Strict Mode 在开发环境下会**模拟组件的卸载和重新挂载**：

```javascript
// Strict Mode 做的事情（简化）
// 1. 正常挂载组件 → 执行 useEffect 的 create
// 2. 立即"卸载"组件 → 执行 destroy
// 3. 重新挂载组件 → 再次执行 create
```

这就解释了为什么在 Strict Mode 下，effect 会执行两次。目的是帮助开发者发现：
- 清理函数是否被正确实现（不执行清理会导致内存泄漏）
- effect 是否具有幂等性（重复执行结果一致）
- 是否有"只该执行一次"的操作被错误地放在无依赖的 effect 中

### 整个链路的全景图

```
1. 函数组件调用 useEffect(create, deps)
    ↓
2. React 内部：
   - mountEffect (首次) / updateEffect (更新)
   - 创建 Hook 对象挂到 fiber.memoizedState
   - pushEffect 创建 Effect 挂到 fiber.updateQueue
   - 设置 tag（HookPassive ｜ [HookHasEffect]）
    ↓
3. Render 阶段（Reconciler）：
   - 组件 render，返回新的 React Element
   - 产出 Fiber 树（workInProgress → current 的切换）
    ↓
4. Commit 阶段（mutation）：
   - 同步执行 DOM 操作
   - 检查是否存在 passive effects
   - 如果有 → scheduleCallback(NormalPriority, flushPassiveEffects)
   - 同步执行 layout effects（useLayoutEffect）
    ↓
5. 浏览器完成绘制（Paint）
    ↓
6. flushPassiveEffects 执行（异步）：
   - 遍历所有 pending passive effects
   - 先执行所有 destroy（清理上一次的 effect）
   - 再执行所有 create（执行新的 effect 回调）
   - 将 destroy 存回 effect 对象
```

## 高频面试题解析

### Q1：useEffect 和 useLayoutEffect 的区别是什么？

**核心区别**在于执行时机：
- `useEffect`：commit 阶段被**调度为异步执行**，在浏览器完成布局和绘制后执行。不阻塞页面渲染。
- `useLayoutEffect`：在 commit 阶段的 `commitLayoutEffects` 中**同步执行**，在浏览器绘制之前完成，会阻塞视觉更新。

**适用场景**：
- 使用 `useEffect` 做一些不依赖 DOM 布局信息的操作（数据请求、事件绑定、日志）
- 使用 `useLayoutEffect` 在更新 DOM 后、浏览器绘制前，同步读取/修改 DOM（测量布局、避免闪烁）

**源码视角**：
```javascript
// useEffect 的 tag
HookPassive | HookHasEffect

// useLayoutEffect 的 tag
HookLayout | HookHasEffect
```

在 commit 阶段，只有 tag 包含 `HookPassive` 的会被异步调度；tag 包含 `HookLayout` 的会同步执行。

### Q2：useEffect 的依赖数组为什么不能省略？

省略依赖数组（或依赖数组声明不全）会导致**性能问题和逻辑错误**：

```jsx
// ❌ 无依赖数组：每次渲染都执行
useEffect(() => {
  doSomething(count);
}); // 本质：依赖项推导为 undefined，每次 rendo 都触发

// ❌ 依赖声明不全
useEffect(() => {
  const result = compute(count + multiplier);
  setResult(result);
}, [count]); // 缺少 multiplier → 闭包陷阱

// ✅ 正确：声明所有依赖
useEffect(() => {
  const result = compute(count + multiplier);
  setResult(result);
}, [count, multiplier]);
```

**最佳实践**：
1. 总是声明所有在 effect 外部使用到的响应式值（props、state、衍生值）
2. 使用 ESLint 插件 `eslint-plugin-react-hooks/exhaustive-deps` 自动检查
3. 如果不想某些变化触发 effect，考虑重新设计数据流（如使用 `useRef` 存储非响应式值）

### Q3：为什么 useEffect 的清理函数会在依赖变化时先执行？

这是 React 设计的一个重要选择，源码中 `flushPassiveEffects` 的"先 destroy 再 create"模式确保了：

1. **幂等性**：清理上一次操作后，执行新的操作，两者之间不会重叠
2. **一致性**：所有清理都发生在任何 create 之前，避免旧数据影响新操作
3. **资源安全**：旧的订阅、定时器在创建新的之前一定被清理

```jsx
useEffect(() => {
  socket.on('message', handleMessage);
  return () => socket.off('message', handleMessage);
}, [handleMessage]);
```

当 `handleMessage` 变化时：
1. **先** `socket.off('message', oldHandleMessage)` —— 移除旧监听
2. **再** `socket.on('message', newHandleMessage)` —— 绑定新监听

这样就不会出现两个监听同时存在或两个监听都不存在的窗口期。

### Q4：useEffect 返回的清理函数是何时被 React 保存的？

在 `pushEffect` 中：

```javascript
// mountEffect 时
pushEffect(HookPassive | HookHasEffect, create, undefined, deps);

// updateEffect 时——依赖变化
pushEffect(HookPassive | HookHasEffect, create, prevEffect.destroy, deps);

// updateEffect 时——依赖未变
pushEffect(HookPassive, create, prevEffect.destroy, deps);
```

关键点：
- **mount 阶段**：destroy 为 `undefined`
- **update 阶段**：新的 effect 对象会**继承**旧 effect 的 `destroy` 值（上一次 `create()` 的返回值）
- 当 effect 执行（`flushPassiveEffects`）时：
  - 先执行 `effect.destroy()`（如果存在）
  - 再执行 `effect.create()` 获取新的 `destroy`
  - 将新的 `destroy` 赋值回 `effect.destroy`

所以清理函数是**上一次** `useEffect` 回调执行时返回的，保存在 effect 对象上，在下一次渲染的 commit 阶段被调用。

### Q5：useEffect 在 React 18 并发模式下有什么变化？

React 18 引入了并发特性（Concurrent Features），`useEffect` 的语义在并发模式下有一些微妙变化：

1. **effect 可能被延迟执行**：在并发模式下，渲染可以被中断、暂停或放弃。这可能导致相关 effect 的执行时机比之前稍晚。

2. **Strict Mode 双重调用**：React 18 在开发环境下，无论是否使用并发特性，Strict Mode 都会严格执行"挂载-卸载-重新挂载"的检查。

3. **Transition 中的 effect**：当状态更新被包裹在 `startTransition` 中时，该更新触发的 effect 仍然会在浏览器绘制后执行，但渲染过程可以被打断。

4. **行为不变性**：React 18 团队强调 useEffect 的基本语义（异步执行、依赖比对、清理机制）在并发模式下**没有变化**，变化主要体现在渲染阶段的可中断性。

```jsx
function SearchResults() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // 这个 effect 在并发模式下行为不变
  useEffect(() => {
    let cancelled = false;
    fetch(`/search?q=${query}`)
      .then(r => r.json())
      .then(data => {
        if (!cancelled) setResults(data);
      });
    return () => { cancelled = true; };
  }, [query]);

  // 低优先级的更新
  const handleChange = (e) => {
    startTransition(() => {
      setQuery(e.target.value);
    });
  };

  return (
    <input onChange={handleChange} />
    // 渲染可能被打断，但 effect 的清理/执行逻辑不变
  );
}
```

## 总结与扩展

### 核心要点回顾

1. **Fiber 架构**：`useEffect` 基于 Fiber 架构的 effect 链表（`fiber.updateQueue`）存储副作用描述对象，每个 effect 包含 `tag`、`create`、`destroy`、`deps`、`next` 五个字段。

2. **异步调度**：通过 `scheduleCallback(NormalSchedulerPriority, flushPassiveEffects)` 将 passive effects 的执行延后到浏览器绘制之后，保证了页面渲染不被阻塞。

3. **依赖比对**：使用 `Object.is` 进行浅比较，相同的引用视为无变化，跳过 effect 执行（移除 `HookHasEffect` 标志）。

4. **先销毁后创建**：`flushPassiveEffects` 先遍历执行所有 destroy 函数，再遍历执行所有 create 函数，保证了操作的非重叠性。

5. **HookHasEffect 标志**：区分"需要执行"和"不需要执行"的 effect，但所有 effect 都会保留在链表中。

### 与 Class 组件生命周期的对比

| Class 组件 | useEffect 对应 |
|-----------|---------------|
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate` | `useEffect(() => { ... })` 或无依赖变化的 effect |
| `componentWillUnmount` | `useEffect(() => { return () => { ... } }, [])` 的清理函数 |
| `componentDidMount` + `componentDidUpdate` | `useEffect(() => { ... })` (无依赖/有依赖) |

### 与 Vue 的 watchEffect 对比

有趣的是，Vue 3 的 `watchEffect` 和 React 的 `useEffect` 虽然都用于管理副作用，但底层机制完全不同：

- **React useEffect**：每个组件渲染时重新追踪依赖（deps 的引用比较），依赖变化 → commit → flushPassiveEffects
- **Vue watchEffect**：依赖追踪在**执行回调时**由响应式系统自动收集（getter 拦截），依赖变化 → 同步重新执行

这体现了两种框架对"响应式"的不同理解：React 偏向显式声明 + 引用比较，Vue 偏向隐式追踪 + 赋值触发。

### 扩展思考：自定义 Hooks 对 useEffect 的再封装

理解 `useEffect` 的实现原理后，我们可以创建更强大的自定义 Hooks：

```javascript
// 自定义：只在挂载时执行一次的 effect
function useEffectOnce(effect) {
  useEffect(effect, []); // 🔑 空依赖
}

// 自定义：带防抖的 effect
function useDebouncedEffect(effect, deps, delay = 300) {
  useEffect(() => {
    const timer = setTimeout(effect, delay);
    return () => clearTimeout(timer);
  }, [...deps, delay]); // 🔑 将 delay 也加入依赖
}

// 自定义：首次渲染不执行的 effect
function useUpdateEffect(effect, deps) {
  const isFirst = useRef(true);
  useEffect(() => {
    if (isFirst.current) {
      isFirst.current = false;
      return;
    }
    return effect();
  }, deps);
}
```

这些自定义 Hooks 都依赖于对 `useEffect` 底层机制的理解——特别是依赖数组的作用、清理函数的执行时机、以及 `useRef` 跨渲染持久化。

### 写在最后

`useEffect` 是 React 函数组件中最常用也最容易被误用的 API。理解其底层实现——从 Fiber 中的 effect 链表管理，到 Scheduler 的异步调度，再到 flushPassiveEffects 的完整性保证——不仅能帮你写出更健壮的代码，还能让你在遇到诡异 bug 时找到正确的排查方向。

React 团队将 `useEffect` 设计为**声明式的、异步的、可组合的**副作用管理方案，用几十行核心代码（加上庞大的调度和协调系统）完成了 Class 组件时代需要三四个生命周期方法才能完成的工作。这不是魔法，是精致的系统工程。
