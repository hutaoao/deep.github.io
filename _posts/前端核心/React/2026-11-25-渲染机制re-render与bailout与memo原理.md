---
layout: post
title: "渲染机制：re-render / bailout / memo 原理"
date: 2026-11-25 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, 重渲染, bailout, memo, useMemo, fiber, 性能优化]
description: >
  面试向讲清 React 的渲染机制：re-render 到底做了什么（render ≠ 更新 DOM）、哪些操作会触发重渲染、
  React 内置的两层 bailout（同值 setState 跳过、fiber 层跳过整棵子树）、
  React.memo 的浅比较原理与它拦不住的三件事、以及比 memo 更根本的 state 下移与 children 提升。
---

## 一句话概括

很多人以为"组件重渲染 = 页面重绘"，其实差得远。**re-render 只是 React 重新调用了一遍你的组件函数，生成新的虚拟 DOM**；只有 diff 之后发现真的不一样，才会去动真实 DOM。所以重渲染本身没那么贵，贵的往往是"重渲染了本不该渲染的大列表"。

React 自己有两层"偷懒"机制（bailout）：值没变就不往下走。`memo` / `useMemo` / `useCallback` 都是给 React 递梯子，让它能早点判定"这次可以跳过"。

一句话先记住：**React 默认就会尽量少干活；`memo` 不是让 React 变快，而是帮它确认"这块真的不用重算"。**

## 核心知识点

### 1. 先分清 render 和 commit

一次更新分两个阶段：

- **render 阶段**：从根开始遍历 fiber 树，调用组件函数、算出新的虚拟 DOM、打上"要增删改"的标记。这个阶段可以被打断、重来（并发特性的基础）。
- **commit 阶段**：把标记一次性应用到真实 DOM，浏览器重绘。

关键推论：**组件函数被调用了 ≠ DOM 被改了。** 你在组件里 `console.log('render')`，看到它打印了 10 次，但 DevTools 的 Performance 面板里可能一次 DOM 变更都没有。面试时把这点说清楚，就已经和"背八股"的拉开差距了。

### 2. 只有三个东西能触发重渲染

| 触发源 | 说明 |
| --- | --- |
| 组件**自身** state 变化 | `setState` / `useReducer` dispatch |
| **父组件**重渲染了 | 子组件默认无条件跟着渲染，哪怕 props 一个字没变 |
| 组件订阅的 **context** 变了 | 和 `memo`、`props` 都无关，是独立通道 |

注意第二条：React 默认**不比较 props**。父组件渲染，子组件的组件函数就会重新执行一遍——这就是"我明明没改 props，子组件怎么一直在渲染"的根源。

```jsx
function Parent() {
  const [n, setN] = useState(0);
  return (
    <div>
      <button onClick={() => setN(n + 1)}>{n}</button>
      <Child name="固定值" />   {/* n 每次变，Child 都会被重新调用 */}
    </div>
  );
}
```

### 3. 内置 bailout 之一：setState 同值直接跳过

新值和当前值 `Object.is` 相等时，React **跳过这次组件及其子组件的渲染**。

```jsx
const [n, setN] = useState(0);
setN(0);            // ✅ 跳过
setN(n);            // ✅ 跳过（同一个引用）
setN((p) => p);     // ✅ 跳过（updater 返回原值）

const [u, setU] = useState({ name: 'A' });
setU({ name: 'A' });        // ❌ 新对象，引用不同，照常渲染
u.name = 'B'; setU(u);      // ❌ 直接改属性再传回去，引用没变，React 认为没变 → 界面不更新
```

官方文档还补了一句很重要的话："虽然 React 有时**仍然需要先调用一次你的组件**才能跳过子组件，但这不影响你的代码。"所以别指望"同值 setState 一定一次都不渲染"，它是优化不是契约。

### 4. 内置 bailout 之二：fiber 层跳过整棵子树

这是 `memo` 生效的底层机制，面试能讲出来很加分。render 阶段走到每个 fiber 都会执行 `beginWork`，里面大致判断：

1. `oldProps === newProps`（同一个引用）且 context 没变？
2. 这个 fiber 自己有没有待处理的更新（lane）？
3. 它的子树里有没有待处理的更新（`childLanes`）？

三条都满足 → 直接跳过这个节点**及其整棵子树**，复用上次的渲染结果。所以：

- 父组件 props 引用不变、自己没更新、子树也没更新 → 整棵子树被跳过。
- 只要子树里有任何一个组件有更新，父节点就不能"完全跳过"，但自己仍然可以**不重新执行函数**（部分 bailout），只是继续往下遍历。

**bailout 跳过的是"调用组件函数"，不是"遍历 fiber"。** React 仍然要走一遍树才能做出判断，只是走得很快。`memo` 的价值在于让判断更早发生在树的高处，从而省掉整棵子树的遍历。

### 5. memo：给父组件传下来的 props 加一道浅比较

```jsx
const Child = memo(function Child({ name }) {
  console.log('Child render');
  return <p>{name}</p>;
});
```

`memo` 返回**一个新的组件**（不改原组件），它会对**每一个 prop** 做 `Object.is` 浅比较：

- 基本类型比值：`3 === 3` ✅
- 对象/数组/函数比引用：`{} === {}` ❌

全部相等 → 跳过这次渲染。也可以传第二个参数自定义比较函数，但官方明确警告两件事：**必须比较包括函数在内的每一个 prop**（函数闭包里带着父组件的 state，漏比会导致子组件一直用旧值）；**别在比较函数里做深比较**，数据一复杂能把页面卡死。

还有一条官方 caveat 值得背下来：**`memo` 是性能优化，不是语义保证**（"memoization is a performance optimization, not a guarantee"）。不要依赖它来保证逻辑正确。

### 6. memo 拦不住的三件事

这是最高频的追问，务必答全：

```jsx
const MemoChild = memo(function Child({ name }) { /* ... */ });
```

1. **自己的 state 变了** → 照样渲染。`memo` 只管父组件传进来的 props。
2. **订阅的 context 变了** → 照样渲染。context 是独立于 props 的订阅通道，`memo` 管不着。
3. **props 里混进了新引用** → 浅比较失败，等于没包。

第三条是最常见的"我明明包了 memo 怎么没用"：

{% raw %}
```jsx
// ❌ 白包：style 和 data 是内联新对象，onClick 是内联新函数，三个 prop 每次渲染都换引用
<MemoChild style={{ color: 'red' }} onClick={() => doIt(id)} data={{ id }} />

// ✅ 稳住引用
const style = useMemo(() => ({ color: 'red' }), []);
const handleClick = useCallback(() => doIt(id), [id]);
const data = useMemo(() => ({ id }), [id]);
<MemoChild style={style} onClick={handleClick} data={data} />
```
{% endraw %}

### 7. 比 memo 更根本的两招：state 下移 + children 提升

`memo` 是"打补丁"，重构组件结构才是"治本"，而且**零成本**。

**state 下移**：把状态挪到真正需要它的那个小组件里，父组件就不用渲染了，下面所有兄弟组件自然被 bailout 掉。

```jsx
// ❌ count 放在 App 里，App 一渲染，整个大列表都跟着渲染
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      <BigList />   {/* 无辜陪跑 */}
    </>
  );
}

// ✅ 把 count 关进 Counter，App 完全不参与
function App() {
  return (
    <>
      <Counter />
      <BigList />   {/* 父组件没渲染，它直接被 bailout */}
    </>
  );
}
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

**children 提升**（也叫 content lifting）：如果 state 必须在上层，就把不依赖 state 的部分通过 `children` 传进去。`children` 这个 prop 的引用在父组件不渲染时是稳定的，天然能被跳过。

```jsx
function Counter({ children }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <button onClick={() => setCount(count + 1)}>{count}</button>
      {children}   {/* 引用没变，React 直接复用，一次都不渲染 */}
    </div>
  );
}

<Counter><BigList /></Counter>
```

### 8. 优化的正确顺序

面试官问"React 性能怎么优化"，按这个顺序答，比一上来就说 `memo` 高级得多：

1. **先让默认 bailout 生效**：状态放在合适的层级，别把所有 state 都堆在根组件。
2. **用 `children` / 拆分组件**隔离变化范围。
3. **再考虑 `memo`**：只给"渲染确实贵 + props 确实稳定"的组件包。
4. **配合 `useMemo` / `useCallback`** 稳住传给 memo 组件的引用。
5. **列表给稳定 `key`**，别用数组下标。
6. **真有性能问题再上**虚拟列表、`useTransition` 这类重型手段——先 profile，别拍脑袋。

补充一句：开了 **React Compiler**（2025 年 10 月已 GA）之后，上面 3、4 两步编译器会自动做，`memo` / `useMemo` / `useCallback` 基本可以删掉。

## 其实你每天都在用

- 输入框每敲一个字整页卡顿：state 提到了页面顶层，往下几百个组件全在陪跑——state 下移立竿见影。
- Tab 切换后表单内容丢了：组件被卸载又重建，用 `key` 控制或者把状态提到不被卸载的层级。
- 大表格勾选一次卡 300ms：行组件没包 `memo`，或者 `onSelect` 每次渲染都是新函数。
- 主题切换整个 App 重渲染：context value 每次是新对象，先 `useMemo` 稳住引用。
- 弹窗打开时背景列表还在渲染：把弹窗内容通过 `children` 传进去，或者用 `useTransition`。
- 搜索联想输入框卡顿：把列表更新标成 transition，输入保持流畅。
- `useMemo` 算了半天反而更慢：计算本身很便宜，缓存的开销比重新算还大——这就是过度优化。

## 常见误解（FAQ）

**❌ 误区1："组件重渲染了，页面就一定重绘了。"**

re-render 只是重新执行组件函数、生成新的虚拟 DOM。React 会 diff，没差异就不碰真实 DOM。真正贵的是"大组件的函数体被执行"和"diff 的规模"，不是浏览器重绘。

**❌ 误区2："父组件渲染，子组件只要 props 没变就不会渲染。"**

默认恰恰相反——**父组件渲染，子组件无条件跟着渲染**，React 不做 props 比较。想拦住必须 `memo`，或者让父组件压根不渲染（state 下移 / children 提升）。

**❌ 误区3："包了 `memo` 组件就不会重渲染了。"**

`memo` 拦不住三件事：自己的 state、订阅的 context、以及 props 里的新引用（inline 对象/数组/函数）。最后一条最常见，必须配 `useMemo` / `useCallback` 才有效。而且官方说了 `memo` 是性能提示不是保证，别用它做逻辑正确性依赖。

**❌ 误区4："`setState` 传同样的值，React 一定不会重新渲染我的组件。"**

大多数情况会跳过，但官方文档明确说"某些情况下 React 仍需要先调用你的组件再跳过子组件"。这是内部优化细节，**不要写依赖这个行为的代码**。

**❌ 误区5："`memo` 的浅比较是深比较，对象里字段一样就相等。"**

是 `Object.is` 逐 prop 比引用。`{ id: 1 } !== { id: 1 }`。反过来也成立：你直接改对象属性再传回去，引用没变，`memo` 会认为"没变"从而**跳过更新**，界面显示的是旧数据。

**❌ 误区6："把每个组件都包上 `memo`，性能一定更好。"**

`memo` 本身有成本：多一层组件、每次要比对所有 props。`useMemo` / `useCallback` 也要维护依赖数组、占用缓存。对渲染很便宜的组件（几个 DOM 节点）来说，这些成本可能比直接重渲染还高。**先 profile，再优化。**

## 一句话总结

**React 默认就会 bailout（同值 setState 跳过、props 与子树都没变就跳过整棵子树）；`memo` 只是给 props 加一道 `Object.is` 浅比较，拦不住自己的 state、context 和新引用；真正该先做的是把 state 放到合适的层级、用 `children` 隔离变化范围——组件结构对了，比包十个 `memo` 都管用。**
