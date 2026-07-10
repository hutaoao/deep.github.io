---
layout: post
title: "useEffect实现原理"
date: 2026-03-08
tags: [React, useEffect, Hooks, Fiber]
categories: [前端核心, React]
---

## 一句话概括

`useEffect` 是 React 声明式副作用管理的核心——把副作用函数和依赖数组挂在 Fiber 的 effect 链表上，在渲染提交后异步执行、commit 前通过依赖比对（`Object.is`）跳过无关更新、并在下次 effect 执行前或组件卸载时自动清理（cleanup）。

## 核心知识点

### 1. useEffect 的执行时机

```
渲染阶段（render） → 提交阶段（commit） → DOM 更新 → 清理上次 effect → 执行本次 effect

具体：浏览器绘制之后，异步执行 effect，不阻塞用户交互
useLayoutEffect 则同步（commit 阶段立即执行，阻塞绘制）
```

```javascript
useEffect(() => {
  console.log('effect: 浏览器绘制后异步执行')
  return () => console.log('cleanup: 下次effect前或卸载时')
}, [dep])

useLayoutEffect(() => {
  console.log('layoutEffect: commit阶段同步执行，阻塞绘制')
}, [dep])
```

### 2. 依赖数组如何比对（Object.is）

React 用 `Object.is` 逐个比对新旧依赖数组的元素，如果一个元素不一样就重新执行 effect。这和 `===` 几乎相同，除了 `NaN` 和 `-0`：

```javascript
Object.is(NaN, NaN)   // true  （=== 返回 false）
Object.is(-0, +0)      // false （=== 返回 true）

// React 内部的浅比较
function areHookInputsEqual(nextDeps, prevDeps) {
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) continue
    return false  // 不同 → 重新执行 effect
  }
  return true  // 全相同 → 跳过
}
```

**注意**：`[]` 作为依赖数组表示只在 mount/unmount 时执行；省略依赖数组（`useEffect(fn)`）表示每次渲染后都执行。

### 3. effect 链表与调度

React 渲染完成后，effect 按**Fiber 树的深度优先顺序**依次执行——先父组件的 cleanup 和 effect，再子组件的。effect 之间是串行的，这保证了副作用的执行顺序可预测。

```typescript
// Fiber 上的 effect 相关字段
interface Fiber {
  updateQueue: {
    lastEffect: Effect | null  // effect 环形链表
  }
}

interface Effect {
  tag: number          // 标记类型（Passive=useEffect, Layout=useLayoutEffect）
  create: () => (void | (() => void))  // 你传的函数
  destroy: (() => void) | undefined   // cleanup 函数
  deps: any[] | null
  next: Effect | null
}
```

### 4. useEffect 的简化实现

```javascript
const allEffects = []  // 全局待执行 effect 队列

function useEffect(create, deps) {
  const hook = getOrCreateHook()
  
  // 比对依赖决定是否重新执行
  const hasChanged = !deps || !hook.deps || deps.some((d, i) => !Object.is(d, hook.deps[i]))
  
  if (hasChanged) {
    hook.deps = deps
    allEffects.push({
      create,
      destroy: hook.destroy,  // 上次的 cleanup
    })
  }
}

// commit 阶段结束时调用
function flushEffects() {
  allEffects.forEach(effect => {
    effect.destroy?.()      // 先清理上一次
    effect.destroy = effect.create()  // 执行新 effect，保存 cleanup
  })
  allEffects.length = 0
}
```

### 5. useEffect vs useLayoutEffect vs useInsertionEffect

| Hook | 执行时机 | 阻塞渲染 | 适用场景 |
|------|---------|---------|---------|
| `useEffect` | commit 后异步 | 否 | 99% 的副作用 |
| `useLayoutEffect` | commit 阶段同步 | 是 | DOM 测量、避免闪烁 |
| `useInsertionEffect` | DOM 变更前同步 | 是 | CSS-in-JS 库注入样式 |

```javascript
// useLayoutEffect 经典场景：读取 DOM 尺寸后同步更新避免闪烁
useLayoutEffect(() => {
  const height = ref.current.offsetHeight
  setTooltipPosition(height)  // 在浏览器绘制前完成，用户看不到抖动
}, [])
```

## 「其实你每天都在用」

1. **数据请求**：`useEffect(() => { fetchData() }, [userId])`，userId 变化自动重新请求。
2. **事件监听**：`useEffect(() => { window.addEventListener('scroll', fn); return () => window.removeEventListener('scroll', fn) }, [])`。
3. **定时器**：`useEffect(() => { const id = setInterval(tick, 1000); return () => clearInterval(id) }, [])`，离开页面自动清理。
4. **DOM 操作**：直接操作 canvas、第三方图表库初始化的场景，在 useEffect 里 mount、cleanup 里 destroy。
5. **路由变化追踪**：`useEffect(() => { analytics.pageView(pathname) }, [pathname])`。

## 常见误解（FAQ）

**❌ 误区：useEffect 依赖数组不传和传 `[]` 效果相同。** 不传依赖 = 每次渲染都执行（因为没有依赖数组，React 不比对直接执行）；传 `[]` = 只在 mount 时执行一次（mount 时执行，后续依赖不变就跳过）。这是两个完全不同的行为。

**❌ 误区：useEffect 在「组件挂载后」执行。** 更准确的描述是「浏览器绘制之后异步执行」。React 官方称之为「提交阶段之后」。在 React 18 的并发模式下，effect 还可能因高优先级更新被打断而延迟执行。

**❌ 误区：依赖数组里的引用类型每次都是新的，所以永远会重新执行。** 这正是为什么需要用 `useMemo` / `useCallback` 来稳定引用。如果你不想 effect 每次跑，就要确保依赖数组的每个元素在 `Object.is` 意义下相同。

**❌ 误区：cleanup 函数只在组件卸载时执行。** cleanup 在**每次 effect 重新执行前**也会执行。`useEffect(() => { subscribe(id) ; return () => unsubscribe(id) }, [id])` 中，id 从 1 变成 2 时，会先用 id=1 执行 cleanup（取消旧订阅），再用 id=2 执行新 effect。

**❌ 误区：useEffect 可以用来计算派生状态。** `useEffect(() => { setFullName(first + ' ' + last) }, [first, last])` 会导致额外一次渲染。正确的做法是直接用 `const fullName = first + ' ' + last` 在渲染期间计算，或者用 `useMemo`。Effect 是给**副作用**（网络、DOM、订阅）用的，不是给数据转换用的。

## 一句话总结

useEffect = 把「什么时候做、做什么、怎么收尾」声明在一个函数里，React 在正确的时机按正确的顺序帮你跑——你不需要关心生命周期钩子的 coupling 问题，只需要声明依赖。
