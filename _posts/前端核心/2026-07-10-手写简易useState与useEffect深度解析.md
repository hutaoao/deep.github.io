---
layout: post
title: "手写简易useState与useEffect"
date: 2026-07-10
tags: [React, useState, useEffect, 手写]
categories: [前端核心, React]
---

## 一句话概括

通过实现一套极简的 React Hooks 运行时（≈200行核心代码），深入理解 `useState` 与 `useEffect` 的工作原理——包括 Fiber 链表存贮 Hook、mount/update 双阶段调度、依赖链比较、清理函数执行以及批量更新队列，彻底告别对 Hooks 的"黑盒恐惧"。

## 背景与意义

### 为什么需要手写 Hooks？

React Hooks（v16.8+）已经成为前端开发的标配，但很多开发者对 Hooks 的内部机制仍然一知半解：

- **为什么 Hooks 有严格的调用顺序要求？**
- **为什么不能在条件语句或循环中使用 Hook？**
- **为什么 useState 多次调用后，每次返回的 state 仍然正确？**
- **为什么 useEffect 第二个参数传入空数组只会执行一次？**
- **React 到底是怎么知道当前是 mount 还是 update 阶段的？**

这些问题的答案就在 React 源码中。通过手写一个简化版但功能完整的 Hooks 运行时，我们可以：

1. **打破黑盒**：亲眼看到 Hook 数据是怎么存储和读取的
2. **加深理解**：只有自己实现过，才能真正理解为什么 React 设计了这样的 API
3. **迁移能力**：了解了一版实现后，阅读其他框架（Preact、Vue Composition API）的类似设计也会更容易

本文实现的目标：一套约 200 行核心代码的 Hooks 运行时，支持：
- 带有批量更新的 `useState`
- 带依赖追踪和清理的 `useEffect`
- Fiber + 链表方式的 Hook 存储
- mount/update 双阶段调度
- 组件渲染模拟

> ⚠️ 说明：本文实现的是"概念验证级"（proof-of-concept）代码，不是 React 源码的精确拷贝。React 源码中有更多优化（如双向链表、优先级调度、fiber 树对比等），但核心思想和数据结构是完全一致的。

## 概念与定义

在动手写代码之前，先厘清几个核心概念。

### Fiber：React 的"工作单元"

在 React 16+ 中，每个组件对应一个 Fiber 节点。Fiber 可以看作是一个 **保存了组件状态和副作用信息的 JavaScript 对象**：

```js
// Fiber 节点的简化结构
const fiber = {
  type: 'Component',       // 函数组件本身
  memoizedState: null,     // 指向第一个 Hook 节点（链表头）
  stateNode: null,         // 组件的 DOM 节点或实例
  alternate: null,         // 当前 Fiber 对应的旧 Fiber（双缓冲）
  // ... 更多属性
};
```

### Hook 节点的链表结构

一个组件内可以调用多个 Hook。React 用 **单向链表** 将所有 Hook 串联起来：

```
memoizedState
   ↓
┌──────────────┐    next    ┌──────────────┐    next    ┌──────────────┐
│  Hook Node 1 │─────────▶│  Hook Node 2 │─────────▶│  Hook Node 3 │
│  (useState)  │           │  (useState)  │           │ (useEffect)  │
│  queue       │           │  queue       │           │  tag = effect │
│  state       │           │  state       │           │  destory     │
│  ...         │           │  ...         │           │  deps        │
└──────────────┘           └──────────────┘           └──────────────┘
```

**为什么必须是链表？**
- 每次组件渲染都会"重走"一遍所有 Hook 调用
- 通过索引（链表遍历的当前位置）来匹配每个 Hook 的调用
- 如果条件分支改变了调用顺序，链表遍历就会"对不上号"
- 这就是为什么 React 禁止在条件/循环中使用 Hook

### mount vs update：双阶段调度

React 内部通过一个全局指针 `currentlyRenderingFiber` 和一些布尔标记来区分阶段：

| 阶段 | 触发条件 | 行为 |
|------|----------|------|
| **mount** | 首次渲染 | 创建 Hook 节点，设置初始值 |
| **update** | 下次渲染 | 复用已有 Hook 节点，按照依赖决定是否更新 |

简单来说：**第一次是"创建"，后面都是"复用"**。

### 批量更新：Batched Updates

React 的 `setState` 并不是立即触发的。连续调用多次 `setState`，React 会：

1. 把它们都放到一个更新队列中
2. 合并或批量处理这些更新
3. 一次性触发重新渲染

这种机制称为"批量更新"（Batched Updates），它能避免不必要的重复渲染。

## 最小示例

在我们实现之前，先看看我们最终想达成的效果——也就是我们手写的 Hooks 用起来的样子：

```jsx
// ----- 手写 Hooks 的使用示例 -----
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');

  useEffect(() => {
    console.log('count 变了:', count);
    return () => {
      console.log('清理上一次的 effect');
    };
  }, [count]);

  return {
    render: () => console.log(`渲染: ${name} 按了 ${count} 次`),
    click: () => setCount(count + 1),
    changeName: (n) => setName(n),
  };
}
// ----- end -----
```

我们的 Hooks 运行时让开发者用几乎一样的语法来定义组件状态和副作用。

## 核心知识点拆解

### 1. 链表结构的 Hook 存储

#### 为什么用链表而不是数组？

其实用索引数组也可以，但链表有几个优势：

1. **扩展性**：后续插入节点时无需搬移元素
2. **与 Fiber 结构一致**：React Fiber 本身就是树状链表结构
3. **遍历成本低**：从头遍历到尾即可

我们的实现中，每个 Hook 节点长这样：

```js
function createHook() {
  return {
    memoizedState: null,   // 当前 state 值
    next: null,            // 下一个 Hook 的指针
    queue: {               // 更新队列
      pending: null,       // 待处理的 update 链表环
      last: null,          // 最后一次更新
    },
    // 以下属于 useEffect 专属字段
    tag: undefined,        // undefined | 'effect'
    destroy: null,         // effect 的清理函数
    deps: undefined,       // 依赖数组
  };
}
```

#### 链表遍历的"游标指针"

全局只维护一个指向当前 Fiber 上 Hook 链表的指针 `workInProgressHook`。每次调用 Hook 时：

- **mount** 阶段：创建新节点，链接到链表尾部
- **update** 阶段：从链表当前位置取下一个节点

```js
// 移动游标指针的伪代码
let workInProgressHook = null;
let currentHook = null;

function nextHook() {
  if (currentHook === null) {
    // update 阶段：从 alternate 的 fiber 上读取旧的 Hook 链表
    workInProgressHook = currentlyRenderingFiber.memoizedState;
  } else {
    // 依次后移
    workInProgressHook = workInProgressHook.next;
    currentHook = currentHook.next;
  }
  return workInProgressHook;
}
```

这个设计带来了一个非常重要的结论：**每个 Hook 的"编号"就是它在 React 源码中被调用的顺序**。

### 2. mount 与 update 的双态设计

我们需要区分组件是首次渲染还是更新渲染。用一个布尔变量即可：

```js
let isMount = true;   // 全局标记：当前是否是 mount 阶段
```

在 useState 的实现中：

```js
function useState(initialValue) {
  let hook;

  if (isMount) {
    // mount: 创建新的 Hook 节点
    hook = { ...createHook(), memoizedState: initialValue };
    // 链接到 Fiber 的链表
    mountHookToFiber(hook);
  } else {
    // update: 从已有的链表中按顺序取 Hook
    hook = getNextHook();
  }

  // ... 返回 [state, dispatch]
}
```

> 核心思想：**mount 创建、update 复用**。同一个函数，不同的行为。

### 3. useReducer 核心——useState 的底层基础

很多人不知道，`useState` 其实是 `useReducer` 的语法糖：

```js
// 在 React 源码中，useState 本质上就是
function useState(initial) {
  return useReducer(basicStateReducer, initial);
}
// 其中 basicStateReducer = (state, action) => action;
```

这意味着如果你实现了 `useReducer`，`useState` 就自然成立了。我们的极简实现也遵循这个思路。

### 4. 批量更新机制的模拟

React 的批量更新通过两种方式实现：

1. **同步上下文（Legacy Mode）**：在事件处理函数中自动开启批量模式
2. **并发模式（Concurrent Mode）**：使用 Scheduler 调度

我们的实现用了一个简单的方案：

```js
let isBatching = false;
let updateQueue = [];

function batchUpdates(fn) {
  isBatching = true;
  fn();
  isBatching = false;
  flushUpdates();
}

function createSetState(hook, component) {
  return (action) => {
    const newState = typeof action === 'function'
      ? action(hook.memoizedState)
      : action;

    hook.queue.pending = newState;

    if (!isBatching) {
      scheduleRender(component);
    }
    // 如果正在批量模式，积压更新，等 flush 时统一渲染
  };
}
```

### 5. useEffect 的依赖比较

`useEffect` 的"魔法"在于它的依赖比较机制：

```js
function useEffect(create, deps) {
  let hook;

  if (isMount) {
    // mount: 直接执行 effect
    hook = createHook('effect');
    hook.deps = deps;
    hook.destroy = create();
  } else {
    // update: 比较依赖
    hook = getNextHook();
    const prevDeps = hook.deps;

    // 如果 deps 没传（undefined），每次都执行
    // 如果 deps 传了空数组，只执行一次
    // 否则逐个比较
    const hasChanged = deps
      ? !prevDeps || deps.some((dep, i) => !Object.is(dep, prevDeps[i]))
      : true;

    if (hasChanged) {
      // 先执行旧的清理函数
      if (hook.destroy) {
        hook.destroy();
      }
      // 再执行新的 effect
      hook.destroy = create();
      hook.deps = deps;
    }
  }
}
```

依赖比较的关键细节：

- **`Object.is` 比较**：React 用 `Object.is` 来比较每个依赖是否发生了变化（与 `===` 基本一致，但能区分 +0/-0 和 NaN）
- **依赖数组为 `undefined`**：每次都要重新执行
- **依赖数组为空 `[]`**：没有元素，`some` 循环 0 次 → `hasChanged = false` → 只在 mount 执行一次

## 实战案例

好，现在我们把这些知识融会贯通，写一个完整的实现。

### 完整实现代码（≈200 行）

```js
// ======================================================
// 手写简易 React Hooks 运行时
// 包含：useState、useEffect、Fiber 模拟、批量更新
// 约 200 行可运行代码
// ======================================================

// ----- 全局状态 -----
let workInProgressFiber = null;   // 当前正在渲染的 Fiber
let workInProgressHook = null;    // 当前正在处理的 Hook 节点
let currentFiber = null;          // 上一次渲染的 Fiber（双缓冲）
let hookIndex = 0;               // 用于 debug 的 Hook 索引
let isMount = true;              // 标记是否是首次挂载

// ----- 组件注册表 -----
const componentRegistry = new Map();   // key: rootId, value: Fiber 树

// ----- batch 批量更新 -----
let isBatching = false;
const batchQueue = new Set();          // 批量队列：去重的待渲染组件

// ============================================================
// 1. Fiber 节点创建
// ============================================================

/**
 * 创建 Fiber 节点
 * @param {Function} component - 函数组件
 * @param {Object} props - 组件的 props
 * @returns {Object} fiber 节点
 */
function createFiber(component, props = {}) {
  return {
    type: component,          // 函数组件引用
    props,                    // props
    memoizedState: null,      // Hook 链表头部
    alternate: null,          // 指向 oldFiber（双缓冲）
    hooksCount: 0,            // Hook 数量（辅助校验）
    component: null,          // 组件渲染后的返回结果
  };
}

// ============================================================
// 2. Hook 链表操作
// ============================================================

/**
 * 创建单个 Hook 节点
 * @param {string} [tag] - Hook 类型标记（'effect'）
 * @returns {Object} hook 节点
 */
function createHookNode(tag) {
  return {
    memoizedState: null,          // 当前的 state 值
    next: null,                   // 下一个 Hook 指针
    queue: { pending: null },     // 更新队列
    tag,                          // 标记：undefined 为 state，'effect' 为 effect
    destroy: null,                // useEffect 的清理函数
    deps: undefined,              // useEffect 的依赖数组
  };
}

/**
 * mount 阶段：创建并链接新的 Hook 节点
 * @param {string} [tag]
 * @returns {Object} 新创建的 hook 节点
 */
function mountWorkInProgressHook(tag) {
  const hook = createHookNode(tag);

  if (!workInProgressFiber.memoizedState) {
    // 第一个 Hook → 设为链表头
    workInProgressFiber.memoizedState = hook;
  } else {
    // 非第一个 → 追加到链表尾部
    workInProgressFiber.memoizedState.next = hook;
  }

  workInProgressHook = hook;
  hookIndex++;
  return hook;
}

/**
 * update 阶段：从旧 Fiber 的 Hook 链表中按顺序读取
 * @returns {Object} 当前 Hook 节点
 */
function updateWorkInProgressHook() {
  // 从 currentFiber（旧 Fiber）读取已挂载的 Hook 链表
  if (currentFiber && currentFiber.memoizedState) {
    if (!hookIndex) {
      // 第一个 Hook：从链表头开始
      workInProgressHook = currentFiber.memoizedState;
      currentFiber = currentFiber.memoizedState;
    } else {
      // 后续 Hook：依次后移
      workInProgressHook = workInProgressHook.next;
      currentFiber = currentFiber.next;
    }
  }

  hookIndex++;
  return workInProgressHook;
}

// ============================================================
// 3. useState 实现
// ============================================================

/**
 * 简化版 useState（基于 useReducer 思想）
 * @param {*} initial - 初始值
 * @returns {[*, Function]} [state, setState]
 */
function useState(initial) {
  let hook;

  if (isMount) {
    // mount：创建新 Hook 节点
    hook = mountWorkInProgressHook();
    hook.memoizedState = typeof initial === 'function' ? initial() : initial;
  } else {
    // update：从旧链表中读取已有 Hook
    hook = updateWorkInProgressHook();

    // 处理积压的更新
    if (hook.queue.pending !== null) {
      // 应用所有待处理的更新（支持函数式更新）
      hook.memoizedState = typeof hook.queue.pending === 'function'
        ? hook.queue.pending(hook.memoizedState)
        : hook.queue.pending;
      hook.queue.pending = null;
    }
  }

  // 保存当前 fiber 的引用供 setState 闭包使用
  const fiberNode = workInProgressFiber;
  const currentHook = hook;

  // 创建 dispatch 函数
  function setState(action) {
    // 计算新状态
    const newState = typeof action === 'function'
      ? action(currentHook.memoizedState)
      : action;

    // 入队
    currentHook.queue.pending = newState;

    // 非批量模式下立即触发调度
    if (!isBatching) {
      scheduleRender(fiberNode.type);
    } else {
      // 批量模式下收集到集合中，去重
      batchQueue.add(fiberNode.type);
    }
  }

  return [hook.memoizedState, setState];
}

// ============================================================
// 4. useEffect 实现
// ============================================================

/**
 * 简化版 useEffect
 * @param {Function} create - effect 回调
 * @param {Array} [deps] - 依赖数组
 */
function useEffect(create, deps) {
  let hook;

  if (isMount) {
    // mount：创建新的 effect Hook
    hook = mountWorkInProgressHook('effect');
    hook.deps = deps;
    // 立即执行 effect，捕获清理函数
    hook.destroy = create();
  } else {
    // update：复用旧 Hook，进行依赖比较
    hook = updateWorkInProgressHook();
    const prevDeps = hook.deps;

    // 判断依赖是否有变化
    // 规则：
    //   deps === undefined → 每次都执行
    //   deps === [] → 只执行一次（mount 已执行）
    //   否则 → 逐个比较
    const hasChanged = deps
      ? !prevDeps || deps.some((dep, i) => !Object.is(dep, prevDeps[i]))
      : true;

    if (hasChanged) {
      // 先执行上一次的清理函数（如果有）
      if (hook.destroy) {
        hook.destroy();
      }
      // 执行新的 effect
      const newDestroy = create();
      hook.destroy = newDestroy;
      hook.deps = deps;
    }
    // 如果没有变化，什么都不做
  }
}

// ============================================================
// 5. 渲染调度引擎
// ============================================================

/**
 * 调度组件渲染（在微任务或下一帧中异步执行）
 * @param {Function} component - 函数组件
 */
function scheduleRender(component) {
  // 使用 Promise 模拟 React 的异步调度
  // 在真正的 React 中，这里会有 Scheduler 的优先级调度
  Promise.resolve().then(() => {
    renderComponent(component);
  });
}

/**
 * 核心：渲染一个组件
 * @param {Function} component - 函数组件
 * @returns {*} 组件返回值
 */
function renderComponent(component) {
  // ---- Phase 1: 准备阶段 ----
  // 获取或创建 Fiber
  const rootFiber = componentRegistry.get(component.name) || createFiber(component);

  // 保存旧 Fiber，建立双缓冲关联
  currentFiber = rootFiber.alternate;
  workInProgressFiber = rootFiber;
  workInProgressHook = null;
  hookIndex = 0;

  // 判断阶段：有 alternate 说明是 update
  isMount = currentFiber === null;

  // ---- Phase 2: 执行组件函数 ----
  // 调用组件函数，过程中会依次执行 useState/useEffect
  // 这会触发 mountWorkInProgressHook 或 updateWorkInProgressHook
  const output = component(props || {});

  // ---- Phase 3: 收尾 ----
  // 更新 alternate，为下一次渲染做准备
  // 注意：这里做的是浅拷贝，真实 React 会更复杂
  rootFiber.alternate = {
    ...rootFiber,
    memoizedState: cloneHookChain(rootFiber.memoizedState),
  };

  // 注册或更新
  componentRegistry.set(component.name, rootFiber);

  // 重置全局状态
  currentFiber = null;
  workInProgressFiber = null;
  workInProgressHook = null;
  hookIndex = 0;

  return output;
}

// ============================================================
// 6. 批量更新 API
// ============================================================

/**
 * 批量更新：积压多次 setState 更新，最终统一渲染
 * @param {Function} fn - 要执行的更新函数
 */
function batchUpdates(fn) {
  isBatching = true;
  fn();
  isBatching = false;
  flushBatchUpdates();
}

/**
 * 刷新批量队列
 */
function flushBatchUpdates() {
  for (const component of batchQueue) {
    renderComponent(component);
  }
  batchQueue.clear();
}

// ============================================================
// 7. 工具函数
// ============================================================

/**
 * 深拷贝 Hook 链表（防止引用污染）
 * @param {Object} hook - Hook 链表头
 * @returns {Object} 新的 Hook 链表头
 */
function cloneHookChain(hook) {
  if (!hook) return null;

  const newHook = { ...hook, next: null };
  let current = newHook;
  let original = hook.next;

  while (original) {
    current.next = { ...original, next: null };
    current = current.next;
    original = original.next;
  }

  return newHook;
}

// ============================================================
// 8. 暴露公共 API
// ============================================================

module.exports = {
  useState,
  useEffect,
  renderComponent,
  batchUpdates,
};
```

### 运行测试

下面我们写一个测试用例，看看这套实现能不能正常工作：

```js
// 从上述文件引入
const { useState, useEffect, renderComponent, batchUpdates } = require('./mini-hooks');

// 定义组件
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');

  useEffect(() => {
    console.log(`[Effect] count 更新为: ${count}`);

    return () => {
      console.log(`[Cleanup] 清理 count: ${count} 的副作用`);
    };
  }, [count]);

  useEffect(() => {
    console.log(`[Mount Effect] name 初始化为: ${name}`);
    // 没有依赖，只会在 mount 时执行一次
  }, []);

  console.log(`[Render] 当前状态: name="${name}", count=${count}`);

  return {
    count,
    name,
    setCount,
    setName,
  };
}

// ---- 首次渲染 ----
console.log('=== 首次渲染 (Mount) ===');
const comp = renderComponent(Counter);
// 预期输出：
// [Render] 当前状态: name="Alice", count=0
// [Effect] count 更新为: 0
// [Mount Effect] name 初始化为: Alice

// ---- 更新 count ----
console.log('\n=== 更新 count ===');
comp.setCount(1);
// 由于 setState 是异步的，等待微任务队列
setTimeout(() => {
  console.log('\n=== 更新 name ===');
  renderComponent(Counter);
}, 0);
```

**运行结果分析**（期望输出）：

```
=== 首次渲染 (Mount) ===
[Render] 当前状态: name="Alice", count=0
[Effect] count 更新为: 0
[Mount Effect] name 初始化为: Alice

=== 更新 count ===
[Cleanup] 清理 count: 0 的副作用  ← useEffect 清理函数
[Render] 当前状态: name="Alice", count=1
[Effect] count 更新为: 1
```

> 注意：这里用 `setTimeout` 模拟异步是因为 `Promise.resolve().then()` 的调度时机。在实际运行中，`setState` 会触发 `scheduleRender`，后者使用 microtask 调度，所以更新会在当前同步代码执行完毕后立即执行。

## 底层原理

### 从代码学到到的 React 设计哲学

#### （1）单链表作为 Hook 的"身份证明"

我们的实现用 **单向链表** 存储每个 Hook 的状态。每次 render 时，React 从链表头部开始，按顺序取出每个 Hook：

```js
// 第一次渲染 (Mount)
// 调用顺序: useState(0) → useState('Alice') → useEffect(fn)
// 链表:
// Node1(useState, 0) → Node2(useState, 'Alice') → Node3(useEffect, fn)

// 第二次渲染 (Update)
// 按照"相同的调用顺序"遍历链表
// useState → Node1 (取count)
// useState → Node2 (取name)
// useEffect → Node3 (比较依赖)
```

**为什么调用顺序必须固定？**

因为 Hook 没有任何"名称"或"标识符"，它们只靠 **链表遍历位置的顺序** 来找到自己的状态。如果你在条件分支中写 Hook：

```js
// ❌ 禁止这样做
if (count > 0) {
  useEffect(() => { /* ... */ });
}
```

第一次渲染时，`useEffect` 是链表中的第 3 个节点。
第二次渲染时（count > 0 不成立），这个 `useEffect` 没有被调用 → 链表遍历少了一个 → 后续所有 Hook 都偏移了！

#### （2）闭包陷阱的本质

理解了 Hook 的存储方式，闭包陷阱就一目了然了：

```js
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log(count);  // 始终是 0！
    }, 1000);

    return () => clearInterval(timer);
  }, []);  // 依赖为空 → 只执行一次

  return /* ... */;
}
```

为什么 `count` 始终是 0？因为：

- `useEffect` 只在 mount 时执行了一次
- 那次执行捕获的是当时的闭包环境——`count` 是 0
- 后续即使 `count` 更新了（新的 Hook 节点有了新值），那个 `setInterval` 回调里的 `count` 仍然指向旧的闭包

**解决方案** 就是加上 `count` 依赖：

```js
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);
  }, 1000);
  return () => clearInterval(timer);
}, [count]);  // count 变化时重新注册
```

或者用 `useRef`：

```js
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setInterval(() => {
    console.log(countRef.current);  // 始终访问最新值
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

#### （3）批量更新与计时器

我们的实现中，连续调用 `setCount` 会怎样？

```js
function handleClick() {
  setCount(count + 1);
  setCount(count + 1);
  setCount(count + 1);
}
```

在没有批量更新的情况下，这会导致 **3 次渲染**。而在 `batchUpdates` 中：

```js
batchUpdates(() => {
  setCount(c => c + 1);
  setCount(c => c + 1);
  setCount(c => c + 1);
});
```

最终只触发 **1 次渲染**。这看起来像是"合并"了。

但在我们的实现中，如果用普通的 `setCount(count + 1)`（非函数式），并且不开启 batch：

```
// 第一次 setCount: queue.pending = 1 (count: 0 → 1)
// 触发 scheduleRender
// 第二次 setCount: queue.pending = 1 (count: 0 → 1)
// 由于上一次调度还没执行，这里的 count 仍然捕获的是 0
// 三次都是 count + 1 = 1
```

**函数式更新的优势** 在这里就体现了：

```js
// 函数式：逐个修改，不是 .pending = action
setCount(prev => prev + 1);  // prev = 0 → 1
setCount(prev => prev + 1);  // prev = 1 → 2
setCount(prev => prev + 1);  // prev = 2 → 3
```

在 React 源码中，每个 `setState` 产生一个 `Update` 对象，形成一个环状链表挂在 `queue` 上。渲染时将链表中的所有更新依次应用到旧状态上，得到最终的新状态。这样就解决了**同时产生多个更新时的合并问题**。

#### （4）双缓冲（Double Buffering）

我们的实现中有一个细节：

```js
rootFiber.alternate = {
  ...rootFiber,
  memoizedState: cloneHookChain(rootFiber.memoizedState),
};
```

这就是 **双缓冲** 的简化版。为什么需要它？

因为渲染过程中可能会有多个 Fiber 节点并行工作。用两个 Fiber 树：
- **current**：当前屏幕上显示的（commit 过的）
- **workInProgress**：正在内存中构建的（未提交的）

当 `workInProgress` 构建完毕后，直接切换指向，新的就变成了 current。这样做的优点：

1. **不需要重建整棵树**：只需修改交替指针
2. **暂停/恢复**：React 可以在处理高优先级更新时中断低优先级的 workInProgress 构建
3. **避免视觉闪烁**：一次性切换，不是渐进式修改

### 和 React 真实实现的对比

| 特性 | 本文实现 | React 真实实现 |
|------|----------|----------------|
| Hook 存储 | 单链表 | 单链表 |
| Fiber | 简化的对象 | 完整 Fiber 架构（包括 child、sibling、return） |
| 优先级 | 无 | Lane 优先级模型 |
| 批处理 | 简单队列 | SyncBatched + Concurrent Mode |
| 调度 | Promise.then | Scheduler（时间切片 + 优先级） |
| 协调 | 无 | 完整的 Diff 算法（key、类型比较） |
| 错误处理 | 无 | Error Boundaries + Suspense |
| DevTools | 无 | 完整的 DevTools 支持 |

## 高频面试题解析

### Q1. 为什么不能在循环、条件或嵌套函数中使用 Hook？

**核心答案**：Hooks 依赖固定的调用顺序来匹配链表中的状态。如果顺序变了，后续 Hook 会 "拣到" 别人的状态。

**详细解释**：

假设第一次渲染时组件调用了 3 个 Hook：

```
调用: useState(A) → useState(B) → useState(C)
链表: NodeA(state=A) → NodeB(state=B) → NodeC(state=C)
```

第二次渲染时，如果 `useState(B)` 被包裹在 `if(false)` 中：

```
调用: useState(A) → useState(C)
链表: NodeA(state=A) → NodeB(state=B) → NodeC(state=C)
                     ↑ 偏移了！
```

`useState(C)` 会取到 NodeB（state=B）的数据，然后返回的 `[state, setState]` 中的 setState 错位了，整个状态就崩了。

**"规则"的本质**：这不是 ESLint 的武断规则，而是数据结构的自然约束。ESLint 的 `react-hooks/rules-of-hooks` 插件就是用来提示这个底层限制的。

### Q2. useState 的 set 和 class component 的 setState 有什么区别？

| 特性 | useState | class setState |
|------|----------|----------------|
| 合并方式 | **替换**旧状态 | **浅合并**旧状态 |
| 初始值 | 可以是函数 `useState(()=>expr)` | 无 |
| 更新方式 | 返回数组 `[state, setState]` | 合并到 this.state |
| 多个状态 | 多个 useState 调用 | 一个 this.state 对象 |

关键区别：**`useState` 是替换，class 的 `setState` 是合并**。

```js
// Class 组件
this.state = { count: 0, name: 'Alice' };
this.setState({ name: 'Bob' });
// this.state → { count: 0, name: 'Bob' }（保留count）

// 函数组件
const [state, setState] = useState({ count: 0, name: 'Alice' });
setState({ name: 'Bob' });
// state → { name: 'Bob' }（count 丢失！）
```

这也是为什么通常用多个 `useState` 而不是一个大型对象的原因。

### Q3. 为什么 useEffect 的第二个参数传 `[]` 只执行一次？

从我们的代码中可以直观看到原因：

```js
const hasChanged = deps
  ? !prevDeps || deps.some((dep, i) => !Object.is(dep, prevDeps[i]))
  : true;
```

当 `deps = []` 时：
- `deps.some(...)` 循环 0 次 → 返回 `false`
- `hasChanged = false`
- 只有第一次 mount 时执行了 `create()`
- 后续 update 阶段因为 `hasChanged = false`，什么都不做

### Q4. useEffect 的清理函数何时被调用？

两个时机：

1. **组件卸载时**：销毁所有 effect
2. **依赖变化时**：执行下一次 effect 之前，先清理上一次

我们的代码展示了第二种情况：

```js
if (hasChanged) {
  // 先清理
  if (hook.destroy) {
    hook.destroy();
  }
  // 再执行
  const newDestroy = create();
  hook.destroy = newDestroy;
  hook.deps = deps;
}
```

每次依赖变化时，清理函数都会执行，**这能防止内存泄漏**——比如清除旧的定时器、取消旧的订阅等。

### Q5. 多个 useEffect 的执行顺序是什么？

**按照声明顺序同步执行**。在我们的链表中，顺序就是 `memoizedState → next → next → ...` 的遍历顺序：

```js
// 声明
useEffect(() => { console.log('effect 1') }, []);
useEffect(() => { console.log('effect 2') }, []);
useEffect(() => { console.log('effect 3') }, []);

// 执行顺序: effect 1 → effect 2 → effect 3
```

但注意：`useEffect` 是 **在浏览器完成渲染之后** 异步执行的（类似 `setTimeout`）。React 中的 LayoutEffect（`useLayoutEffect`）是同步执行的，在我们的实现中没有区分。

## 总结与扩展

### 一张图总结

```
                    React Hooks 运行时
                ┌─────────────────────────┐
                │   Fibe—工作单元          │
                │   ┌─────────────────┐    │
                │   │ memoizedState → │    │  Hook (单向链)
                │   │ Node1 → Node2   │    │
                │   │ → Node3 → ...   │    │
                │   └─────────────────┘    │
                │         │                │
                │   ┌─────┴─────┐          │
                │   │  mount     │ update   │
                │   │ (创建)     │ (复用)   │
                │   └───────────┘          │
                │                          │
                │  useState  useEffect     │
                │  简化的 setter  依赖比较  │
                └─────────────────────────┘
```

### 核心收获

1. **Hook 就是链表**：每个 `useState`/`useEffect` 调用对应链表中的一个节点
2. **顺序即身份**：Hook 没有名字，只有位置——这就是调用顺序必须固定的原因
3. **mount vs update**：同一个函数根据阶段做不同的事（创建 vs 复用）
4. **依赖比较控制执行**：`useEffect` 的第二参数决定了副作用是否执行
5. **批量更新减少渲染**：多次 `setState` 合并为一次渲染

### 如何继续深入

如果这篇文章让你觉得意犹未尽，可以继续研究：

1. **React 源码**：看真正的 `ReactFiberHooks.js` 的实现
2. **Preact Hooks**：Preact 的实现更加简洁易读，只有几百行
3. **实现 useMemo / useCallback**：它们的本质就是带缓存的 Hook，在依赖不变时返回旧值
4. **实现 useRef**：本质上就是一个 `{ current: initialValue }` 对象，存在 Hook 节点上
5. **实现 useState 的完整 UpdateQueue**：支持环状链表 + 多个更新合并
6. **实现 useReducer**：`useState` 的底层基础，也是理解 Flux 架构的好起点

### 动手挑战

读完这篇文章后，试试自己实现这些：

- [ ] 给我们的运行时加上 `useMemo` 支持
- [ ] 实现 `useRef`（提示：它在 mount 时创建一个 `{ current: initial }` 对象）
- [ ] 支持 `useReducer`（提示：`useState(initial)` = `useReducer((s, a) => a, initial)`）
- [ ] 实现一个简单的 `useLayoutEffect`（同步执行 effect）
- [ ] 为我们的运行时添加 DevTools 支持，让每个 Hook 的值可视化

---

*"你无法真正理解一个东西，直到你亲手实现它。"* —— Richard Feynman
