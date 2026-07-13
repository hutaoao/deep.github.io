---
layout: post
title: "手写简易useState与useEffect"
date: 2026-03-11
tags: [React, useState, useEffect, 手写]
categories: [前端核心, React]
---

## 一句话概括

通过写一套约 150 行的手写 Hooks 运行时——包含链表存储、mount/update 双阶段、批量更新队列和 useEffect 依赖比较——把 React Hooks 从"黑盒"变成"透明的数据结构操作"。

## 核心知识点

### 1. 数据模型：Fiber + Hook 链表 + 双阶段

```js
// 全局变量
let workInProgressFiber = null;   // 正在渲染的 fiber
let workInProgressHook = null;    // 当前处理的 Hook
let currentHook = null;           // 上一轮的同位置 Hook
let isMount = true;               // mount 还是 update
```

mount 时在链表尾部追加新节点，update 时游标步进读取已有节点：

```js
function mountWorkInProgressHook() {
  const hook = { memoizedState: null, queue: { pending: null }, next: null };
  if (!workInProgressFiber.memoizedState) {
    workInProgressFiber.memoizedState = hook;
  } else {
    workInProgressHook.next = hook; // 追加到尾部
  }
  return workInProgressHook = hook;
}

function updateWorkInProgressHook() {
  currentHook = currentHook ? currentHook.next : currentFiber.memoizedState;
  return cloneHook(currentHook); // 克隆到 workInProgress
}
```

### 2. useState ≈ useReducer 的语法糖

React 源码中 `useState(initial)` 本质就是 `useReducer(basicStateReducer, initial)`，其中 `basicStateReducer = (state, action) => action`。

```js
function useState(initial) {
  const hook = isMount ? mountWorkInProgressHook() : updateWorkInProgressHook();

  if (isMount) {
    hook.memoizedState = typeof initial === 'function' ? initial() : initial;
  } else {
    // 处理积压的更新
    if (hook.queue.pending !== null) {
      hook.memoizedState = typeof hook.queue.pending === 'function'
        ? hook.queue.pending(hook.memoizedState) // 函数式更新
        : hook.queue.pending;                    // 直接值
      hook.queue.pending = null;
    }
  }

  // 创建 setState，绑定当前 fiber 和 hook
  const setState = (action) => {
    hook.queue.pending = action;
    isBatching ? batchQueue.add(component) : scheduleRender(component);
  };

  return [hook.memoizedState, setState];
}
```

### 3. useEffect 核心逻辑：依赖比较 + 清理

```js
function useEffect(create, deps) {
  const hook = isMount ? mountWorkInProgressHook() : updateWorkInProgressHook();

  if (isMount) {
    hook.deps = deps;
    hook.destroy = create(); // 首次执行，保存清理函数
    return;
  }

  // update：比较依赖
  const prevDeps = hook.deps;
  const hasChanged = deps
    ? !prevDeps || deps.some((d, i) => !Object.is(d, prevDeps[i]))
    : true; // 无 deps → 每次都执行

  if (hasChanged) {
    hook.destroy?.();       // 先执行旧清理
    hook.destroy = create(); // 再执行新 effect
    hook.deps = deps;
  }
}
```

**关键细节**：`Object.is(NaN, NaN)` 返回 `true`，`Object.is(+0, -0)` 返回 `false`。这和 `===` 的行为略有不同。

### 4. 批量更新：多次 setState → 一次渲染

```js
let isBatching = false;
const batchQueue = new Set();

function batchUpdates(fn) {
  isBatching = true;
  fn();
  isBatching = false;
  // 集中触发一次渲染
  for (const comp of batchQueue) renderComponent(comp);
  batchQueue.clear();
}

// 使用
batchUpdates(() => {
  setCount(c => c + 1);
  setName('Bob');
  setAge(a => a + 1);
});
// 三次 set → 一次 re-render
```

### 5. 双缓冲：current ↔ workInProgress

每次渲染后用 `fiber.alternate` 建立两棵树的关系：

```js
function commitRoot(finishedWork) {
  root.current = finishedWork; // 交换指针
  // 旧的 current 变成 alternate，下次更新时可复用
}
```

mount 时没有 alternate → 创建全新节点。update 时 alternate 存在 → 复用并只更新可变字段，大幅减少 GC。

## 「其实你每天都在用」

1. **`useState` 返回的 `setState` 引用永远不变**：`queue.dispatch` 在 mount 时绑定到 `bind(null, fiber, queue)`，后续直接复用，所以不需要放在依赖数组。

2. **`useEffect(fn, [])` 只跑一次**：空数组 `some` 循环 0 次 → `hasChanged = false` → 只在 mount 执行，后续跳过。

3. **`useEffect(fn)` 无依赖每次都跑**：`deps === undefined` → `hasChanged = true` → 每次渲染都销毁重建。

4. **React 18 的自动批处理**：`setTimeout` 里的多次 `setState` 也会合并为一次渲染，不再需要 `unstable_batchedUpdates`。

5. **依赖比较用的 `Object.is`**：你传给 `useMemo` 的依赖数组，React 用 `Object.is` 逐个比较，这和 Redux 的 `shallowEqual` 不一样。

## 常见误解 (FAQ)

**❌ 误区 1："React 内部用数组存 Hook 状态"**

用数组的话，条件跳过一个 Hook 后后面的下标全错。用链表的话，跳过意味着 `next` 指针被跳过——同样错。数据结构不是关键，"按顺序取"的机制才是。

**❌ 误区 2："手写实现就是 React 源码"**

本文是概念验证代码，React 真实源码多了环状更新链表、Lane 优先级、Scheduler 调度、错误边界处理。但核心数据结构（链表 + 双阶段 + 双缓冲）是一致的。

**❌ 误区 3："useRef 的更新也走调度"**

`useRef` 的 `memoizedState` 存的是 `{ current: value }`。`ref.current = x` 直接修改对象属性，不经过 `queue.pending`，不触发调度。所以它是最简的 Hook。

**❌ 误区 4："useEffect 的清理函数在组件卸载时才调用"**

也在**依赖变化时**调用——每次执行新 effect 之前，先调用上一次的清理函数。StrictMode 下还会在 mount→remount 模拟中额外触发一次清理。

## 一句话总结

手写一遍 Hooks 就是最好的逆向工程课——等你亲手实现了链表遍历、依赖比较和批量合并，再看 React 源码就像读熟悉的老朋友。

