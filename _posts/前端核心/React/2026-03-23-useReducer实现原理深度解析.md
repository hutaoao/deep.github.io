---
layout: post
title: "useReducer实现原理"
date: 2026-03-23
categories: ["前端核心", "React"]
tags: ["React", "Hooks", "useReducer", "useState", "Reducer", "面试"]
---

## 一句话概括

`useReducer` 是 React 复杂状态管理的利器——它通过 reducer 纯函数集中管理所有状态转换，内部和 `useState` 共享同一套 Fiber 更新链路。事实上 `useState` 就是预设了 `basicStateReducer` 的 `useReducer` 封装。

## 核心知识点

### 1. useState = useReducer + basicStateReducer

```typescript
// React 源码中的 useState 就是这么实现的
function useState(initialState) {
  return useReducer(basicStateReducer, initialState);
}

function basicStateReducer(state, action) {
  // 如果 action 是函数 → 函数式更新 setState(prev => prev + 1)
  // 否则 → 直接替换 setState(newValue)
  return typeof action === 'function' ? action(state) : action;
}
```

这个事实意味着：学会了 useReducer，你就理解了 React 所有状态更新的底层机制。

### 2. useReducer 的三要素

```typescript
// reducer: (state, action) => newState  纯函数，不修改原 state
// dispatch: (action) => void             引用稳定，永远不变
// useReducer: (reducer, initialArg, init?) => [state, dispatch]

// 最小示例
const [state, dispatch] = useReducer(
  (state, { type, payload }) => {
    switch (type) {
      case 'ADD':    return { count: state.count + payload };
      case 'RESET':  return { count: 0 };
      default:       return state;
    }
  },
  { count: 0 }
);
dispatch({ type: 'ADD', payload: 5 }); // state.count = 5
```

### 3. dispatch 为什么引用稳定？

```typescript
// mountReducer 阶段（首次渲染）
function mountReducer(reducer, initialArg, init) {
  const hook = mountWorkInProgressHook();
  hook.memoizedState = init ? init(initialArg) : initialArg;

  // 创建更新队列
  const queue = { pending: null, dispatch: null, lastRenderedReducer: reducer };
  hook.queue = queue;

  // dispatch 是一次性 bind 的，后续 update 阶段复用同一个引用
  const dispatch = dispatchReducerAction.bind(null, currentlyRenderingFiber, queue);
  queue.dispatch = dispatch;

  return [hook.memoizedState, dispatch];
}
```

dispatch 在 mount 时创建，之后永远不变。所以**安全地把它传给子组件，不会触发重渲染**——这比用 useCallback 包 setState 更"底层免费"。

### 4. 多个 dispatch 调用是批处理的

```jsx
function handleClick() {
  dispatch({ type: 'increment' });
  dispatch({ type: 'increment' });
  dispatch({ type: 'increment' });
  // React 18 自动批量：组件只渲染一次，count 只 +3
}
```

React 18 在所有场景（事件处理、setTimeout、Promise）都自动批处理。React 17 只在合成事件中批处理。

### 5. 惰性初始化

```typescript
// 方式一：直接传初始值
useReducer(reducer, { count: 0 });

// 方式二：传惰性初始化函数（首次渲染时才执行一次）
function init(initialCount) {
  return { count: initialCount, timestamp: Date.now() };
}
const [state, dispatch] = useReducer(reducer, 0, init);
// init(0) 只在 mount 时调用，后续 re-render 不执行
```

当初始值需要从 localStorage 读或做复杂计算时，惰性初始化避免每次渲染都重复计算。

## 其实你每天都在用

- **购物车状态**：商品列表、总价、优惠券、选中状态——用多个 useState 管理太散，用 useReducer 一个 reducer 搞定所有操作
- **表单多字段**：`dispatch({ type: 'SET_FIELD', field: 'email', value })` 比多个 setState 清晰得多
- **撤销/重做**：reducer 是纯函数，天然支持 `{ past, present, future }` 三栈结构实现 undo/redo
- **useState 本身就是在用 useReducer**：`setCount(5)` 底层就是 `dispatch(5)`，`setCount(c => c + 1)` 就是 `dispatch(c => c + 1)`

## 常见误解

- **❌ 误区：「useReducer 就是微型 Redux」** 不完全。useReducer 管理的是组件局部状态，Redux 管理的是全局单一状态树。useReducer 没有中间件、没有 DevTools 时间旅行、没有跨组件共享能力。

- **❌ 误区：「useReducer 比 useState 性能更好」** 不会。它们底层走同一套链路。useReducer 的价值是代码组织（集中式状态转换），不是性能。

- **❌ 误区：「dispatch 是异步的」** dispatch 本身是同步调用的（立即把 update 加入队列），但状态更新和重渲染是异步批处理的。调用 dispatch 后立即读 state 拿到的是旧值。

- **❌ 误区：「reducer 里可以用 push/splice」** 绝对不行。reducer 必须是纯函数——返回新对象，不修改传入的 state。用 `[...arr, newItem]` 而非 `arr.push(newItem)`。或配合 Immer 的 `produce` 用可变写法。

## 一句话总结

`useState` 是 `useReducer` 的语法糖，`useReducer` 是 React 状态更新的底层抽象——当你发现组件的 setState 散落各处、状态转换逻辑交错时，就是该引入 reducer 的时刻。
