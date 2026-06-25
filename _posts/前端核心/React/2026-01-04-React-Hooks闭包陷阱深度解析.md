---
layout: post
title: "React Hooks闭包陷阱深度解析"
date: 2026-01-04 00:00:00 +0800
categories: ["前端核心", "React"]
tags: ["React", "Hooks", "闭包", "useState", "useEffect", "useRef", "useCallback"]
---

## 一句话概括

React Hooks 每次渲染都是一次全新的函数调用，闭包捕获的是**当前渲染时的状态快照**而非最新值。空依赖让闭包"冻结"在初始值上，定时器/事件监听/异步回调读到旧值的根源全在这。解法三条：**函数式更新**、**正确声明依赖**、**useRef 穿透**。

```javascript
// ❌ 闭包陷阱：count 永远是 0
function Bug() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setInterval(() => {
      console.log(count);     // forever 0
      setCount(count + 1);    // forever 1
    }, 1000);
  }, []);
}

// ✅ 函数式更新：React 把最新值传给你
function Fix() {
  const [count, setCount] = useState(0);
  useEffect(() => {
    setInterval(() => setCount(c => c + 1), 1000);
  }, []);
}
```

## 核心知识一：为什么会有闭包陷阱

函数组件每次渲染都是一次全新的函数执行，每次产生一套全新的闭包，各捕获各的值。问题出在**你让老闭包活到了未来**：

```
Mount:         count=0   effect 执行 → 定时器回调捕获 count=0
用户点击:      count=1   新渲染，新闭包有 count=1，但定时器还在跑老回调
1000ms 后:     setInterval 触发 → 调用老闭包 → count 还是 0 ← 陷阱
```

核心原因就一句：`useEffect(fn, [])` → effect 只执行一次 → 回调永远是 mount 那次渲染的闭包。

## 核心知识二：函数式更新 — 绕开快照

```javascript
// ❌ 点击快，count 可能不准（读的是闭包里的旧值）
<button onClick={() => setCount(count + 1)}>+1</button>

// ✅ React 保证 c 是最新状态
<button onClick={() => setCount(c => c + 1)}>+1</button>
```

`setState(fn)` 不是从闭包里读，而是 **React 把当前最新状态传给你**。新状态依赖旧状态时必须用它，上面 `Fix` 组件就是标准写法。

## 核心知识三：依赖数组 — 让闭包重新"照相"

读到最新值做判断时，函数式更新不够用——你需要依赖数组让 effect 重来：

```javascript
function Search({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    let cancelled = false;
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(data => { if (!cancelled) setResults(data); });
    return () => { cancelled = true; };
  }, [query]); // query 变了 → 旧 effect 清理 → 新闭包捕获新 query

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}
```

**诀窍**：effect 里用了什么外部变量，就放进依赖。`react-hooks/exhaustive-deps` 规则帮你做这个检查。

## 核心知识四：useRef — 穿透闭包的"任意门"

想读到最新值却不想频繁重建 effect？**useRef**：

```javascript
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  const scrollYRef = useRef(scrollY);
  scrollYRef.current = scrollY; // 每次渲染同步

  useEffect(() => {
    const onScroll = () => console.log('最新位置:', scrollYRef.current);
    window.addEventListener('scroll', onScroll);
    return () => window.removeEventListener('scroll', onScroll);
  }, []); // 只绑定一次，但永远拿到最新 scrollY
}
```

精髓：闭包捕获的是 ref **对象**（引用不变），通过 `.current` 读值绕过了快照限制。代价是改 `.current` **不触发渲染**。

## 核心知识五：useCallback + useRef 黄金组合

`useCallback` 的经典矛盾：传给 `React.memo` 子组件想要引用稳定，又想读最新 state：

```javascript
function Parent() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count;

  // 引用永不变化，通过 ref 读最新值
  const handleClick = useCallback(() => {
    console.log('最新值:', countRef.current);
    setCount(c => c + 1);
  }, []);

  return <Child onClick={handleClick} />;
}

const Child = React.memo(({ onClick }) => {
  console.log('Child 只在必要时渲染');
  return <button onClick={onClick}>+1</button>;
});
```

**三解法速查：**

| 场景 | 解法 | 代价 |
|------|------|------|
| 只更新状态 | `setState(c => c + 1)` | 无 |
| 读最新值做计算 | 正确依赖 `[dep]` | effect 频繁重建 |
| 读最新值但不想重建 | `useRef` | 改动不触发渲染 |

## 其实你每天都在用

### 场景一：搜索防抖

```javascript
function SearchBox() {
  const timerRef = useRef(null);
  const handleChange = (e) => {
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => fetch(`/api?q=${e.target.value}`), 300);
  };
  return <input onChange={handleChange} />;
}
```

timer ID 不参与 UI，`useRef` 天生合适。

### 场景二：WebSocket 消息

```javascript
function ChatRoom({ roomId }) {
  const [messages, setMessages] = useState([]);
  const messagesRef = useRef(messages);
  messagesRef.current = messages;

  useEffect(() => {
    const ws = new WebSocket(`wss://chat/${roomId}`);
    ws.onmessage = (e) => {
      const msg = JSON.parse(e.data);
      setMessages([...messagesRef.current, msg]); // 永远基于最新列表
    };
    return () => ws.close();
  }, [roomId]);

  return <ul>{messages.map(m => <li key={m.id}>{m.text}</li>)}</ul>;
}
```

`onmessage` 绑定一次，但 `messagesRef` 永远拿到最新列表。

### 场景三：第三方库事件绑定

```javascript
function MapWidget({ onMarkerClick }) {
  const cbRef = useRef(onMarkerClick);
  cbRef.current = onMarkerClick;

  useEffect(() => {
    const map = new ThirdPartyMap('#container');
    map.on('click', marker => cbRef.current(marker)); // 永远调最新回调
    return () => map.destroy();
  }, []);

  return <div id="container" />;
}
```

地图库初始化一次，但回调每次渲染都是新的——`useRef` 让老监听器能调到最新回调。

### 场景四：effect 清理的闭包隔离

```javascript
function DataFetcher({ userId }) {
  useEffect(() => {
    let cancelled = false;
    fetchUser(userId).then(u => { if (!cancelled) setUser(u); });
    return () => { cancelled = true; };
  }, [userId]);
}
```

`userId` 变化 → 执行上次清理（那次 `cancelled` 变 true）→ 再执行新 effect。闭包隔离的正确用法。

## 常见误解 FAQ

**❌ 误区一："useEffect 加空依赖就一定有问题"**

空依赖本身不是错，错的是**里面读了需要最新值的外部变量**。如果只用函数式更新和 ref，空依赖完全正确：

```javascript
// ✅ 没问题：只更新不读取
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**❌ 误区二："改了 ref.current 就会触发渲染"**

`ref.current = xxx` **永远不会触发渲染**——这是它和 `setState` 的根本区别。UI 要响应变化 → state；只是记个东西不重画 → ref。

**❌ 误区三："用 useCallback 包住所有函数就没闭包问题了"**

`useCallback` 本身就是闭包——依赖数组里的值会被"冻结"。空依赖内部读 state，读到的就是旧值：

```javascript
const handle = useCallback(() => console.log(count), []); // count 永远是初始值
```

**❌ 误区四："把依赖全加上就万事大吉"**

加依赖 = effect 每次变化都重建。定时器场景下 count 一变就销毁重开，低效且可能闪烁。选方案看场景，不能无脑加。

## 一句话总结

> React Hooks 闭包陷阱的本质是**每次渲染都是独立快照**，老闭包活到未来就看不到新值。解法按需选：只改状态用**函数式更新**，要读值用**正确依赖**，要读值但不想重建用 **useRef**。面试记住三个词：快照、闭包、ref 穿透。
