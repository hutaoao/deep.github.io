---
layout: post
title: "口述：React Fiber工作流程"
date: 2026-03-16
tags: [React, Fiber, 工作流程, render, commit]
categories: [前端核心, React]
---

## 一句话概括

React Fiber 的渲染分为 render（构建 workInProgress 树，可中断）和 commit（DOM 生效，不可中断）两大阶段——把"计算该改什么"和"真正去改"分离开，是可中断渲染的核心设计。

## 核心流程

### 1. 两大阶段总览

```
setState → scheduleUpdateOnFiber()
                    ↓
        ┌─── Render 阶段（可中断）───┐
        │  workLoop:                 │
        │    beginWork()  向下递      │
        │    completeWork() 向上归    │
        │  → 产出 effectList         │
        └──────────┬────────────────┘
                  ↓
        ┌─── Commit 阶段（不可中断）──┐
        │  ① Before Mutation         │
        │  ② Mutation (DOM操作)      │
        │  ③ Layout (useLayoutEffect)│
        │  → root.current 指针交换    │
        └────────────────────────────┘
```

### 2. beginWork：向下递

判断当前节点是否需要更新：

```js
function beginWork(current, workInProgress, renderLanes) {
  if (current) {
    // 检查能否跳过（bailout）
    if (oldProps === newProps && !hasContextChanged()
        && !includesSomeLane(renderLanes, childLanes)) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }
  }

  // 按 tag 分发
  switch (workInProgress.tag) {
    case FunctionComponent: return updateFunctionComponent(...);
    case ClassComponent:    return updateClassComponent(...);
    case HostComponent:     return updateHostComponent(...);
  }
}
```

返回第一个子节点（向下深入），或 null（没有子节点，进入 completeWork）。

### 3. completeWork：向上归

```js
function completeWork(current, workInProgress) {
  switch (workInProgress.tag) {
    case HostComponent: {
      if (current && workInProgress.stateNode) {
        updateHostComponent(current, workInProgress); // 更新已有 DOM
      } else {
        const dom = createInstance(type, newProps);   // 创建新 DOM
        appendAllChildren(dom, workInProgress);       // 归约子 DOM
        workInProgress.stateNode = dom;
      }
      break;
    }
  }
  bubbleProperties(workInProgress); // 冒泡副作用到父节点
}
```

递归过程：从最深的叶子 DOM 开始创建，逐层向上归约——父节点的 `appendAllChildren` 会把自己的子 DOM 全部挂到自身上。

### 4. performUnitOfWork：一次处理一个节点

```js
function performUnitOfWork(unitOfWork) {
  const next = beginWork(current, unitOfWork); // 递

  if (next === null) {
    completeUnitOfWork(unitOfWork); // 无子节点，向上归
  } else {
    workInProgress = next; // 有子节点，继续向下
  }
}

function completeUnitOfWork(unitOfWork) {
  let completedWork = unitOfWork;
  do {
    completeWork(current, completedWork);
    const sibling = completedWork.sibling;
    if (sibling) {
      workInProgress = sibling; // 去处理兄弟
      return;
    }
    completedWork = completedWork.return; // 上溯父节点
  } while (completedWork);
}
```

**时间切片插在这个循环里**——`while (workInProgress && !shouldYield())`。

### 5. commitRoot：三个子阶段

```js
function commitRoot(root) {
  const finishedWork = root.finishedWork;

  // ① Before Mutation：getSnapshotBeforeUpdate、调度 useEffect
  // ② Mutation：遍历 effectList，执行 Placement / Update / Deletion
  let nextEffect = finishedWork.firstEffect;
  while (nextEffect) {
    if (flags & Placement)  commitPlacement(nextEffect);
    if (flags & Update)     commitUpdate(nextEffect);
    if (flags & Deletion)   commitDeletion(nextEffect);
    nextEffect = nextEffect.nextEffect;
  }
  // 交换指针
  root.current = finishedWork;

  // ③ Layout：useLayoutEffect、componentDidMount/Update、ref
}
```

commit 必须一口气干完——因为操作的是真实 DOM，中断会让用户看到半更新的页面。

## 「其实你每天都在用」

1. **setState 不立即更新 DOM**：它只是创建一个 Update 对象挂到 fiber 的 lane 上，真正的 DOM 变更在 commit 阶段统一生效。

2. **useEffect 在渲染后异步执行**：effect 的回调在 commit 阶段收集，浏览器绘制后通过 Scheduler 异步调用——不在 render 也不在 commit 中同步跑。

3. **useLayoutEffect 在 DOM 变更后、绘制前同步执行**：用来读取布局信息并同步修改 DOM，避免闪烁——比如测量元素尺寸后调整位置。

4. **shouldComponentUpdate 返回 false → bailout**：beginWork 阶段检测到 props/state 不变，直接跳过该组件及其子树。

5. **Profiler 中的灰色块**：Chrome Performance 面板里 React 渲染间的空白，是时间切片让出的空闲时间。

## 常见误解 (FAQ)

**❌ 误区 1："render 阶段就更新了 DOM"**

render 阶段只是构建 workInProgress 树和打副作用标记，DOM 的真实创建和修改在 commit 阶段。你能在 render 阶段看到 Fiber 树的 JS 对象变了，但页面没动。

**❌ 误区 2："componentDidMount 在 DOM 插入后、用户能看到前执行"**

对的。它和 `useLayoutEffect` 都在 commit 的 Layout 子阶段同步执行，在浏览器绘制之前。这也是为什么 CMD 里做 DOM 测量是安全的。

**❌ 误区 3："render 阶段被中断就要从头再来"**

不一定全丢弃。如果高优先级更新和未完成工作没有冲突（比如两个不同子树），React 理论上可以保留部分进度。实践中为了代码简单通常重建。

## 一句话总结

render 阶段是在草稿纸上画草图（可擦可改可弃），commit 阶段是用钢笔誊写到展览板（一笔定稿）——Fiber 精妙之处就在这两个阶段的分离。

