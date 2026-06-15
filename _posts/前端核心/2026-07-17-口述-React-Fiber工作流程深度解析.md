---
title: 口述：React Fiber工作流程深度解析
date: 2026-07-17
categories: [前端核心, React]
tags: [前端, React, Fiber, 工作流程]
description: 用口述讲解的方式，结合手绘图示意，深度解析React Fiber从render阶段到commit阶段的完整工作流程，涵盖beginWork、completeWork、commitRoot三大核心子流程的递进关系。
---

## 一句话概括

React Fiber 的渲染工作流程分为 render（构建/协调）和 commit（提交/生效）两大阶段，render 阶段可被高优先级任务中断，commit 阶段则同步一次性完成 DOM 变更，两者通过 Fiber 树的结构化遍历和副作用链（effectList）无缝衔接。

## 背景与意义

### 为什么需要理解 Fiber 工作流程？

如果你只使用 React 而不关心底层，写 `setState` → 组件重渲染 → DOM 更新这个"黑盒流程"已经足够。但当你需要：

- **排查性能问题**——为什么某个更新很慢？哪个子流程耗时最长？
- **调试诡异 Bug**——为什么组件生命周期函数调用了多次？
- **优化渲染路径**——怎样减少不必要的组件重渲染？
- **理解 Concurrent 模式**——为什么 `useTransition` 能让界面不卡顿？

——理解 Fiber 工作流程就不再是面试题，而是解决问题的**必备地图**。

想象你在一个复杂的地下停车场找车——如果知道"结构是 B2 层 E 区 120 号"，找起来比"在底层大概南边"快得多。Fiber 工作流程就是你导航所需要的"车库结构图"。

### Stack Reconciler 的回顾

在讲 Fiber 工作流程之前，先回忆 Stack Reconciler：

```
Stack Reconciler 的递归过程：
render(Component) {
  1. 递归 render 所有子组件，生成虚拟 DOM
  2. 与旧虚拟 DOM 进行 Diff 比较
  3. 生成 DOM 操作（增/删/改）
  4. 执行 DOM 操作
  
  整个过程：不可中断，一次性完成
}
```

Fiber 所做的改变是：**将这种"递归 + 一次性"改为"遍历 + 分片"**。

## 概念与定义

### 两大阶段

```
React Fiber 工作流程总览：

┌─────────────────────────────────────────────────────────┐
│                    触发更新（setState/dispatch）            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│    Render Phase（渲染阶段）           可中断（可被抢占）       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  beginWork（递）：自上而下遍历，标记变更                  │   │
│  │  ↓                                                 │   │
│  │  遇到子节点？继续下行，直到叶子节点                         │   │
│  │  ↓                                                 │   │
│  │  completeWork（归）：自下而上回溯，构建 effectList       │   │
│  └──────────────────────────────────────────────────┘   │
│                     ↓ 交出 effectList                    │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│    Commit Phase（提交阶段）           不可中断（同步执行）     │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Before Mutation：Snapshot/DOM 变更前                    │
│  │  ↓                                                 │   │
│  │  Mutation：执行 DOM 操作（增、删、改）                    │
│  │  ↓                                                 │   │
│  │  Layout：useLayoutEffect、DOM 变更后                    │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Render Phase（render 阶段）

也称为 "Reconciliation 阶段" 或 "协调阶段"。这个阶段做的事情是**构建或更新 workInProgress Fiber 树**，并标记所有需要进行的 DOM 变更（称为"副作用"或 flags）。

这个阶段是**可中断的**——Scheduler 可以在执行了 5ms 的时间切片后暂停工作，让出主线程。

### Commit Phase（commit 阶段）

这个阶段将 render 阶段产出的 effectList 应用到真实的 DOM 上。它是**同步且不可中断的**——一旦开始 commit，必须一口气完成所有 DOM 变更。

### EffectList（副作用链表）

在 render 阶段的 completeWork 过程中，React 将所有标记了副作用的 Fiber 节点串联成一个单链表。commit 阶段只需要遍历这个链表，而不需要重新遍历整棵 Fiber 树。

## 最小示例

用一个极简的组件树来追踪 Fiber 的遍历过程：

```jsx
function App() {
  return (
    <div>
      <span>Hello</span>
      <button>Click</button>
    </div>
  );
}
```

对应的 Fiber 树结构：

```
FiberRoot
  └── App (FunctionComponent)
       └── div (HostComponent)
            ├── span (HostComponent)
            │    └── "Hello" (HostText)
            └── button (HostComponent)
                 └── "Click" (HostText)
```

遍历过程的伪代码执行路线：

```
1. beginWork(App)
   → 判断是否需要更新 → 需要
   → 创建/复用 div Fiber → 进入子节点
   
2. beginWork(div)
   → 判断是否需要更新 → 需要
   → 进入第一个子节点 span
   
3. beginWork(span)
   → 判断是否需要更新 → 需要
   → 进入子节点 "Hello" (HostText)
   
4. beginWork("Hello")
   → HostText 没有子节点 → 进入 completeWork
   
5. completeWork("Hello")
   → 没有 stateNode → 创建文本节点
   → 父节点 span 获得 effect
   → 检查 sibling → button
   
6. beginWork(button)
   → 判断 → 检查子节点 "Click"
   
7. beginWork("Click")
   → 没有子节点 → 进入 completeWork
   
8. completeWork("Click")
   → 创建文本节点
   → 父节点 button 获得 effect
   → 检查 sibling → null
   
9. completeWork(button)
   → 创建 DOM 元素 <button>
   → 将子文本节点挂载
   → 父节点 div 获得 effect
   → 检查 sibling → null
   
10. completeWork(span)
    → 创建 DOM 元素 <span>
    → 挂载子文本节点
    → 父节点 div 获得 effect
    → 检查 sibling → button（但 button 已在 effectList 中）
    
11. completeWork(div)
    → 创建 DOM 元素 <div>
    → 将 span 和 button 挂载
    → 父节点 App 获得 effect
    
12. completeWork(App)
    → FunctionComponent 无需 DOM 创建
    → 将 div 的 effect 收集
    → 父节点 FiberRoot 获得 effect
    
13. completeWork(FiberRoot)
    → 所有 effectList 串联完成
    → 交给 commit 阶段执行
```

## 核心知识点拆解

### 1. beginWork——"递"的过程

`beginWork` 是 render 阶段的"下行"过程，从 FiberRoot 开始向下遍历每个节点。它的核心工作是：

```typescript
// 源码简化：packages/react-reconciler/src/ReactFiberBeginWork.js

function beginWork(current: Fiber | null, workInProgress: Fiber, renderLanes: Lanes): Fiber | null {
  // 1. 快速跳过（bailout）检查
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    if (oldProps === newProps && !hasContextChanged()) {
      // props 没有变，且没有 context 变化
      // → 检查是否有 childLanes（子节点的更新）
      if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
        // 子树也没有更新 → 完全跳过
        return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
      }
    }
  }
  
  // 2. 根据 tag 类型分发处理
  switch (workInProgress.tag) {
    case FunctionComponent: {
      return updateFunctionComponent(current, workInProgress, renderLanes);
    }
    case HostComponent: {
      return updateHostComponent(current, workInProgress, renderLanes);
    }
    case ClassComponent: {
      return updateClassComponent(current, workInProgress, renderLanes);
    }
    case HostText: {
      return updateHostText(current, workInProgress);
    }
    // ... 其他类型
  }
}
```

**beginWork 的核心决策**：
- **能否 bailout（跳过）？** 如果节点及其子树的 props 和 lanes 都没有变化，直接复用 current 树的子节点，不再深入
- **是否重新渲染？** 如果节点有更新，调用对应的 `update*` 函数，执行组件渲染函数、Diff 子节点等
- **返回什么？** 返回第一个要处理的子 Fiber 节点（如果有子节点），或 null（叶子节点或无障碍通过）

### 2. completeWork——"归"的过程

`completeWork` 是 render 阶段的"上行"过程。当一个节点及其所有子节点都被处理完毕（没有子节点，或所有子节点都 complete 了），React 进入 `completeWork`：

```typescript
// 源码简化
function completeWork(current: Fiber | null, workInProgress: Fiber): Fiber | null {
  const newProps = workInProgress.pendingProps;
  
  switch (workInProgress.tag) {
    case FunctionComponent:
      // 函数组件不需要创建 DOM 节点
      // 但需要调用 useEffect 的清理函数
      // 并记录 useLayoutEffect 和 useEffect
      return null;
      
    case HostComponent:
      // DOM 元素的创建/更新
      if (current !== null && workInProgress.stateNode !== null) {
        // 已有 DOM 节点：更新属性
        updateHostComponent(current, workInProgress, type, newProps);
      } else {
        // 没有 DOM 节点：创建
        const instance = createInstance(type, newProps);
        // 将所有子节点挂载到这个新创建的元素上
        appendAllChildren(instance, workInProgress);
        workInProgress.stateNode = instance;
      }
      // 设置最终属性
      finalizeInitialChildren(instance, type, newProps);
      break;
      
    case HostText:
      // 文本节点的创建/更新
      if (current !== null && workInProgress.stateNode !== null) {
        updateHostText(current, workInProgress, newProps);
      } else {
        workInProgress.stateNode = createTextNode(newProps);
      }
      break;
  }
  
  // 冒泡副作用到父节点
  bubbleProperties(workInProgress);
  return null;
}
```

**completeWork 的核心产出**：
- 创建/更新 DOM 节点的 stateNode
- 将子节点挂载到父节点上
- 通过 `bubbleProperties` 聚合副作用和更新信息到父节点
- 构建 effectList（将所有有副作用的节点串联）

### 3. workLoop——协调的驱动引擎

`workLoop` 连接了 beginWork、completeWork 和 Scheduler 的时间切片：

```typescript
// 源码简化：packages/react-reconciler/src/ReactFiberWorkLoop.js

// render 阶段的工作循环
function workLoopConcurrent() {
  // 循环条件：有工作要做，且时间预算充足
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

// 同步工作循环（不可中断）
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(unitOfWork: Fiber): void {
  // 1. beginWork：向下递
  const next = beginWork(current, unitOfWork, renderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  
  if (next === null) {
    // 没有子节点了，开始向上归
    completeUnitOfWork(unitOfWork);
  } else {
    // 有子节点，继续向下遍历
    workInProgress = next;
  }
}

function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  
  do {
    // 2. completeWork：当前节点归
    const next = completeWork(current, completedWork);
    
    // 如果有 sibling，返回 sibling 继续 processing
    if (next !== null) {
      workInProgress = next;
      return;
    }
    
    // 3. 向上回到父节点
    const returnFiber = completedWork.return;
    if (returnFiber === null) {
      workInProgress = null; // 到达 FiberRoot，render 阶段结束
    } else {
      completedWork = returnFiber;
    }
  } while (completedWork !== null);
}
```

### 4. 从 completeUnitOfWork 到 effectList

在 `completeUnitOfWork` 的"归"过程中，React 会构建 effectList：

```typescript
// 在 completeUnitOfWork 中
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  
  do {
    const returnFiber = completedWork.return;
    
    if (returnFiber !== null) {
      // 如果当前节点有副作用
      if (completedWork.flags !== NoFlags) {
        // 将当前节点添加到父节点的 effectList 中
        if (returnFiber.lastEffect === null) {
          returnFiber.firstEffect = completedWork;
        } else {
          returnFiber.lastEffect.nextEffect = completedWork;
        }
        returnFiber.lastEffect = completedWork;
      }
      
      // 将子树的副作用也冒泡到父节点
      if (completedWork.subtreeFlags !== NoFlags) {
        if (returnFiber.lastEffect === null) {
          returnFiber.firstEffect = completedWork.firstEffect; // 注意这里是取 completedWork 的 firstEffect
        } else {
          returnFiber.lastEffect.nextEffect = completedWork.firstEffect;
        }
        returnFiber.lastEffect = completedWork.lastEffect;
      }
    }
    
    // 处理 sibling 或回到父节点
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;
    }
    completedWork = returnFiber;
  } while (completedWork !== null);
}
```

渲染结束时，`fiberRoot.finishedWork` 上的 `firstEffect` 就是整个 effectList 的头节点。

## 实战案例（含完整代码）

### 案例：追踪 Fiber 渲染流程的调试工具

下面的代码不是生产组件，而是帮助你**在控制台观察** Fiber 工作流程的工具：

{% raw %}
```jsx
import React, { useState, useEffect, useRef } from 'react';

// 在组件挂载后，手动触发一次更新并追踪 Fiber 流程
function FiberFlowTracker() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('A');
  const containerRef = useRef(null);
  const [logs, setLogs] = useState([]);

  const addLog = (msg) => {
    setLogs(prev => [...prev.slice(-49), `[${new Date().toISOString().slice(11, 23)}] ${msg}`]);
  };

  useEffect(() => {
    addLog('组件挂载完成');
    addLog('Fiber 树结构:');
    // 通过 DOM 节点追踪对应的 Fiber
    if (containerRef.current) {
      const fiberKey = Object.keys(containerRef.current).find(
        k => k.startsWith('__reactFiber$')
      );
      if (fiberKey) {
        const fiber = containerRef.current[fiberKey];
        logFiberTree(fiber, 0);
      }
    }
  }, []);

  const handleUpdate = (type) => {
    addLog(`=== 触发更新: ${type} ===`);

    // 记录更新前的 Fiber 状态
    if (containerRef.current) {
      const fiberKey = Object.keys(containerRef.current).find(
        k => k.startsWith('__reactFiber$')
      );
      if (fiberKey) {
        const fiber = containerRef.current[fiberKey];
        addLog(`更新前 - 容器 Fiber lanes: ${fiber.lanes.toString(2).padStart(5, '0')}`);
        addLog(`更新前 - 容器 childLanes: ${fiber.childLanes.toString(2).padStart(5, '0')}`);
      }
    }

    if (type === 'count') {
      setCount(c => c + 1);
    } else if (type === 'text') {
      setText(t => t === 'A' ? 'B' : 'A');
    } else if (type === 'both') {
      setCount(c => c + 1);
      setText(t => t === 'A' ? 'B' : 'A');
    }
  };

  // 当 DOM 更新后，记录新的 Fiber 状态
  useEffect(() => {
    if (count > 0 || text !== 'A') {
      // 使用微任务等待 commit 完成
      queueMicrotask(() => {
        addLog('commit 完成');
        if (containerRef.current) {
          const fiberKey = Object.keys(containerRef.current).find(
            k => k.startsWith('__reactFiber$')
          );
          if (fiberKey) {
            const fiber = containerRef.current[fiberKey];
            addLog(`更新后 - 容器 Fiber lanes: ${fiber.lanes.toString(2).padStart(5, '0')}`);
            // 检查 alternate
            if (fiber.alternate) {
              addLog('✅ 双缓冲已建立: 当前节点有 alternate 引用');
            }
          }
        }
      });
    }
  }, [count, text]);

  // 递归输出 Fiber 树结构
  const logFiberTree = (fiber, depth) => {
    if (!fiber) return;
    const indent = '  '.repeat(depth);
    const tagNames = ['HostRoot', 'HostComponent', 'HostText', 'FunctionComponent', 'ClassComponent'];
    const tag = tagNames[fiber.tag] || `Tag(${fiber.tag})`;
    const info = fiber.memoizedProps 
      ? (fiber.memoizedProps.className || fiber.memoizedProps.id || '')
      : '';
    addLog(`${indent}${tag} ${info || fiber.type || ''}`);
    
    if (fiber.child) logFiberTree(fiber.child, depth + 1);
    if (fiber.sibling) logFiberTree(fiber.sibling, depth + (fiber.child ? 1 : 0));
  };

  return (
    <div ref={containerRef} className="fiber-tracker">
      <h3>🔄 Fiber 渲染流程观察器</h3>
      
      <div style={{ display: 'flex', gap: 8, marginBottom: 12 }}>
        <button onClick={() => handleUpdate('count')}>更新 count</button>
        <button onClick={() => handleUpdate('text')}>更新 text</button>
        <button onClick={() => handleUpdate('both')}>同时更新</button>
        <button onClick={() => setLogs([])}>清空日志</button>
      </div>

      <div style={{ display: 'flex', gap: 16, marginBottom: 12 }}>
        <div>count: <strong>{count}</strong></div>
        <div>text: <strong>{text}</strong></div>
      </div>

      <div style={{
        border: '1px solid #ccc',
        borderRadius: 8,
        padding: 8,
        background: '#1e1e1e',
        color: '#d4d4d4',
        fontSize: 12,
        fontFamily: 'monospace',
        maxHeight: 400,
        overflow: 'auto',
      }}>
        {logs.map((log, i) => (
          <div key={i} style={{
            padding: '1px 4px',
            background: log.startsWith('===') ? '#2d2d2d' : 'transparent',
            fontWeight: log.startsWith('===') ? 'bold' : 'normal',
          }}>
            {log}
          </div>
        ))}
        {logs.length === 0 && <div style={{ color: '#666' }}>等待触发更新...</div>}
      </div>
    </div>
  );
}

export default FiberFlowTracker;
```
{% endraw %}

运行这个组件，在控制台观察每次更新时：
1. beginWork 开始前 Fiber 的 lanes 状态
2. commit 后 Fiber 树的 alternate 关系变化
3. 多次不同优先级更新时的调度行为差异

## 底层原理（含源码分析）

### 1. performConcurrentWorkOnRoot 的执行入口

整个 Fiber 工作流程从 `performConcurrentWorkOnRoot` 开始：

```typescript
// 源码简化：packages/react-reconciler/src/ReactFiberWorkLoop.js

function performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean) {
  // 确保这是当前最重要的渲染工作
  const originalCallbackNode = root.callbackNode;
  
  // 检查是否有过期任务（饥饿处理）
  const lanes = getNextLanes(root, NoLanes);
  if (lanes === NoLanes) return null;
  
  // 是否应该同步渲染：任务已过期或没有时间切片的必要
  const shouldTimeSlice = 
    !includesBlockingLane(root, lanes) &&
    !includesExpiredLane(root, lanes) &&
    !didTimeout;
  
  // 执行渲染
  const exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)   // 并发渲染，可中断
    : renderRootSync(root, lanes);         // 同步渲染，不可中断
  
  if (exitStatus !== RootIncomplete) {
    // render 阶段完成，进入 commit
    const finishedWork = root.current.alternate;
    root.finishedWork = finishedWork;
    root.finishedLanes = lanes;
    commitRoot(root);
  }
  
  // 如果还有未完成的工作，返回继续进度
  if (root.callbackNode === originalCallbackNode) {
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

### 2. renderRootConcurrent 的可中断循环

这是"可中断渲染"的核心——在时间切片内循环执行 workLoop：

```typescript
function renderRootConcurrent(root: FiberRoot, lanes: Lanes) {
  // ... 初始化 ...
  
  do {
    try {
      // 在工作循环中执行 beginWork → completeWork
      workLoopConcurrent();
      break;
    } catch (thrownValue) {
      // 错误边界处理或 Suspense 挂起
      if (thrownValue === SuspenseException) {
        // Suspense 抛出的 Promise → 挂起当前渲染
        // 清空 workInProgress，等待 Promise resolve 后重新开始
      } else {
        throw thrownValue;
      }
    }
  } while (true);
  
  // 检查工作状态
  if (workInProgress !== null) {
    // 还有未完成的工作（时间片到了）
    return RootIncomplete;
  }
  
  // 全部完成
  return RootCompleted;
}
```

### 3. commitRoot 的三个子阶段

commit 阶段不可中断，分为三个子阶段执行：

```typescript
function commitRootImpl(root: FiberRoot, ...) {
  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;
  
  // 如果没有任何副作用，快速返回
  if (finishedWork.subtreeFlags === NoFlags && 
      finishedWork.flags === NoFlags) {
    root.current = finishedWork;
    return;
  }
  
  // =====================
  // 第一阶段：Before Mutation
  // =====================
  // 调用 getSnapshotBeforeUpdate（class 组件）
  // 调度 useEffect（安排异步执行，不在此阶段执行）
  // 生命周期钩子：componentWillUnmount
  
  // =====================
  // 第二阶段：Mutation
  // =====================
  // 遍历 effectList
  let nextEffect = finishedWork.firstEffect;
  while (nextEffect !== null) {
    const flags = nextEffect.flags;
    
    if (flags & Placement) {
      // 插入或移动 DOM 节点
      commitPlacement(nextEffect);
    }
    if (flags & Update) {
      // 更新 DOM 属性
      commitUpdate(nextEffect);
    }
    if (flags & Deletion) {
      // 删除 DOM 节点
      commitDeletion(nextEffect);
    }
    
    nextEffect = nextEffect.nextEffect;
  }
  
  // 重置 fiberRoot.current
  root.current = finishedWork;
  
  // =====================
  // 第三阶段：Layout
  // =====================
  // 再次遍历 effectList
  // 调用 useLayoutEffect 的回调
  // 调用 componentDidMount / componentDidUpdate
  // 设置 refs
  
  // 安排 useEffect 的回调在浏览器绘制后异步执行
}
```

## 高频面试题解析

### Q1：React Fiber 的 render 阶段为什么是可中断的，而 commit 阶段不是？

**参考答案：**

这是由两个阶段的内部差异决定的：

**render 阶段可中断的理由：**
- render 阶段操作的是 workInProgress 树——一棵**用户看不到**的树
- 如果被中断，workInProgress 树是一个"未提交的草稿"，可以直接丢弃
- current 树（用户看到的 UI）始终保持不变，用户感知不到中断
- render 阶段的中间状态不会暴露给外部

**commit 阶段不可中断的理由：**
- commit 阶段操作的是**真实 DOM**，每一次 DOM 操作都在改变用户看到的 UI
- 如果在 DOM 操作中途中断，用户会看到"半更新"的 UI——部分元素更新、部分未更新
- DOM 的删除、插入、属性修改都不是可逆操作，中断后无法回滚到一致状态
- 许多组件的 lifecycle hooks（componentDidMount/Update）依赖 DOM 的完整状态

简单说：render 阶段修改的是内存中的"草稿"，commit 阶段修改的是屏幕上的"成稿"。草稿可以撕了重写，成稿必须一次搞定。

### Q2：React 怎么知道要跳过某些子树的渲染（bailout）？

**参考答案：**

bailout 的判定在 beginWork 中执行：

```typescript
// beginWork 中的 bailout 判定逻辑
if (current !== null) {
  // 条件 1：props 没有变化（浅比较）
  if (oldProps === newProps) {
    // 条件 2：没有 context 变化
    if (!hasContextChanged()) {
      // 条件 3：渲染 lanes 不包含子树的 childLanes
      if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
        // 三个条件都满足 → bailout
        return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
      }
    }
  }
}
```

三个条件缺一不可：
1. 该节点的 props 引用没有变（`===` 浅比较）
2. context 没有变化
3. 该节点的子树上没有任何更新（childLanes 为空）

bailout 时，React 会调用 `bailoutOnAlreadyFinishedWork`，它不会创建新的子 Fiber，而是**直接复用 current 树的子结构**。这通常意味着几百到几千个子组件无需重新渲染。

bailout 是 React 性能优化的核心机制——通过 `React.memo`、`useMemo`、`useCallback` 保持 props 引用稳定性，能大幅增加 bailout 的机会。

### Q3：从触发 setState 到 DOM 更新，完整的 Fiber 事件循环是怎样的？

**参考答案：**

```
1. setState/dispatch 触发
   → 创建 Update 对象，标记 lane 到 fiber 及其父链
   → 调度 root：scheduleUpdateOnFiber(fiber, lane)

2. scheduleUpdateOnFiber
   → 确保 fiberRoot 被调度
   → performConcurrentWorkOnRoot 或 performSyncWorkOnRoot

3. renderRootConcurrent/renderRootSync
   → 创建 workInProgress 树（或复用 alternate）
   → 执行 workLoop（beginWork → completeWork 循环）
   → 构建 effectList
   → render 阶段结束

4. render 阶段结束判断
   → RootCompleted：进入 commit
   → RootIncomplete：时间片用完，等待下一帧

5. commitRoot
   → before mutation：getSnapshotBeforeUpdate
   → mutation：执行 DOM 操作（Placement/Update/Deletion）  
   → 切换 fiberRoot.current
   → layout：useLayoutEffect，componentDidMount/Update

6. 浏览器绘制（由浏览器控制）
   → 用户看到新的 UI
   → useEffect 的回调在绘制后执行
```

## 总结与扩展

### 总结

React Fiber 的工作流程是一条清晰的"双阶段流水线"：

- **render 阶段**（可中断）：beginWork（递）→ completeWork（归）→ effectList
- **commit 阶段**（不可中断）：before mutation → mutation → layout

理解这两个阶段及它们的子流程，是深入理解 React 性能优化、Concurrent 模式、Suspense、和各类 Hooks 执行时机的**先决条件**。

### 扩展：Fiber 工作流的魔改实验

如果你想更深入地理解 Fiber，可以尝试在 React 源码中修改 `yieldInterval` 的值（默认 5ms），观察不同的时间片大小对渲染流畅度的影响：

```typescript
// packages/scheduler/src/forks/SchedulerHostConfig.default.js
// 调大时间片：UI 卡顿更明显，但渲染更快完成
// 调小时间片：UI 更流畅，但渲染完成更慢

forceFrameRate(10);  // 10ms 时间片（更流畅但更慢）
// 或
forceFrameRate(2);   // 2ms 时间片（极度流畅但极慢）
```

这种实验能让你直观感受到时间切片的核心权衡：**响应性 vs 吞吐量**。
