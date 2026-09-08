---
layout: post
title: "Context 原理与 useContext 性能陷阱"
date: 2026-11-23 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, Context, useContext, 重渲染, useMemo, 状态管理, 性能优化]
description: >
  面试向讲清 Context：它到底是"依赖注入"而不是状态管理、Provider 的值存在哪、useContext 是怎么向上查找并订阅的、
  value 用 Object.is 比较带来的两大性能陷阱（新对象 / 全体消费者重渲染）、
  四种优化手段（useMemo、拆 context、state 与 dispatch 分离、外部 store），以及 memo 为什么挡不住 context 更新。
---

## 一句话概括

Context 是 React 内置的**跨层级传值管道**，用来解决"props 一层层往下透传"（prop drilling）的问题。它本身**不管状态**——只是把某个值"广播"给下面所有需要它的组件。

面试考 Context，其实就考一个字：**边界**。谁能读到、值变了谁会重渲染、为什么 `memo` 挡不住它。大部分人只知道"全局传值很方便"，说不清"为什么我一改主题，整个 App 都在重渲染"。

一句话记住：**Context 是投递机制，不是存储机制；`value` 一变，所有消费者无条件全部重渲染。**

## 核心知识点

### 1. Context 不是状态管理

这是最先要摆正的概念。Context 只负责"把值送到任意深度的后代"，值的更新逻辑还是得靠 `useState` / `useReducer`，或者外部 store。

```jsx
const ThemeContext = createContext('light');   // 括号里是默认值，找不到 Provider 时才用它

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light'); // 状态在这
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}

function Button() {
  const theme = useContext(ThemeContext);      // 消费在这
  return <button className={theme}>按钮</button>;
}
```

两个细节面试常问：

- **默认值只在"上面一个 Provider 都没有"时生效。** 如果树上是 `<ThemeContext.Provider value={undefined}>`，那 `useContext` 拿到的就是 `undefined`，不是默认值。
- **`useContext` 只向上找，不找自己。** 在同一个组件里既渲染 Provider 又调 `useContext`，读不到自己刚放进去的值。

### 2. 原理：值存在 fiber 上，查找是向上遍历

不用背源码，记住这条链路就够回答"它怎么工作的"：

1. `createContext(default)` 创建一个 Context 对象，它有两个角色：`Provider` 组件 + 一个"值槽位"。
2. Provider 渲染时，React 把 `value` 记录在**这个 Provider 对应的 fiber 节点**上。
3. 组件调用 `useContext(Ctx)` 时，React 沿着 fiber 树**向上找最近的那个 Provider**，读它的值，同时把当前组件**登记为订阅者**。
4. `value` 变化时（用 `Object.is` 比较前后两次），React 把这个 Provider 下面所有订阅者标记为需要重渲染。

顺带一个很实际的排查经验：**Provider 和 Consumer 必须是同一个 Context 对象**（`===`）。monorepo / pnpm symlink 场景下如果打包出两份 Context 实例，值就传不过去——这是 Context "突然失效"最常见的原因。

### 3. 陷阱一：value 每次渲染都是新对象

React 用 `Object.is` 比较 `value`，也就是**比引用**。这点决定了下面这段代码的命运：

```jsx
// ❌ 每次 Provider 重渲染都重新创建对象，引用必变 → 所有消费者跟着重渲染
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = { user, setUser, login }; // 直接写在 value 里也一样，关键是"每次都是新对象"
  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}

// ✅ 用 useMemo 稳住引用：只有依赖项变了才换新对象
function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);  // setUser 本身引用稳定
  return <AppContext.Provider value={value}>{children}</AppContext.Provider>;
}
```

注意 `useMemo` 治的是"**Provider 因为别的原因重渲染，导致 value 白换一次引用**"。它治不了"值真的变了"——值真变了，消费者本来就该更新。

### 4. 陷阱二：一个字段变了，全体消费者重渲染

Context 没有"细粒度订阅"。React **不知道**你只用到了 value 里的哪个字段：

```jsx
// 只用了 user...
function UserProfile() {
  const { user } = useContext(AppContext);
  return <p>{user?.name}</p>;
}
// 只用了 theme...
function ThemeToggle() {
  const { theme } = useContext(AppContext);
  return <span>{theme}</span>;
}
// theme 一变，UserProfile 也重渲染；user 一变，ThemeToggle 也重渲染
```

对策是**按更新频率拆分 context**。最经典的拆法是"状态和动作分开"：

```jsx
const CountStateContext = createContext(0);
const CountDispatchContext = createContext(null);

function CountProvider({ children }) {
  const [count, setCount] = useState(0);
  return (
    <CountStateContext.Provider value={count}>
      <CountDispatchContext.Provider value={setCount}>
        {children}
      </CountDispatchContext.Provider>
    </CountStateContext.Provider>
  );
}

function AddButton() {
  const setCount = useContext(CountDispatchContext); // 只订阅 dispatch
  return <button onClick={() => setCount(c => c + 1)}>+1</button>; // count 变了它也不重渲染
}
```

因为 `setCount` 的引用永远稳定，`AddButton` 从头到尾只渲染一次。

### 5. 陷阱三：memo 挡不住 context 更新

这条是官方 caveat，面试说出来很能拉开差距：

```jsx
const MemoChild = memo(function Child() {
  const { theme } = useContext(ThemeContext);
  return <div>{theme}</div>;
});

// 即使 MemoChild 的 props 一个都没变，theme 变了它照样重渲染
```

原因很直接：**context 订阅是独立于 props 的一条更新通道。** `memo` 拦的是"props 没变就不渲染"，拦不住"我订阅的 context 变了"。想让它不渲染，只有一个办法——别让它订阅（把读取下沉/上提，见下条）。

### 6. 四种优化手段，按优先级用

| 手段 | 解决什么 | 代价 |
| --- | --- | --- |
| `useMemo` / `useCallback` 稳引用 | Provider 无关重渲染导致的 value 换新 | 要维护依赖数组 |
| 拆 context（按变化频率 / 状态与动作分离） | 一个字段变、全体陪跑 | Provider 嵌套变多 |
| 把读取下移到更深的叶子组件 | 中间层组件不必订阅 | 组件结构要调整 |
| 换外部 store（`useSyncExternalStore` / Zustand） | 高频更新 + 需要细粒度订阅 | 引入依赖 |

关于第三种"下移"，具体做法是：**中间的大组件别读 context，把读取包进一个很小的子组件**，大组件只负责渲染：

```jsx
// ❌ 沉重的列表组件自己读 context，每次 theme 变整个列表重渲染
function BigList() {
  const { theme } = useContext(ThemeContext);
  return <ul>{/* 上千行数据 */}</ul>;
}

// ✅ 只有小标签订阅，BigList 用 children 原样渲染
function BigList({ children }) { return <ul>{children}</ul>; }
function ThemeTag() { const { theme } = useContext(ThemeContext); return <span>{theme}</span>; }
```

### 7. 什么时候该换状态库（送分题）

Context 适合**低频变化、全局读取**的数据：主题、语言、登录用户信息、功能开关、配置。

不适合：输入框实时值、滚动位置、拖拽坐标、WebSocket 高频推送、大列表的逐项状态。这些一旦进 Context，每次变化都要广播给所有消费者，页面直接卡给你看。这时候用 Zustand / Jotai 这类支持"按 selector 订阅"的库，或者自己用 `useSyncExternalStore` 接一个外部 store。

顺带提一句 React 19：新增了 `use(Context)` 的写法，和 `useContext` 读到的值一样，但**可以在条件语句、try/catch、提前 return 之后调用**，写起来更灵活。

```jsx
// React 19：可以在条件里读
if (isAdmin) {
  const theme = use(ThemeContext);
}
```

## 其实你每天都在用

- 深色模式切换：最典型的低频全局值，`ThemeContext` 几乎是每个项目的标配。
- 中英文切换：所有文案组件都读 `I18nContext`，切换时整棵树本来就该重渲染。
- 登录用户信息：导航栏、头像、权限判断都要读，用 Context 比层层透传 props 干净得多。
- 路由库：`React Router` 的 `useNavigate` / `useLocation` 底层就是读它自己内部的 Context。
- 数据请求库：SWR、TanStack Query 都用 Context 下发全局配置（默认 fetcher、缓存时间）。
- 组件库：Ant Design 的 `ConfigProvider`、各种 `FormProvider` 都是 Context 的实际应用。
- 表单状态：Formik 那种把整个表单状态放进 Context 的用法，表单字段多了就要小心重渲染。

## 常见误解（FAQ）

**❌ 误区1："Context 可以替代 Redux / Zustand。"**

Context 是**投递**机制，不是状态管理器——它不管状态怎么变、不提供细粒度订阅、没有中间件和时间旅行。`useReducer + Context` 能凑合做简单全局状态，但高频更新场景下性能远不如带 selector 的库。

**❌ 误区2："外面套了 `memo`，context 变了子组件就不会重渲染。"**

挡不住。`memo` 只拦 props 变化，context 是一条独立的订阅通道；只要组件调了 `useContext`，值一变它就得更新。官方 caveat 原话就是如此。

**❌ 误区3："把 value 用 `useMemo` 包起来，消费者就永远不会多余重渲染了。"**

`useMemo` 只保证**依赖没变时引用不变**。依赖真变了，消费者该更新还是得更新——那本来就是正确行为。它治的是"Provider 自己重渲染导致 value 白换引用"这一类。

**❌ 误区4："我只解构了其中一个字段，其它字段变了跟我没关系。"**

React 不知道你解构了什么，它只看 `value` 整体引用有没有变。要"只关心一部分"，只能拆 context，或者用带 selector 的库。

**❌ 误区5："`createContext` 传了默认值，所以忘了包 Provider 也能正常用。"**

能"不报错"，但值是**永远不会变**的默认值——你 set 了半天没反应。默认值只适合当兜底和测试环境的占位。

**❌ 误区6："Context 里放什么数据都行，反正 React 会优化。"**

高频变化的数据放进去，等于每次变化都广播全体消费者。表单输入、滚动位置、实时坐标这类数据，请交给外部 store。

## 一句话总结

**Context 只负责把值送下去、不负责管状态；value 用引用比较，所以要么稳住引用（useMemo）、要么拆细 context（状态与动作分离），高频更新的数据别放进来。**
