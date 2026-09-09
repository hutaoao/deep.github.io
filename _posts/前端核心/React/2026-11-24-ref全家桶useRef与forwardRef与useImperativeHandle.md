---
layout: post
title: "ref 全家桶：useRef / forwardRef / useImperativeHandle"
date: 2026-11-24 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, useRef, forwardRef, useImperativeHandle, 回调ref, React19, TypeScript]
description: >
  面试向讲清 React 的 ref 三件套：useRef 到底是个什么盒子、改它为什么不触发渲染、它和 useState 怎么选；
  为什么自定义组件默认拿不到 ref，React 18 的 forwardRef 和 React 19 的 ref as prop 有什么区别；
  useImperativeHandle 怎么只暴露必要方法，以及回调 ref 与 React 19 新增的 ref 清理函数。
---

## 一句话概括

React 是声明式的：你描述 UI 长什么样，React 负责把 DOM 变过去。但总有几件事声明式做不了——让输入框聚焦、滚动到某个位置、播放视频、拿到上一次的值。这些是**命令式**动作，得有个口子让你直接伸手进 DOM 或者存一个"不该触发渲染"的变量。这个口子就是 ref。

面试问 ref，基本就三件事：

1. **`useRef` 是什么**——一个 `{ current }` 盒子，同一个对象贯穿组件一生，改它不触发渲染。
2. **怎么把 ref 递给子组件**——React 18 用 `forwardRef`，React 19 直接当普通 prop 传。
3. **怎么不把整个 DOM 暴露出去**——用 `useImperativeHandle` 只给父组件几个方法。

一句话先记住：**ref 是 React 的逃生舱，能写成 prop 的别写成 ref。**

## 核心知识点

### 1. useRef 的本质：一个跨渲染复用的盒子

```jsx
const ref = useRef(0);
// ref 就是一个 { current: 0 }，之后每次渲染 useRef 都返回同一个对象
```

只有首次渲染时 `initialValue` 生效，之后**这个参数被彻底忽略**。React 在 fiber 上给你存了这个对象，之后每次渲染原样还给你。

所以它有两个能力：

- **跨渲染存活**（普通局部变量每次渲染都重新创建，做不到）
- **改它不触发渲染**（React 根本不知道你改了，它就是个普通 JS 对象）

```jsx
function Counter() {
  const countRef = useRef(0);

  function handleClick() {
    countRef.current += 1;         // 改了
    alert(countRef.current);        // 值确实变了
  }

  return <button onClick={handleClick}>点我</button>;
  // 界面永远不会更新，因为没有任何东西告诉 React "该重新渲染了"
}
```

### 2. ref 还是 state？一张决策表

这是最高频的追问，别背概念，按这个顺序判断：

| 判断点 | 结论 |
| --- | --- |
| 这个值变化时，界面要不要跟着变？ | 要 → `useState`；不要 → `useRef` |
| 这个值会参与 JSX 渲染吗？ | 会 → `useState`（放 ref 里界面不更新） |
| 只是存定时器 ID、上一次的值、DOM 节点？ | → `useRef` |

```jsx
// ❌ 用 ref 存要显示的数据：点了没反应，因为不触发渲染
const [list] = useState([]);
const keywordRef = useRef('');
return <div>{keywordRef.current}</div>;   // 永远是空字符串

// ✅ 要显示就用 state
const [keyword, setKeyword] = useState('');
return <div>{keyword}</div>;
```

### 3. 铁律：不要在渲染期读写 ref.current

官方文档原话是 "Do not write *or read* `ref.current` during rendering"。原因是 React 要求组件函数像纯函数：同样的 props/state/context 必须返回同样的 JSX。渲染期读写 ref 会破坏这个前提，在并发特性和 StrictMode 下行为不可预测。

```jsx
// ❌ 渲染期读写
function Bad() {
  myRef.current = 123;                 // 🚩 写
  return <h1>{otherRef.current}</h1>;  // 🚩 读
}

// ✅ 只在事件处理函数或 effect 里读写
function Good() {
  useEffect(() => { myRef.current = 123; });
  function handleClick() { doSomething(otherRef.current); }
  return <h1>标题</h1>;
}
```

唯一的例外是**惰性初始化**这个官方认可的模式：

```jsx
const playerRef = useRef(null);
if (playerRef.current === null) {
  playerRef.current = new VideoPlayer();  // ✅ 只跑一次，结果永远一样
}
```

顺带一个很容易踩的性能坑：`useRef(new VideoPlayer())` 里的 `new` **每次渲染都会执行**，只是结果被丢弃了。创建对象很贵时才用上面那个惰性写法。

### 4. ref 的三张面孔

**(1) DOM ref**——最常见，React 在 commit 阶段把真实 DOM 节点塞进 `current`，节点移除时置回 `null`。

```jsx
const inputRef = useRef(null);
return <input ref={inputRef} />;
// 注意：渲染期间 current 还是 null，DOM 节点要等 commit 之后才有
```

**(2) 可变值容器**——定时器 ID、上一轮的 props、防抖的 timer。

```jsx
const timerRef = useRef(null);
useEffect(() => {
  timerRef.current = setInterval(tick, 1000);
  return () => clearInterval(timerRef.current);   // 清理时要用同一个 ID
}, []);
```

**(3) 回调 ref**——传函数而不是对象。React 在节点挂载时用节点调用它，卸载时用 `null` 调用。适合"节点一出现就要立刻做事"的场景，比如测量尺寸（effect 里测会晚一帧）。

```jsx
const [h, setH] = useState(0);
// 用 useCallback 包一下，否则每次渲染函数引用都变，React 会反复 detach/attach
const measureRef = useCallback((node) => {
  if (node !== null) setH(node.getBoundingClientRect().height);
}, []);
return <div ref={measureRef}>高度是 {h}</div>;
```

**React 19 新增：回调 ref 可以返回清理函数**，不用再写 `if (node) ... else ...` 了。注意别返回箭头函数以外的值（比如 `ref={node => set.add(node)}` 会返回 Set，React 19 会把它当成清理函数报错）。

```jsx
<div
  ref={(node) => {
    const observer = new ResizeObserver(handler);
    observer.observe(node);
    return () => observer.disconnect();   // ✅ React 19
  }}
/>
```

### 5. 自定义组件为什么默认拿不到 ref

给原生标签加 `ref`，React 知道该挂哪个 DOM 节点。但给你自己的组件加 `ref`，React 不知道——这个组件可能渲染 10 个元素，也可能一个都不渲染。**React 18 及以前会直接把 `ref` 从 props 里剔除并警告**。

这就是 `forwardRef` 存在的唯一理由：它把渲染函数包一层，让它多收一个 `ref` 参数，由你自己决定往哪挂。

```jsx
// React 18：必须包 forwardRef
const MyInput = forwardRef(function MyInput(props, ref) {
  return <input {...props} ref={ref} />;
});
```

**React 19 起，`ref` 就是普通 prop，不用包了**。官方文档已经把 `forwardRef` 标注为 **Deprecated**："In React 19, `forwardRef` is no longer necessary. Pass `ref` as a prop instead." 它现在还能用（存量项目和组件库到处都是），官方也说未来版本会正式废弃，迁移可以用官方 codemod。

```jsx
// ✅ React 19：直接从 props 里解构
function MyInput({ ref, ...rest }) {
  return <input {...rest} ref={ref} />;
}
// 用法完全不变：<MyInput ref={inputRef} />
```

面试时一句话说明白：**变的只是"怎么拿到 ref"，`useImperativeHandle` 的用法一点没变。**

类组件不受影响——类组件的 ref 一直指向实例，从来不需要 `forwardRef`。

### 6. useImperativeHandle：只给父组件必要的几个方法

直接把 DOM 节点给父组件，等于把整个节点的所有权交出去了，父组件能改你的样式、能读你的内部结构。`useImperativeHandle` 让你自己定义"对外暴露什么"。

```jsx
function MyInput({ ref }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() { inputRef.current.focus(); },
    scrollIntoView() { inputRef.current.scrollIntoView(); },
  }), []);   // deps：createHandle 里用到的响应式值都要写进来

  return <input ref={inputRef} />;
}

// 父组件：只能调暴露出来的两个方法
ref.current.focus();
ref.current.style.opacity = 0.5;   // ❌ 拿不到 DOM 节点，报错
```

三个要点：

- 第二个参数是个工厂函数，返回什么父组件就拿到什么，可以是对象也可以是任何值。
- 第三个 deps 决定 handle 什么时候重建：不传 → **每次渲染都重建**（能拿到最新 state，但引用一直变）；传 `[]` → 只建一次，这时**里面绝不能引用 state**，否则父组件永远调到的是首帧的旧值。用到 state 就老实写进 deps，规则跟 `useMemo` 一样。
- 官方明确提醒**不要滥用**：能用 prop 表达的（`isOpen`、`open`）就别做成命令式 API（`{ open(), close() }`）。

### 7. TypeScript 怎么写（加分项）

```tsx
import { useRef, useImperativeHandle, type Ref } from 'react';

// DOM ref：初始值给 null，泛型写元素类型，读出来是 HTMLInputElement | null
const inputRef = useRef<HTMLInputElement>(null);

// 可变值：直接给初值，类型自动推，读出来不带 null
const timerRef = useRef<number | null>(null);

// React 19 组件 props 里声明 ref：用 Ref<T>，别用 RefObject<T>
type Handle = { focus: () => void };
function MyInput({ ref }: { ref?: Ref<Handle> }) {
  useImperativeHandle(ref, () => ({ focus() { /* ... */ } }), []);
  return <input />;
}
```

为什么用 `Ref<T>` 而不是 `RefObject<T>`：父组件可能传 ref 对象，也可能传回调函数，`Ref<T>` 是这几种的联合类型，包容性更好。

### 8. 常见坑速记

| 坑 | 表现 | 解法 |
| --- | --- | --- |
| 渲染期读 `ref.current` | 拿到 `null` 或过期值 | 挪到 effect / 事件里读 |
| `useRef(new X())` | 每次渲染都 new 一次 | 惰性初始化 |
| 回调 ref 没包 `useCallback` | 每次渲染都 detach/attach | `useCallback` 稳住引用 |
| `useImperativeHandle` 漏 deps | 父组件调到旧 state | 把用到的响应式值写进 deps |
| 给自定义组件传 ref 报 null | React 18 没包 `forwardRef` | 包一层，或升级写法用 ref as prop |

## 其实你每天都在用

- 页面进入自动聚焦搜索框：`inputRef.current.focus()`，最经典的 ref 用法。
- 表单校验失败滚动到第一个错误字段：`scrollIntoView({ behavior: 'smooth' })`，声明式写不出来。
- 弹窗点遮罩关闭：需要判断 `event.target` 是不是在内容区外，得拿到内容区的 DOM 节点。
- 视频播放/暂停：`<video>` 的 `play()` / `pause()` 是命令式 API，只能 ref 调。
- 清理定时器 / 取消请求：`AbortController` 和 `timeoutId` 存 ref，卸载时能用上同一个引用。
- 拿到上一次的 props：`const prev = useRef(); useEffect(() => { prev.current = value; })`。
- 组件库：Ant Design 的 `Form` 实例、`Table` 的 `scrollTo`，底层都是 `useImperativeHandle` 暴露的方法。
- 虚拟列表：测量每一项真实高度，靠的就是回调 ref。

## 常见误解（FAQ）

**❌ 误区1："`useRef` 和 `useState` 差不多，都能存值，用哪个都行。"**

差别在"改了会不会重新渲染"。`ref.current` 变了 React 完全不知情，界面不会更新；`setState` 会触发一次渲染。**要显示在界面上的数据放 ref 里，是一个新手经典 bug**：值明明变了但页面不动。

**❌ 误区2："渲染的时候读一下 `ref.current` 也没事，我试过能跑。"**

StrictMode 下组件函数会被调用两次、并发渲染下渲染可能被丢弃重来，渲染期读写 ref 的结果**不保证稳定**。官方明令禁止，唯一的例外是 `if (ref.current === null)` 这种惰性初始化。

**❌ 误区3："`forwardRef` 是转发 ref 的高级技巧，现在必须得会写。"**

React 19 里 `ref` 已经是普通 prop，官方文档直接把 `forwardRef` 标了 **Deprecated**（"no longer necessary"，未来版本废弃）。新代码直接 `function C({ ref })` 就行。但**存量项目和组件库里到处都是 `forwardRef`，你还是得看懂**——面试答出"React 19 之后不再需要，但旧代码要能读懂"是最到位的。

**❌ 误区4："`useImperativeHandle` 就是把 DOM 节点转发出去。"**

正好相反。它的价值是**不**转发整个节点：内部 DOM 结构保持私有，只暴露父组件真正需要的两三个方法。要是你想给父组件完整 DOM 节点，直接 `ref={ref}` 挂上去就行，根本不需要这个 Hook。

**❌ 误区5："`useImperativeHandle` 的 deps 随便写，反正差不多。"**

两个方向都会出事：写了 `[]` 但 handle 里引用了 state → 父组件拿到的**永远是首帧的旧值**；不传 deps → 每次渲染都重建 handle，里面要是持有事件监听之类，就会反复挂载。要么把用到的响应式值写进 deps，要么压根别在 handle 里引用 state（改用 ref 中转）。

**❌ 误区6："回调 ref 里返回个值没关系。"**

React 19 起，回调 ref 的返回值会被当成清理函数。写成 `ref={node => mySet.add(node)}`（`Set.add` 返回 Set 本身）会直接报错。要么加大括号不返回值，要么老实用 `if (node) ... else ...` 的老写法。

## 一句话总结

**`useRef` 是一个跨渲染不变、改了不触发渲染的 `{ current }` 盒子；要显示的数据用 state，要命令式操作的用 ref；自定义组件要接 ref，React 19 直接当 prop 解构（旧的 `forwardRef` 要能读懂）；不想暴露整个 DOM 就用 `useImperativeHandle` 只给几个方法——但凡是能用 prop 表达的，都别做成 ref。**
