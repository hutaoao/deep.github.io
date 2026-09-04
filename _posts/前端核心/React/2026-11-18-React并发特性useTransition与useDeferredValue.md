---
layout: post
title: "React 并发特性：useTransition 与 useDeferredValue"
date: 2026-11-18 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, 并发特性, useTransition, useDeferredValue, isPending, 渲染优先级]
description: >
  面试向讲清 React 并发：为什么一次昂贵渲染会卡住输入框、useTransition 与 useDeferredValue 分别该在什么时候用、
  isPending / isStale 怎么提示、与防抖节流的区别、React 19 异步 Action 的坑，
  以及「输入框千万别包进 startTransition」「useDeferredValue 不减少网络请求」等高频误区。
---

## 一句话概括

**React 18 之前，所有 `setState` 一律平等紧急。** 你在输入框里敲一个字，React 要同步把整个列表重新渲染完，浏览器才有空把这个字画到屏幕上——列表一万条，输入框就一卡一卡的，因为渲染是同步的、不可打断的。

并发特性干的事就一件：**给更新分级**。紧急的（输入、点击、拖拽）先走；不紧急的（过滤一万条、切换 Tab、跳页）放后台渲染，而且**渲染到一半可以被紧急更新打断、丢掉重来**。

`useTransition` 和 `useDeferredValue` 是这件事的两个入口，面试官最常问的就是：**"它俩什么区别？什么时候用哪个？"** 记住一句话就够——

> **能拿到 setState 就用 `useTransition`，只有值（比如从父组件传下来的 prop）就用 `useDeferredValue`。**

## 核心知识点

### 1. 先看没有并发时为什么会卡

```jsx
function App() {
  const [query, setQuery] = useState('');
  // query 一变，BigList 就要同步渲染 1 万条，期间浏览器什么都干不了
  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <BigList filter={query} />
    </>
  );
}
```

`setQuery` 触发的是一次**同步、不可中断**的渲染。React 17 及以前没有"让一让"的能力，主线程被渲染占满，下一个按键要排队。

常见的老办法是防抖：**等 300ms 不输入了再过滤**。问题是——防抖只是"推迟卡顿的时刻"，真到渲染那一帧照样卡；而且延迟是写死的，好机器白等 300ms，烂机器 300ms 也不够。

### 2. useTransition：把「这次 setState」标成低优先级

```jsx
{% raw %}
const [isPending, startTransition] = useTransition();

function handleChange(e) {
  const value = e.target.value;
  setQuery(value);                      // 紧急：输入框必须立刻回显
  startTransition(() => {
    setFilter(value);                   // 不紧急：可被后续按键打断
  });
}

return (
  <>
    <input value={query} onChange={handleChange} />
    {isPending && <span>更新中…</span>}
    <BigList filter={filter} />
  </>
);
{% endraw %}
```

要点：

- `useTransition()` 不接参数，返回 `[isPending, startTransition]`，**必须在组件顶层调用**。
- `startTransition(fn)` 是**立即同步执行** `fn` 的，不是 `setTimeout`。只有 `fn` 同步执行期间调度的更新才被标成 transition；你要在 `setTimeout` 里 setState，那次更新跟 transition 没关系。
- `isPending` 从调用 `startTransition` 那一刻变 `true`，一直保持到这次过渡彻底渲染完，用来显示"更新中"提示。
- **最经典的错**：把输入框自己的 state 也包进去。

```jsx
{% raw %}
// ❌ 输入框变卡了，完全违背初衷
startTransition(() => {
  setQuery(e.target.value);
});

// ✅ 输入在外面，重活在里面
setQuery(e.target.value);
startTransition(() => setFilter(e.target.value));
{% endraw %}
```

官方文档写得很明确：**transition 更新不能用来控制文本输入框**，因为输入必须是同步的。

**React 19 的异步 Action 坑（高频追问）**：`startTransition` 可以接 async 函数，但 `await` **之后**的 setState 不再被自动标记为 transition（React 拿不回 async context）：

```jsx
startTransition(async () => {
  await saveData();
  // ❌ 这个更新不再是 transition，会同步打断页面
  setPage('/about');
  // ✅ 得再包一层
  startTransition(() => setPage('/about'));
});
```

另外两个官方点名的限制：多个并行的 transition 目前会被 React 批处理在一起；transition 里的异步 Action **不保证执行顺序**，先发后到会覆盖新状态——真遇到乱序问题，用 `useActionState` 或现成的请求库，别手搓。

### 3. useDeferredValue：延迟一个「值」，不需要 setter

```jsx
{% raw %}
function SearchResults({ query }) {          // query 是父组件传的 prop，我改不了它的 setState
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;   // 自己算"过时了"

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      <BigList filter={deferredQuery} />
    </div>
  );
}
{% endraw %}
```

- 签名：`useDeferredValue(value, initialValue?)`。可选的第二个参数 `initialValue` 只在**首屏渲染**时用（不传的话首屏不延迟——因为此时没有"旧值"可以顶上）。这个参数在 React 19 已是稳定 API，19.2 的类型定义就是上面这个签名。
- 更新时它会渲染**两遍**：第一遍用旧值（先让输入框画出来），第二遍在后台用新值，后台那遍可打断——你又敲一个字，后台渲染直接丢掉从头再来。
- **必须配合 `memo`**。父组件用新 query 重渲染时，子组件拿到的 `deferredQuery` 还是旧值，`memo` 才能让它跳过这次渲染；不套 `memo`，它照样跟着渲染一遍，等于白做。
- `useDeferredValue` 没有 `isPending`，想要提示就自己比 `query !== deferredQuery`。

**另外一个高频误区**：

```jsx
{% raw %}
// ❌ 没用：expensiveCompute(query) 每一帧照跑，defer 的只是"用结果去渲染"这一步
const deferred = useDeferredValue(expensiveCompute(query));

// ✅ 先 defer 值，再把重计算绑在 deferred 值上
const deferredQuery = useDeferredValue(query);
const result = useMemo(() => expensiveCompute(deferredQuery), [deferredQuery]);
{% endraw %}
```

### 4. 怎么选：一张表

| 维度 | useTransition | useDeferredValue |
| --- | --- | --- |
| 延迟对象 | 一次状态更新（setState） | 一个值 |
| 前提 | 你得拿得到 setter | 任何值都行，包括 prop / context |
| 返回值 | `[isPending, startTransition]` | 延迟后的值 |
| 加载提示 | 内置 `isPending` | 手动比 `value !== deferredValue` |
| 典型位置 | 事件处理函数里 | 组件体里（尤其是子组件） |
| 官方推荐场景 | 导航、Tab 切换 | 父传子的大列表、图表 |

**面试口述版**：*"两个底层都是同一套 Lane 优先级调度，效果也基本一样，区别只在控制权在谁手里。`useTransition` 要求你持有 setState，好处是白送一个 `isPending`；`useDeferredValue` 只要拿到值就能用，所以父组件把 query 传下来、子组件自己想降级的场景只能用它。"*

### 5. 它们不是万能的：什么时候别用

这几个反问面试官很爱追：

- **瓶颈是纯计算（不是渲染）** → 用 `useMemo` 或丢给 Web Worker。并发只改变"什么时候渲染"，不改变计算成本。
- **数据只有几十条** → 加不加没区别，纯属增加心智负担。官方建议先用 Profiler 确认瓶颈再上。
- **请求太多想省流量** → 官方文档原话：`useDeferredValue` **本身不会减少网络请求**。它延迟的是"把结果显示出来"，不是"发请求"。省请求该用防抖 / AbortController / 请求库。

### 6. 和防抖、节流比好在哪

官方文档专门有一节讲这个，三条：

1. **不用拍脑袋定延迟**。设备快就几乎无感，设备慢就自然地"慢多少跟不上多少"。
2. **可打断**。防抖节流本质上还是阻塞渲染，只是把阻塞的时刻往后挪；并发渲染是真的能在中途让路。
3. **深度集成 React**。跟 `Suspense` 打通：后台渲染如果挂起了，用户看到的还是旧内容，不会闪一下 fallback。

## 其实你每天都在用

- **搜索框 + 大列表**：输入框秒回显，列表慢半拍，典型 `useTransition`。
- **Tab / 路由切换**：点完新页面数据还没到，旧页面先留着，配 `isPending` 显示"加载中"。
- **图表随滑块联动**：滑块跟手，图表用 `useDeferredValue` 降级。
- **后台管理筛选表单**：多个筛选项联动重算表格，把重算包进 transition。
- **富文本 / 大表格实时校验**：输入不卡，校验结果稍后出。
- **下一步/上一步向导**：翻页时新页数据加载期间保留上一页。
- **路由切换**：React Router 6.4+、Next.js App Router 内部已经把导航更新包进了 transition，这就是切路由通常不闪全局 loading 的原因。

## 常见误解（FAQ）

**❌ 误区1："用了并发特性，请求就变少了。"**
不变。每次按键的请求照发。`useDeferredValue` 延迟的是"渲染新结果"，网络请求一次不少（官方 caveat 明确写了）。想省请求得自己防抖或取消。

**❌ 误区2："把 setState 都包进 startTransition 更流畅。"**
把受控输入框的 state 包进去，输入框直接变迟钝——这是官方点名的禁用场景。只有"可以慢半拍"的更新才该进 transition。

**❌ 误区3："isPending 可以当空数据判断用。"**
`results.length === 0` 是"真的没数据"，`isPending` 是"正在算"。写成 `results.length === 0 && <Spinner />`，用户搜了个不存在的词会一直转圈。

**❌ 误区4："useDeferredValue 能取消掉昂贵的计算。"**
不能。它只推迟"用这个值去渲染"的时机，函数本身每帧照跑。要么用 `useMemo` 把计算绑到 deferred 值上，要么开 Worker。

**❌ 误区5："startTransition 是延迟执行，跟 setTimeout 差不多。"**
它是**立即同步执行**回调的，只是把回调里产生的更新标记为低优先级。回调外、`setTimeout` 里的更新跟它无关。

**❌ 误区6："async Action 里 await 之后的 setState 也自动是 transition。"**
不是。React 19 里 `await` 会丢掉 async context，`await` 后的更新会退化成普通（同步）更新，必须再包一层 `startTransition`。

## 一句话总结

**并发特性不是让渲染变快，而是让"重要的更新先画出来"——能拿到 setState 就用 `useTransition`（白送 `isPending`），只有值就用 `useDeferredValue`（记得套 `memo`），并且永远别把输入框的 state 包进去。**
