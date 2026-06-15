---
title: 口述：React Diff完整过程深度解析
date: 2026-07-24
categories: [前端核心, React]
tags: [前端, React, Fiber, Diff, 完整过程]
description: 用口述讲解的方式，结合流程图层次，深度解析React Diff的完整三轮遍历过程——从第一轮的线性扫描、第二轮的Map+lastPlacedIndex算法、到第三轮的删除清理，覆盖单节点与多节点的Diff全貌。
---

## 一句话概括

React Diff 的完整过程是一个"两阶段、三轮次"的协调过程：先通过单节点 Diff 处理单个子元素的变化，再通过多节点 Diff 的三轮遍历（线性扫描→Map匹配→删除残存）处理列表变化，其中第一轮以 O(n) 快速扫描、第二轮以 Map 哈希查找定位重排节点、第三轮清理残余，整体遵循"尽可能复用、不得已重建"的核心原则。

## 背景与意义

### 为什么需要理解完整的 Diff 过程？

在 React 开发中，如果你写过 `map` 渲染列表、做过条件渲染、或者经历过"输入框切换后内容消失"的 Bug——你就已经和 Diff 打过交道，只是不知道它的名字而已。

理解 Diff 完整过程的价值在于：

1. **精准定位性能瓶颈**：当你知道 Diff 第一轮遍历什么情况下会跳出、什么情况下会创建大量新 Fiber，你就能针对性地优化列表结构或使用 key
2. **理解 key 的重要性**：不只是一个"消除控制台警告"的任务——错误使用 key 会直接导致错误的节点复用或多余的 DOM 操作
3. **预测渲染行为**：当你有一个复杂的嵌套列表交互时，你能"心算"出 React 会如何 Diff，从而预测渲染结果
4. **面试核心考点**：React Diff 完整过程是大厂前端面试的必考题

### Diff 的全局流程

```
触发更新 → scheduleUpdateOnFiber
    ↓
render 阶段开始 → 创建 workInProgress 树
    ↓
父组件的 beginWork → reconcileChildren
    ↓
┌─────────────────────────────────────────────┐
│       reconcileChildren 的 Diff 入口           │
│                                             │
│  ├─ newChild 是单个元素？                    │
│  │   └─ reconcileSingleElement（单节点 Diff） │
│  ├─ newChild 是数组？                        │
│  │   └─ reconcileChildrenArray（多节点 Diff） │
│  ├─ newChild 是文本/数字？                    │
│  │   └─ reconcileSingleTextNode             │
│  └─ newChild 是 null/false？                 │
│      └─ 标记删除所有旧子节点                  │
└─────────────────────────────────────────────┘
    ↓
子节点 Fiber 创建/复用完成 → 继续 beginWork 遍历
    ↓
completeWork → 构建 effectList
    ↓
commit 阶段 → 执行 DOM 操作
```

## 概念与定义

### Diff 的触发时机

React 的 Diff 发生在 render 阶段的 `beginWork` 子流程的 `reconcileChildren` 函数中。具体来说，当 React 进入一个组件的 beginWork 时：

```typescript
function beginWork(current, workInProgress, renderLanes) {
  // ... 其他逻辑 ...
  
  switch (workInProgress.tag) {
    case FunctionComponent: {
      // 调用函数组件 → 获取 React Elements
      const nextChildren = renderWithHooks(current, workInProgress, Component, props);
      
      // 开始 Diff！对比旧的子 Fiber 和新的 React Elements
      reconcileChildren(current, workInProgress, nextChildren, renderLanes);
      break;
    }
    case HostComponent: {
      // DOM 元素的子节点
      const nextChildren = workInProgress.pendingProps.children;
      reconcileChildren(current, workInProgress, nextChildren, renderLanes);
      break;
    }
  }
}
```

### reconcileChildren 的分发逻辑

```typescript
function reconcileChildren(current, workInProgress, nextChildren, renderLanes) {
  if (current === null) {
    // 首次挂载：不需要 Diff，直接创建所有 Fiber
    workInProgress.child = mountChildFibers(workInProgress, null, nextChildren, renderLanes);
  } else {
    // 更新阶段：执行 Diff！
    workInProgress.child = reconcileChildFibers(workInProgress, current.child, nextChildren, renderLanes);
  }
}
```

关键理解：**首次挂载时没有 Diff**，所有子节点都是新建的。Diff 只在组件更新时执行。

### 单节点与多节点的分发入口

```typescript
function reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes) {
  // 判断新 child 的类型
  const isObject = typeof newChild === 'object' && newChild !== null;
  
  if (isObject) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        // 单节点 Diff
        return placeSingleChild(
          reconcileSingleElement(returnFiber, currentFirstChild, newChild, lanes),
        );
    }
  }
  
  if (typeof newChild === 'string' || typeof newChild === 'number') {
    // 文本节点 Diff
    return placeSingleChild(
          reconcileSingleTextNode(returnFiber, currentFirstChild, '' + newChild, lanes),
    );
  }
  
  if (isArray(newChild)) {
    // 多节点 Diff
    return reconcileChildrenArray(returnFiber, currentFirstChild, newChild, lanes);
  }
  
  // null / false / undefined → 删除所有旧子节点
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

## 核心知识点拆解

### 1. 单节点 Diff 的两步决策树

单节点 Diff 的口述思路（可以拿这个去面试现场画出来）：

```
场景：一个旧子 Fiber（currentFirstChild）VS 一个新的 React Element

第一步：旧 Fiber 是否存在？
├── 不存在 → 直接 createFiberFromElement，返回新 Fiber

第二步：存在 → 比较 key
├── key 不同 → deleteChild（标记删除）→ 继续检查旧的同级节点
│            → 如果有同级的，继续比较（走 while 循环）
│            → 直到：
│              ├─ 找到 key 匹配的 → 进入 type 比较
│              └─ 没有找到 → 创建新 Fiber
│
├── key 相同 → 进入第三步

第三步：比较 type
├── type 相同 → useFiber 复用！updateElement → 仅更新 props
└── type 不同 → deleteChild + createFiber → 必须重建
```

**特殊情况：从列表变成单元素**

```jsx
// 之前是列表：<div key="a"/>, <div key="b"/>, <div key="c"/>
// 变成单个：<div key="a"/>

// 注意：旧的 currentFirstChild 是 key="a" 的 fiber
// 它的 sibling 指向 key="b" 和 key="c" 的 fiber
// reconcileSingleElement 在复用 key="a" 的 fiber 后
// 会调用 deleteRemainingChildren 删除 key="b" 和 key="c"
```

### 2. 多节点 Diff 的三轮遍历

这是面试中最被高频考查的部分。以下是**口述模板**：

```
多节点 Diff 发生在 reconcileChildrenArray 函数中。

第一轮：从左到右线性扫描
  ├── 同时遍历旧 Fiber 链表和新 Element 数组
  ├── 调用 updateSlot 比较当前节点的 key
  │   ├── key 匹配 → 比较 type
  │   │   ├── type 匹配 → 复用！继续下一个
  │   │   └── type 不匹配 → 标记删除，创建新的
  │   └── key 不匹配 → 跳出第一轮！
  │
  ├── 如果所有旧节点用完 → 剩余新节点都是新增的
  ├── 如果所有新节点用完 → 删除所有剩余旧节点
  └── 如果同时有剩余 → 进入第二轮

第二轮：Hash Map 查找
  ├── 将剩余旧节点放入 Map（key → fiber）
  ├── 遍历剩余新节点
  │   ├── 在 Map 中查找 key
  │   │   ├── 找到 → 比较 type
  │   │   │   ├── type 匹配 → 复用
  │   │   │   └── type 不匹配 → 删除旧，创建新
  │   │   └── 没找到 → 创建新节点
  │   └── 判断是否需要移动（lastPlacedIndex）
  └── 第三轮：删除 Map 中所有未被匹配的旧节点
```

### 3. lastPlacedIndex 的手算方法

手动计算一个排序变化的移动次数：

```
旧索引顺序：0:A, 1:B, 2:C, 3:D, 4:E
新列表顺序：A, E, D, C, B

将新列表中的每个元素映射为它在旧列表中的索引：
A→0, E→4, D→3, C→2, B→1

现在计算的 lastPlacedIndex：

已处理: [] → lastPlacedIndex = 0
处理 A: A 的旧索引=0, 0>=0 → 不移
          lastPlacedIndex = 0

已处理: [A] → lastPlacedIndex = 0
处理 E: E 的旧索引=4, 4>=0 → 不移
          lastPlacedIndex = 4

已处理: [A, E] → lastPlacedIndex = 4
处理 D: D 的旧索引=3, 3<4 → 移动！
          lastPlacedIndex 不变 = 4

已处理: [A, E, D] → lastPlacedIndex = 4
处理 C: C 的旧索引=2, 2<4 → 移动！
          lastPlacedIndex 不变 = 4

已处理: [A, E, D, C] → lastPlacedIndex = 4
处理 B: B 的旧索引=1, 1<4 → 移动！
          lastPlacedIndex 不变 = 4

结果：不移 1 个（A），移动 3 个（D、C、B）
```

### 4. 三轮遍历的总体拓扑图

```
reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes)
│
├─ [第一轮] for(; oldFiber && newIdx < newChildren.length; newIdx++)
│   │
│   ├─ updateSlot → key 匹配，type 匹配 → 复用，继续 next
│   ├─ updateSlot → key 匹配，type 不匹配 → 标记删除+创建，继续 next
│   └─ updateSlot → key 不匹配 → break ← 跳出！！！
│
├─ [出口判断]
│   ├─ oldFiber === null → 剩余新增 → 创建 → return
│   ├─ newIdx === newChildren.length → 剩余旧节点 → 全删 → return
│   └─ 都有剩余 → 进入第二轮
│
├─ [第二轮] existingChildren = new Map(剩余旧节点)
│   for(; newIdx < newChildren.length; newIdx++)
│   │
│   ├─ updateFromMap → 在 Map 中查找 key
│   │   ├─ 找到 → type 匹配 → 复用
│   │   │            → lastPlacedIndex 判断是否需要 Placement
│   │   ├─ 找到且 type 不匹配 → 删除旧+创建新
│   │   └─ 没找到 → 创建新节点
│   │
│   └─ 连接 newFiber 到结果链表
│
└─ [第三轮] 遍历 Map → 删除剩余未被匹配的旧 fiber
```

## 最小示例

```jsx
import React, { useState } from 'react';

// 通过控制台观察 Diff 完整过程
function DiffFullProcessDemo() {
  const [list, setList] = useState([
    { id: 'a', text: 'A' },
    { id: 'b', text: 'B' },
    { id: 'c', text: 'C' },
    { id: 'd', text: 'D' },
    { id: 'e', text: 'E' },
  ]);

  // 预设多种列表变化场景
  const scenarios = {
    append: () => {
      setList(prev => [...prev, { id: 'f', text: 'F' }]);
    },
    prepend: () => {
      setList(prev => [{ id: 'z', text: 'Z' }, ...prev]);
    },
    remove_middle: () => {
      setList(prev => prev.filter(item => item.id !== 'c'));
    },
    swap: () => {
      setList(prev => {
        const newList = [...prev];
        // 把 C 移到最后
        const [c] = newList.splice(2, 1);
        newList.push(c);
        return newList;
      });
    },
    shuffle: () => {
      setList(prev => {
        // E, D, C, B, A
        return [...prev].reverse();
      });
    },
    new_in_middle: () => {
      setList(prev => {
        const newList = [...prev];
        newList.splice(2, 0, { id: 'x', text: 'X' });
        return newList;
      });
    },
  };

  const [activeScenario, setActiveScenario] = useState('append');
  const [round, setRound] = useState(0);

  const handleClick = (scenarioId) => {
    setActiveScenario(scenarioId);
    setRound(prev => prev + 1);
    scenarios[scenarioId]();
  };

  return (
    <div style={{ fontFamily: 'system-ui, sans-serif', maxWidth: 500 }}>
      <h3>🔄 React Diff 完整过程演示</h3>
      <p style={{ fontSize: 12, color: '#999' }}>
        当前轮次：{round} 次更新
      </p>

      <div style={{ display: 'flex', gap: 6, flexWrap: 'wrap', marginBottom: 12 }}>
        <button onClick={() => handleClick('append')} style={btnStyle}>末尾追加</button>
        <button onClick={() => handleClick('prepend')} style={btnStyle}>开头插入</button>
        <button onClick={() => handleClick('remove_middle')} style={btnStyle}>删除中间</button>
        <button onClick={() => handleClick('swap')} style={btnStyle}>移动末尾</button>
        <button onClick={() => handleClick('shuffle')} style={btnStyle}>反转顺序</button>
        <button onClick={() => handleClick('new_in_middle')} style={btnStyle}>中间插入</button>
      </div>

      <div style={{ border: '1px solid #e0e0e0', borderRadius: 8, padding: 16 }}>
        <ul style={{ padding: 0, listStyle: 'none' }}>
          {list.map(item => (
            <li key={item.id} style={{
              padding: '8px 12px',
              margin: '4px 0',
              background: '#e3f2fd',
              borderRadius: 6,
              border: '1px solid #bbdefb',
              display: 'flex',
              alignItems: 'center',
              gap: 8,
            }}>
              <span style={{
                background: '#1a73e8',
                color: '#fff',
                borderRadius: '50%',
                width: 24,
                height: 24,
                display: 'flex',
                alignItems: 'center',
                justifyContent: 'center',
                fontSize: 12,
                fontWeight: 'bold',
              }}>
                {item.id.toUpperCase()}
              </span>
              <span>Item {item.text}</span>
              <span style={{ marginLeft: 'auto', fontSize: 11, color: '#999' }}>
                key="{item.id}"
              </span>
            </li>
          ))}
        </ul>
      </div>

      <DiffPathAnalysis scenario={activeScenario} />
    </div>
  );
}

function DiffPathAnalysis({ scenario }) {
  const analysis = getAnalysis(scenario);
  return (
    <div style={{
      marginTop: 16,
      background: '#1e1e1e',
      color: '#d4d4d4',
      padding: 12,
      borderRadius: 8,
      fontFamily: 'monospace',
      fontSize: 11,
      lineHeight: 1.6,
      overflow: 'auto',
    }}>
      {analysis}
    </div>
  );
}

function getAnalysis(scenario) {
  switch (scenario) {
    case 'append':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：末尾追加 (A, B, C, D, E → A, B, C, D, E, F)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=A → 匹配，复用 ✅',
        '  newIdx=1: key=B → 匹配，复用 ✅',
        '  newIdx=2: key=C → 匹配，复用 ✅',
        '  newIdx=3: key=D → 匹配，复用 ✅',
        '  newIdx=4: key=E → 匹配，复用 ✅',
        '  所有旧节点处理完毕 (oldFiber === null)',
        '',
        '出口 → 进入剩余新增节点处理：',
        '  newIdx=5: key=F → 没有旧节点，创建新 Fiber ✨',
        '',
        'Diff 结果：5 个复用，1 个新增',
        '总 DOM 操作：0（新增 F 的 DOM 创建是必须的）',
      ].join('\n');
    case 'prepend':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：开头插入 (A, B, C, D, E → Z, A, B, C, D, E)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=Z vs old key=A → ❌ key 不匹配！',
        '  立即跳出第一轮',
        '',
        '剩余旧节点：[A, B, C, D, E]',
        '剩余新节点：[Z, A, B, C, D, E]',
        '',
        '第二轮：构建 Map {key:A→fiber, key:B→fiber, ...}',
        '  newIdx=0: key=Z → Map 中未找到 → 创建新 Fiber ✨',
        '  newIdx=1: key=A → Map 中找到 ✅ → 复用',
        '             oldIndex=0 >= lastPlacedIndex=0 → 不移 ✅',
        '             lastPlacedIndex = 0',
        '  newIdx=2: key=B → Map 中找到 → 复用',
        '             oldIndex=1 >= lastPlacedIndex=0 → 不移 ✅',
        '             lastPlacedIndex = 1',
        '  ...依次匹配 C, D, E...',
        '',
        '第三轮：Map 中无剩余，无需删除',
        '',
        'Diff 结果：5 个复用，1 个新增，0 个移动',
        '总 DOM 操作：1（新增 Z）',
      ].join('\n');
    case 'remove_middle':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：删除中间 (A, B, C, D, E → A, B, D, E)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=A → 匹配，复用 ✅',
        '  newIdx=1: key=B → 匹配，复用 ✅',
        '  newIdx=2: key=D vs old key=C → ❌ key 不匹配！',
        '  跳出第一轮',
        '',
        '剩余旧节点：[C, D, E]',
        '剩余新节点：[D, E]',
        '',
        '第二轮：构建 Map {key:C, key:D, key:E}',
        '  newIdx=2: key=D → Map 中找到 ✅ → 复用',
        '  newIdx=3: key=E → Map 中找到 ✅ → 复用',
        '',
        '第三轮：Map 中剩余 key:C → 标记删除 ❌',
        '',
        'Diff 结果：4 个复用（A,B,D,E），0 新增，0 移动，1 删除',
        '总 DOM 操作：1（删除 C 的 DOM 节点）',
      ].join('\n');
    case 'swap':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：把C移到最后 (A, B, C, D, E → A, B, D, E, C)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=A → 匹配，复用 ✅',
        '  newIdx=1: key=B → 匹配，复用 ✅',
        '  newIdx=2: key=D vs old key=C → ❌ key 不匹配！',
        '  跳出第一轮',
        '',
        '剩余旧节点：[C, D, E]',
        '剩余新节点：[D, E, C]',
        '',
        '第二轮：构建 Map {key:C, key:D, key:E}',
        '  lastPlacedIndex = 0',
        '  newIdx=2: key=D → Map中找到, oldIndex=1',
        '     oldIndex(1) >= lastPlacedIndex(0) → 不移 ✅',
        '     lastPlacedIndex = 1',
        '  newIdx=3: key=E → Map中找到, oldIndex=2',
        '     oldIndex(2) >= lastPlacedIndex(1) → 不移 ✅',
        '     lastPlacedIndex = 2',
        '  newIdx=4: key=C → Map中找到, oldIndex=0',
        '     oldIndex(0) < lastPlacedIndex(2) → 需要移动 🔄',
        '     标记 Placement',
        '',
        '第三轮：Map 中无剩余，无需删除',
        '',
        'Diff 结果：5 个全部复用，0 新增，1 个移动，0 删除',
        '总 DOM 操作：1（insertBefore 移动 C）',
      ].join('\n');
    case 'shuffle':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：反转顺序 (A, B, C, D, E → E, D, C, B, A)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=E vs old key=A → ❌ key 不匹配！',
        '  立即跳出第一轮',
        '',
        '剩余旧节点：[A, B, C, D, E]',
        '剩余新节点：[E, D, C, B, A]',
        '',
        '第二轮：构建 Map {key:A, key:B, key:C, key:D, key:E}',
        '  lastPlacedIndex = 0',
        '  newIdx=0: key=E → Map中找到, oldIndex=4',
        '     oldIndex(4) >= lastPlacedIndex(0) → 不移 ✅',
        '     lastPlacedIndex = 4',
        '  newIdx=1: key=D → Map中找到, oldIndex=3',
        '     oldIndex(3) < lastPlacedIndex(4) → 移动 🔄',
        '  newIdx=2: key=C → Map中找到, oldIndex=2 < 4 → 移动 🔄',
        '  newIdx=3: key=B → Map中找到, oldIndex=1 < 4 → 移动 🔄',
        '  newIdx=4: key=A → Map中找到, oldIndex=0 < 4 → 移动 🔄',
        '',
        '第三轮：Map 中无剩余',
        '',
        'Diff 结果：5 个全部复用，0 新增，4 个移动，0 删除',
        '总 DOM 操作：4（insertBefore 移动 D,C,B,A）',
      ].join('\n');
    case 'new_in_middle':
      return [
        '=== Diff 完整过程分析 ===',
        '',
        '📋 列表变化：中间插入 (A, B, C, D, E → A, B, X, C, D, E)',
        '',
        '第一轮遍历：',
        '  newIdx=0: key=A → 匹配，复用 ✅',
        '  newIdx=1: key=B → 匹配，复用 ✅',
        '  newIdx=2: key=X vs old key=C → ❌ key 不匹配！',
        '  跳出第一轮',
        '',
        '剩余旧节点：[C, D, E]',
        '剩余新节点：[X, C, D, E]',
        '',
        '第二轮：构建 Map {key:C, key:D, key:E}',
        '  lastPlacedIndex = 0',
        '  newIdx=2: key=X → Map中未找到 → 创建新 Fiber ✨',
        '  newIdx=3: key=C → Map中找到, oldIndex=0',
        '     oldIndex(0) >= lastPlacedIndex(0) → 不移 ✅',
        '     lastPlacedIndex = 0',
        '  newIdx=4: key=D → Map中找到, oldIndex=1 >= 0 → 不移 ✅',
        '     lastPlacedIndex = 1',
        '  newIdx=5: key=E → Map中找到, oldIndex=2 >= 1 → 不移 ✅',
        '',
        '第三轮：Map 中无剩余（C,D,E 都被匹配了）',
        '',
        'Diff 结果：5 个复用（A,B,C,D,E），1 个新增（X），0 移动',
        '总 DOM 操作：1（创建 X 的 DOM）',
      ].join('\n');
    default:
      return '';
  }
}

const btnStyle = {
  padding: '6px 12px',
  border: '1px solid #ccc',
  borderRadius: 6,
  background: '#fff',
  cursor: 'pointer',
  fontSize: 11,
};

export default DiffFullProcessDemo;
```

## 实战案例（含完整代码）

### 案例：Todo 看板三轮完整模拟器

一个模拟真实 TODO 应用场景的组件，展示每轮 Diff 的工作细节：

```jsx
import React, { useState, useRef, useEffect } from 'react';

const INITIAL_TODOS = [
  { id: 1, text: '买咖啡豆 ☕', done: false },
  { id: 2, text: '写周报 📝', done: true },
  { id: 3, text: '健身 💪', done: false },
  { id: 4, text: '看电影 🎬', done: false },
  { id: 5, text: '给妈妈打电话 📞', done: false },
];

function TodoDiffFullDemo() {
  const [todos, setTodos] = useState(INITIAL_TODOS);
  const [filter, setFilter] = useState('all');
  const [logs, setLogs] = useState([]);
  const [dragIdx, setDragIdx] = useState(null);
  const logRef = useRef(null);

  const addLog = (msg) => {
    setLogs(prev => [...prev.slice(-39), msg]);
  };

  // 每次 todos 变化时，分析 Diff 过程
  useEffect(() => {
    // 在微任务中分析，确保 DOM 已更新
    queueMicrotask(() => {
      addAnalysis();
    });
  }, [todos, filter]);

  const displayTodos = todos.filter(t => {
    if (filter === 'active') return !t.done;
    if (filter === 'completed') return t.done;
    return true;
  });

  const toggleDone = (id) => {
    setTodos(prev => prev.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    ));
    addLog(`切换完成状态: id=${id}`);
  };

  const addTodo = () => {
    const newId = Date.now();
    setTodos(prev => [...prev, { id: newId, text: `新任务 #${prev.length + 1}`, done: false }]);
    addLog(`新增任务: id=${newId}`);
  };

  const removeTodo = (id) => {
    const todo = todos.find(t => t.id === id);
    setTodos(prev => prev.filter(t => t.id !== id));
    addLog(`删除任务: id=${id} "${todo?.text}"`);
  };

  const shuffleTodos = () => {
    const shuffled = [...todos];
    for (let i = shuffled.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1));
      [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
    }
    setTodos(shuffled);
    addLog('随机排序 todo 列表');
  };

  const resetTodos = () => {
    setTodos(INITIAL_TODOS);
    setFilter('all');
    addLog('重置到初始状态');
  };

  // 拖拽排序
  const handleDragStart = (idx) => setDragIdx(idx);
  const handleDrop = (idx) => {
    if (dragIdx === null || dragIdx === idx) return;
    const newTodos = [...todos];
    const [moved] = newTodos.splice(dragIdx, 1);
    newTodos.splice(idx, 0, moved);
    setTodos(newTodos);
    addLog(`拖拽排序: [${dragIdx}] → [${idx}]`);
    setDragIdx(null);
  };

  const addAnalysis = () => {
    // 这是一个模拟分析，实际 Diff 在 React 内部
    // 这里我们展示概念性的分析步骤
    addLog('  ↪ 第一轮/第二轮/第三轮处理完成（见上方分类）');
  };

  return (
    <div style={{ fontFamily: 'system-ui, sans-serif', maxWidth: 600 }}>
      <h3>📋 Todo Diff 三轮遍历完整模拟</h3>

      <div style={{ display: 'flex', gap: 8, flexWrap: 'wrap', marginBottom: 12 }}>
        <button onClick={addTodo} style={btnStyle}>+ 新增</button>
        <button onClick={shuffleTodos} style={btnStyle}>🔀 随机排序</button>
        <button onClick={resetTodos} style={{ ...btnStyle, border: '1px solid #e74c3c', color: '#e74c3c' }}>
          ↺ 重置
        </button>
        <span style={{ marginLeft: 'auto' }}>
          {['all', 'active', 'completed'].map(f => (
            <button
              key={f}
              onClick={() => { setFilter(f); addLog(`切换筛选: ${f}`); }}
              style={{
                ...btnStyle,
                border: filter === f ? '2px solid #1a73e8' : '1px solid #ccc',
                background: filter === f ? '#e8f0fe' : '#fff',
              }}
            >
              {f === 'all' ? '全部' : f === 'active' ? '未完成' : '已完成'}
            </button>
          ))}
        </span>
      </div>

      <div style={{ border: '1px solid #e0e0e0', borderRadius: 8, padding: 12 }}>
        {displayTodos.map((todo, idx) => (
          <div
            key={todo.id}
            draggable
            onDragStart={() => handleDragStart(todos.indexOf(todo))}
            onDragOver={e => e.preventDefault()}
            onDrop={() => handleDrop(todos.indexOf(todo))}
            style={{
              padding: '8px 12px',
              margin: '4px 0',
              background: todo.done ? '#e8f5e9' : '#fff',
              borderRadius: 6,
              border: '1px solid #e0e0e0',
              display: 'flex',
              alignItems: 'center',
              gap: 8,
              cursor: 'grab',
            }}
          >
            <input
              type="checkbox"
              checked={todo.done}
              onChange={() => toggleDone(todo.id)}
              style={{ cursor: 'pointer' }}
            />
            <span style={{
              flex: 1,
              textDecoration: todo.done ? 'line-through' : 'none',
              color: todo.done ? '#999' : '#333',
            }}>
              {todo.text}
            </span>
            <span style={{ fontSize: 10, color: '#999' }}>
              key={todo.id}
            </span>
            <button
              onClick={() => removeTodo(todo.id)}
              style={{
                border: 'none',
                background: '#ffebee',
                color: '#e74c3c',
                borderRadius: 4,
                padding: '2px 8px',
                cursor: 'pointer',
                fontSize: 12,
              }}
            >
              ✕
            </button>
          </div>
        ))}
      </div>

      <div ref={logRef} style={{
        marginTop: 12,
        background: '#1e1e1e',
        color: '#d4d4d4',
        padding: 12,
        borderRadius: 8,
        fontFamily: 'monospace',
        fontSize: 11,
        maxHeight: 250,
        overflow: 'auto',
        lineHeight: 1.6,
      }}>
        <div style={{ color: '#888', marginBottom: 4 }}>
          # Diff 执行日志 (最新 40 条)
        </div>
        {logs.length === 0 && (
          <div style={{ color: '#666' }}>操作 Todo 查看 Diff 分析...</div>
        )}
        {logs.map((log, i) => (
          <div key={i} style={{
            color: log.startsWith('=') ? '#ffcc00' : log.startsWith('  ') ? '#888' : '#d4d4d4',
            fontWeight: log.startsWith('=') ? 'bold' : 'normal',
          }}>
            {log}
          </div>
        ))}
      </div>

      <div style={{
        marginTop: 12,
        background: '#f5f5f5',
        border: '1px solid #e0e0e0',
        borderRadius: 8,
        padding: 12,
        fontSize: 12,
      }}>
        <strong>📌 完整 Diff 过程分类：</strong>
        <div style={{ display: 'flex', gap: 16, marginTop: 8, flexWrap: 'wrap' }}>
          <div><span style={labelStyle}>第一轮</span> 线性扫描，key 匹配则复用</div>
          <div><span style={labelStyle}>第二轮</span> Map 哈希查找 + lastPlacedIndex</div>
          <div><span style={labelStyle}>第三轮</span> 删除未被匹配的旧节点</div>
        </div>
      </div>
    </div>
  );
}

const btnStyle = {
  padding: '6px 12px',
  border: '1px solid #ccc',
  borderRadius: 6,
  background: '#fff',
  cursor: 'pointer',
  fontSize: 12,
};

const labelStyle = {
  display: 'inline-block',
  background: '#1a73e8',
  color: '#fff',
  borderRadius: 4,
  padding: '1px 6px',
  fontSize: 10,
  fontWeight: 'bold',
};

export default TodoDiffFullDemo;
```

## 底层原理（含源码分析）

### 1. reconcileChildrenArray 的完整源码结构

```typescript
// 源码位置：packages/react-reconciler/src/ReactChildFiber.js
// 完整的 reconcileChildrenArray 函数骨架

function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<*>,
  lanes: Lanes,
): Fiber | null {
  // ---------- 初始化 ----------
  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;
  let oldFiber = currentFirstChild;
  let nextOldFiber = null;
  let newIdx = 0;
  
  // ---------- 第一轮：线性扫描 ----------
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    nextOldFiber = oldFiber.sibling;
    
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
    
    if (newFiber === null) {
      // key 不匹配 → 跳出
      if (previousNewFiber === null) {
        resultingFirstChild = null;
      }
      break;
    }
    
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    
    oldFiber = nextOldFiber;
  }
  
  // ---------- 出口判断 ----------
  if (oldFiber === null) {
    // 旧节点全部处理完，剩余新节点都是新增
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) continue;
      
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }
  
  if (newIdx === newChildren.length) {
    // 新节点全部处理完，剩余旧节点都要删除
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }
  
  // ---------- 第二轮：Map 匹配 ----------
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
  
  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber, newIdx, newChildren[newIdx], lanes,
    );
    
    if (newFiber !== null) {
      if (newFiber.alternate !== null) {
        // 从 Map 中找到并复用了旧节点
        const oldIndex = newFiber.alternate.index;
        
        if (oldIndex < lastPlacedIndex) {
          newFiber.flags |= Placement;
        } else {
          lastPlacedIndex = oldIndex;
        }
      }
      
      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }
  
  // ---------- 第三轮：删除未匹配 ----------
  existingChildren.forEach(child => deleteChild(returnFiber, child));
  
  return resultingFirstChild;
}
```

### 2. 关键辅助函数：Placement 的延迟执行

Placement flag 不会在 Diff 过程中立即执行 DOM 操作。DOM 操作通过 effectList 在 commit 阶段执行：

```typescript
// commit 阶段中的 Placement 处理
function commitPlacement(finishedWork: Fiber): void {
  // 找到最近的宿主节点（有 DOM stateNode 的最近祖先）
  const parentFiber = getHostParentFiber(finishedWork);
  
  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;
  
  if (parentFiber.tag === HostComponent) {
    parent = parentStateNode;
    isContainer = false;
  } else {
    parent = parentStateNode.containerInfo;
    isContainer = true;
  }
  
  // 找到插入位置（在新列表中该节点的前一个兄弟
  // 对应的 DOM 节点作为 insertBefore 的参考节点）
  const before = getHostSibling(finishedWork);
  
  if (isContainer) {
    insertOrAppendChild(parent, finishedWork.stateNode, before);
  } else {
    parent.insertBefore(finishedWork.stateNode, before);
  }
}
```

### 3. 首次挂载时的 mountChildFibers

在首次挂载时，React 调用的是 `mountChildFibers` 而不是 `reconcileChildFibers`。两者最核心的区别：

```typescript
// mountChildFibers 使用的 shouldTrackSideEffects 为 false
function mountChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  return reconcileChildFibersImpl(
    returnFiber,
    currentFirstChild,
    newChild,
    lanes,
    false,  // shouldTrackSideEffects = false
  );
}

// reconcileChildFibers 使用的 shouldTrackSideEffects 为 true
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
  return reconcileChildFibersImpl(
    returnFiber,
    currentFirstChild,
    newChild,
    lanes,
    true,  // shouldTrackSideEffects = true
  );
}
```

当 `shouldTrackSideEffects` 为 false 时，`deleteChild` 等函数不会记录副作用。因为首次挂载时不需要删除旧的 DOM 节点——根本没有旧的 DOM。

## 高频面试题解析

### Q1：请口述 React Diff 的完整过程

**参考答案：**

这是一个经常在面试中出现的问题。建议按以下层次组织回答：

**第一层：入口判断**
```
reconcileChildren 根据 newChild 的类型分发：
- 单元素 → reconcileSingleElement
- 数组 → reconcileChildrenArray
- 文本/数字 → reconcileSingleTextNode
- null/false → 删除所有旧子节点
```

**第二层：多节点 Diff 的三轮遍历**
```
第一轮：从左到右，逐个比较 key
  ├── key+type 都匹配 → 复用，继续下一个
  ├── key 匹配但 type 不匹配 → 删除旧，创建新的，继续下一个
  └── key 不匹配 → 立即跳出
  
退出时有三种情况：
  1. 旧节点用完 → 剩余新节点全部新增
  2. 新节点用完 → 剩余旧节点全部删除
  3. 双端都有剩余 → 进入第二轮

第二轮：Map + lastPlacedIndex
  1. 剩余旧节点 → 放入 Map<key, fiber>
  2. 遍历剩余新节点
     在 Map 中查找 key：找到且 type 匹配→复用；没找到→新建
     lastPlacedIndex 判断是否需要移动

第三轮：删除 Map 中未被匹配的旧节点
```

**第三层：核心思想**
```
- 尽可能复用已有 DOM 节点
- 移动优于删除重建
- 最少 DOM 操作原则
```

### Q2：没有 key 的列表为什么会性能差？可能出现什么 Bug？

**参考答案：**

没有 key 时，React 在 Diff 第二轮使用索引（index）作为默认 key。这会导致两个问题：

**性能问题：**
```jsx
// 旧列表：[A, B, C, D, E]
// 新列表：[Z, A, B, C, D, E]

// 无 key 时：
// 第一轮：Z(0) vs A → key 不匹配（都是 null 所以key相同，但类型看看）...
// 实际上无 key 时 key 都是 null，所以 key 比较永远通过
// → type 比较：A(div) vs Z(div) → 类型相同→被错误复用
//    React 把旧 A 节点替换成了 Z 的内容
// → 旧 B 节点替换成了 A...
// → 全部被重新渲染（不是复用，是换成内容）
// 最终：E 节点被删除，遍历完
// 性能结果：5 次"伪复用"（实际上 prop 全变了，相当于重建）+
//           1 次新增 + 1 次删除
```

**Bug：输入框状态错乱**
```jsx
// 最常见的 Bug：带输入框的列表
{todos.map((todo, index) => (
  <InputItem key={index} todo={todo} />
))}

// 删除第一个 todo 后：
// 旧：index=0(A), 1(B), 2(C)
// 新：index=0(B), 1(C)
// → index=0 被复用（之前是 A，现在本应是 B）
//   但 Fiber 复用了 A 的实例
//   → 如果 InputItem 内部有 input 且内容由 state 维护，
//      输入框的内容不会更新！
```

### Q3：React Diff 和 Vue 3 Diff 在处理列表更新时有什么区别？

**参考答案：**

| 对比维度 | React Diff | Vue 3 Diff |
|---------|-----------|-----------|
| 列表数据结构 | Fiber 单向链表 | 普通数组 |
| 第一轮策略 | 单向扫描（从左到右） | 双端扫描（两端向中间） |
| 第二轮策略 | Map + lastPlacedIndex（贪心） | 最长递增子序列 LIS（精确） |
| 复杂度 | O(n) 最佳 / O(n) 最差 | O(n) 最佳 / O(n) 最差 |
| 移动操作数量 | 接近最优（可能多 1-2 个） | 严格最优 |
| key 的重要性 | 必须 | 必须 |

**最核心的区别**在于数据结构和遍历策略：

- React 使用 **Fiber 单向链表**，只能从头遍历到尾。这使得双端扫描难以实现（要从尾遍历必须反转链表或额外存储尾部引用）
- Vue 3 使用**数组**，天然支持双端操作，可以通过索引直接访问任意位置的元素

两种 Diff 算法在大多数业务场景下的性能差距很小。React 选择"单向扫描 + Map 兜底"的工程取舍，使其代码更简洁且易于维护。

## 总结与扩展

### 总结

React Diff 完整过程可以概括为一张决策树：

```
触发 update
  ├─ 首次挂载 → mountChildFibers（不需要 Diff，全新建）
  └─ 更新 → reconcileChildFibers
       ├─ 单个新子元素 → reconcileSingleElement（两步：key→type）
       └─ 数组新子元素 → reconcileChildrenArray（三轮遍历）
            ├─ 第一轮：线性扫描，key 匹配则复用
            ├─ 第二轮：Map 查找 + lastPlacedIndex
            └─ 第三轮：删除残存旧节点
```

### 扩展：如何通过理解 Diff 来优化 React 应用性能

1. **优先保证 key 的稳定性**：选择天然唯一的 ID（数据库 ID、UUID），避免使用 index 或随机数
2. **优先追加而非插入**：追加操作第一轮就能全部匹配，进入第二三轮的概率更低
3. **避免"先删后增"模式**：如果列表需要更新，尽量通过更新 props 而非卸载重建来实现
4. **考虑使用 composition pattern**：将频繁变化的列表封装在兄弟层级，使更高层的 Diff 更容易 bailout
5. **对于大型列表，使用虚拟滚动**：从根本上减少需要 Diff 的 DOM 节点数量
