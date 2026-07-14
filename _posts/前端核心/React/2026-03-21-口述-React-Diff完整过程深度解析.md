---
layout: post
title: "口述：React Diff完整过程"
date: 2026-03-21
categories: ["前端核心", "React"]
tags: ["React", "Diff", "Fiber", "key", "面试"]
---

## 一句话概括

React Diff 是两次判断 + 三轮遍历的协作：先判断子元素是单个还是数组，单元素走两步（key → type），数组走三轮（线性扫描 → Map 匹配 → 删除残余），全程贯彻"能复用绝不重建"的最低 DOM 操作原则。

## 核心知识点

### 1. 入口分发：newChild 决定了走哪条路

```typescript
function reconcileChildren(workInProgress, newChild) {
  // 首次挂载没有 Diff，直接创建
  if (current === null) return mountChildFibers(...);

  // 更新阶段：根据 newChild 类型分发
  if (typeof newChild === 'object' && newChild.$$typeof)
    return reconcileSingleElement(...);    // <div/>  → 单节点 Diff
  if (typeof newChild === 'string')
    return reconcileSingleTextNode(...);   // "hello" → 文本 Diff
  if (Array.isArray(newChild))
    return reconcileChildrenArray(...);    // [<A/>,<B/>] → 多节点 Diff
  // null/false → 删除所有旧子节点
}
```

### 2. 单节点 Diff：key 过不了就删，type 过不了也删

```
旧 Fiber 存在？
  ├─ 不存在 → 直接创建新 Fiber
  └─ 存在
       ├─ key 不同 → deleteChild → 检查旧 Fiber 的下一个 sibling
       │              (继续循环，直到 key 匹配或用完旧节点)
       └─ key 相同
            ├─ type 相同 → useFiber! 复用，只更新 props
            └─ type 不同 → deleteChild + createFiber，必须重建
```

特别注意：key 匹配后如果旧 fiber 链表还有 sibling（之前是列表），要 `deleteRemainingChildren` 全删。

### 3. 多节点 Diff 三轮遍历（面试口述模板）

```
第一轮：从左到右逐个比 key
  for oldFiber 和 newElement 同时存在:
    key 匹配且 type 匹配 → 复用，继续
    key 匹配但 type 不匹配 → 删旧建新，继续
    key 不匹配 → break！（立刻跳出第一轮）

跳出后判断：
  oldFiber === null → 剩余新节点全部新增
  newIdx === newChildren.length → 剩余旧节点全部删除
  两者都有 → 进入第二轮

第二轮：Map + lastPlacedIndex
  ① 剩余旧节点 → new Map(key → fiber)
  ② 遍历剩余新节点:
     在 Map 中 key 查找
       → 找到且 type 匹配: 复用，lastPlacedIndex 判断是否移动
       → 找到但 type 不同: 删旧建新
       → 没找到: 新建

第三轮：Map 中剩余的旧节点 → 全部删除（打 Deletion flag）
```

### 4. 手算 lastPlacedIndex

```
旧: A(0) B(1) C(2) D(3) E(4)
新: A   E   D   C   B

lastPlacedIndex = 0
A: oldIndex=0 >= 0 → 不移, lastPlacedIndex=0
E: oldIndex=4 >= 0 → 不移, lastPlacedIndex=4
D: oldIndex=3 < 4  → 移动 🔄
C: oldIndex=2 < 4  → 移动 🔄
B: oldIndex=1 < 4  → 移动 🔄

不复用0，移动3个。如果 "E" 也不复用就是 0 复用全移动。
```

### 5. 首次挂载为什么不需要 Diff

```typescript
// mount 阶段 shouldTrackSideEffects = false
// 所有节点直接创建，不标记删除/移动 flag
// 因为根本没有旧 DOM 需要清理
function mountChildFibers(...) {
  return reconcileChildFibersImpl(..., false); // false = 不追踪副作用
}
```

## 其实你每天都在用

- **列表追加**：`[...items, newItem]` 追加到底部，第一轮全部匹配，直接跳出，O(n) 搞定，不会进入第二三轮
- **条件渲染切换**：`{show ? <A/> : <B/>}` → 单节点 Diff，key 相同 type 不同 → 删了重建
- **搜索过滤**：`items.filter(...)` 结果变小，第一轮匹配几个后新节点用完 → 剩余旧节点全删（不进入第二轮，走出口判断快车道）
- **key 是 ID vs key 是 index**：用数据库 ID 做 key，Map 能精确匹配到对应旧节点；用 index 做 key 删除中间项后所有后续项内容全错位

## 常见误解

- **❌ 误区：「Diff 每次都要走完三轮」** 不是。末尾追加、末尾删除这些场景第一轮就走到底了（oldFiber 或 newElement 用完），直接走出口判断跳过二三两轮。

- **❌ 误区：「单节点 Diff 只比较一个旧 fiber」** 错。如果 key 不匹配，React 会沿着 `oldFiber.sibling` 链表逐个比较，直到找到 key 匹配或链表走完。

- **❌ 误区：「Diff 结果直接影响 DOM」** 不。Diff 只打 flag（Placement/Deletion/Update），真正的 DOM 操作在 commit 阶段执行。两个阶段严格分离是 Fiber 架构的核心设计。

- **❌ 误区：「React 和 Vue 的 Diff 是一样的」** 本质不同。React 是单向链表 → 只支持从左到右扫描，Vue 3 是数组 → 支持双端比较（头头、尾尾、头尾、尾头四种匹配）。Vue 的双端比较在某些场景下更高效。

## 一句话总结

Diff 的本质是把"新老两棵树找最小差异"的 NP 难题降级为 O(n) 的工程近似解——通过三个假设（同级比较、type 不同即重建、key 标识）换取可接受的性能和开发体验。
