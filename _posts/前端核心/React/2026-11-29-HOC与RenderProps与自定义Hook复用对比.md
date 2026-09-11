---
layout: post
title: "HOC / Render Props / 自定义 Hook 复用对比"
date: 2026-11-29 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, HOC, RenderProps, 自定义Hook, 逻辑复用, 设计模式]
description: >
  面试向讲清 React 三代逻辑复用方案：HOC 高阶组件怎么才算写对（透传 props / displayName / 别在 render 里创建 / 转发 ref）、
  Render Props 和 HOC 的本质区别是静态组合 vs 动态组合、自定义 Hook 共享的究竟是逻辑还是状态、
  三者横向对比表，以及 Hooks 时代 HOC 和 Render Props 还有哪些场景不可替代。
---

## 一句话概括

HOC、Render Props、自定义 Hook 解决的**是同一件事：跨组件复用"带状态的逻辑"**（权限判断、鼠标位置、订阅外部数据、请求封装）。它们是 React 在不同时期的答案，演进顺序是：

**Mixin → HOC → Render Props → 自定义 Hook**

一代换一代，每一代都在修上一代的痛点：Mixin 有命名冲突和隐式依赖；HOC 会堆出"包装地狱"和 props 冲突；Render Props 解决了冲突但带来回调嵌套；Hooks 干脆**不加任何组件层级**，直接在函数里复用逻辑。

面试问这题，想听的不是"三个分别是啥"，而是：**为什么 Hooks 成了默认答案？以及 HOC / Render Props 什么时候还非用不可？** 后半句才是区分度。

一句话记住：**HOC 靠"包组件"复用，Render Props 靠"传函数"复用，自定义 Hook 靠"调函数"复用——层级越来越浅，耦合越来越松。**

## 核心知识点

### 1. 演进脉络：每一代到底在修什么问题

| 方案 | 时期 | 做法 | 被淘汰的原因 |
| --- | --- | --- | --- |
| Mixin | React 早期（`createReactClass`） | 把方法混进组件 | 命名冲突、依赖来源不清、ES6 class 不支持，官方直接判了"harmful" |
| HOC | React 0.13 起流行 | 函数接收组件，返回增强组件 | 包装地狱、props 来源不明、多个 HOC 注入同名 prop 会互相覆盖、TS 类型难推 |
| Render Props | React 16 前后 | 组件调用一个函数 prop 来决定渲染什么 | 逻辑没问题，但多层嵌套时 JSX 缩进灾难 |
| 自定义 Hook | React 16.8+ | 抽成 `useXxx` 函数 | 当前默认方案 |

注意一点：**演进的动力一直是"减少为了复用逻辑而付出的结构代价"**，不是因为哪一代"不能用"了。

### 2. HOC：函数接收组件，返回新组件

{% raw %}
```tsx
// 本质就是一个"组件 → 组件"的函数
const EnhancedComponent = withAuth(MyComponent);
```
{% endraw %}

写法上有 5 条硬规矩，漏一条就是坑，面试最好能一口气报出来：

1. **必须透传 `...props`**，不能只传你自己关心的那个；
2. **不要在 render 里创建 HOC**（见下面反例）；
3. **设置 `displayName`**，否则 DevTools 里全是 `_Anonymous`；
4. **转发 ref**：React 19 之前要用 `forwardRef`；React 19 起 `ref` 就是普通 prop，直接透传即可；
5. **拷贝静态方法**：被包装组件的静态属性不会自动带过来（现在基本用 `Object.assign` 或直接不依赖静态方法）。

{% raw %}
```tsx
// ❌ 反例：在 render 里创建 HOC —— 每次渲染都是一个新的组件类型
function Page() {
  const Enhanced = withAuth(Profile);   // 每次 render 都是新函数新类型
  return <Enhanced />;                 // React 会卸载旧子树、重新挂载，状态和 DOM 全丢
}

// ❌ 反例：只透传自己的 props，父组件传的 className / style 全被吞掉
function withAuth(Wrapped: any) {
  return function (props: any) {
    const user = useUser();
    return <Wrapped user={user} />;     // props 没往下传
  };
}
```
{% endraw %}

{% raw %}
```tsx
// ✅ 正确写法：透传 + displayName + 类型安全（React 19 中 ref 就是普通 prop）
type WithAuthProps = { user: User };

function withAuth<P extends object>(
  Wrapped: React.ComponentType<P & WithAuthProps>,
) {
  function WithAuth(props: P) {
    const { user, isLoading } = useUser();
    if (isLoading) return <Skeleton />;
    if (!user) return <Navigate to="/login" replace />;
    return <Wrapped {...(props as P)} user={user} />;   // ① 透传
  }
  WithAuth.displayName = `withAuth(${Wrapped.displayName ?? Wrapped.name ?? 'Component'})`; // ③
  return WithAuth;
}
```
{% endraw %}

**HOC 的硬伤**（为什么被 Hooks 取代）：

- **包装地狱**：`withRouter(withTheme(withAuth(withLogger(Page))))`，DevTools 里一层套一层；
- **props 来源不明**：组件里出现一个 `user`，你得顺着 4 层 HOC 找是谁注入的；
- **props 冲突**：两个 HOC 都注入 `user`，后一个静默覆盖前一个，不报错；
- **TS 类型难推**：泛型 + 高阶函数的组合，推断体验远差于一个普通 Hook。

### 3. Render Props：把"渲染什么"的决定权交给调用方

{% raw %}
```tsx
// props.children 是一个函数，组件把内部状态喂给它
<Mouse>
  {(mouse) => <Cat x={mouse.x} y={mouse.y} />}
</Mouse>;

// 内部实现
function Mouse({ children }: { children: (pos: { x: number; y: number }) => React.ReactNode }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const onMove = (e: MouseEvent) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', onMove);
    return () => window.removeEventListener('mousemove', onMove);
  }, []);
  return <>{children(pos)}</>;
}
```
{% endraw %}

**和 HOC 的本质区别，一句话：HOC 是静态组合，Render Props 是动态组合。**

- HOC 在**定义组件时**就把增强关系定死了，`withAuth(Page)` 写下去就改不了；
- Render Props 在**运行时**才决定：拿到 `mouse` 之后你可以根据 `mouse.x` 决定渲染猫还是狗，甚至什么都不渲染。

所以 React 官方的说法是"两者能力等价，能互相改写"，但动态性上 Render Props 更灵活，代价是嵌套深：

{% raw %}
```tsx
// ❌ 三层 Render Props 嵌套，缩进劝退（而且每层都是新函数，容易触发不必要的重渲染）
<Theme>
  {(theme) => (
    <Mouse>
      {(mouse) => (
        <User>
          {(user) => <Profile theme={theme} mouse={mouse} user={user} />}
        </User>
      )}
    </Mouse>
  )}
</Theme>

// ✅ 换成 Hooks，扁平，而且每个值的来源一目了然
function Profile() {
  const theme = useTheme();
  const mouse = useMouse();
  const user = useUser();
  return <ProfileView theme={theme} mouse={mouse} user={user} />;
}
```
{% endraw %}

### 4. 自定义 Hook：把逻辑抽成函数，不加任何层级

规则就三条，**都来自官方，不是社区约定**：

1. **名字必须 `use` + 大写字母开头**（`useOnlineStatus`）。这不是风格问题——linter 靠这个前缀判断"这个函数里能不能调 Hook"，你把它改成 `getOnlineStatus`，内部再调 `useState` 就会被 lint 报错；
2. **只有组件和 Hook 能调 Hook**，且必须在顶层调用，不能写在 `if` / 循环里；
3. **不调用任何 Hook 的函数别加 `use` 前缀**，写成普通函数（这样才允许条件调用）。

**最关键的一条认知：自定义 Hook 共享的是"有状态逻辑"，不是"状态本身"。**

{% raw %}
```tsx
// ❌ 误以为两个组件用同一个 Hook 就能共享数据 —— 不可能
function A() {
  const count = useCounter();   // 自己的 count
}
function B() {
  const count = useCounter();   // 另一份 count，互不影响
}
```
{% endraw %}

每次调用都完全独立，就像两次 `useState` 互不干扰一样。**真要跨组件共享同一份状态，得配 Context 或外部 store**（Zustand / `useSyncExternalStore`）。

{% raw %}
```tsx
// ✅ 共享逻辑 + 通过 Context 共享状态，这是标准答案
const ThemeContext = createContext<Theme | null>(null);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');
  return <ThemeContext.Provider value={{ theme, setTheme }}>{children}</ThemeContext.Provider>;
}

export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme 必须在 ThemeProvider 内使用');  // 别返回 null，早失败
  return ctx;
}
```
{% endraw %}

**还有一条官方明说的反模式**：别造 `useMount` / `useEffectOnce` 这种"生命周期马甲" Hook。它只是把 `useEffect` 换个名字，反而让调用方搞不清依赖是怎么处理的。Hook 要按**业务用例**命名——`useChatRoom`、`useOnlineStatus`、`useTodoList`，而不是 `useEffectOnce`。

### 5. 三者横向对比（这张表背下来，面试直接画）

| 对比项 | HOC | Render Props | 自定义 Hook |
| --- | --- | --- | --- |
| 本质 | 组件包组件 | 函数作 props | 函数调用 |
| 复用的是什么 | 逻辑 + 渲染增强 | 逻辑（渲染交给调用方） | 纯逻辑 |
| 是否增加组件层级 | 是 | 是 | **否** |
| props / 命名冲突 | 有（同名注入互相覆盖） | 无（参数自己命名） | 无（返回值自己命名） |
| 组合方式 | 嵌套调用，静态 | 嵌套 JSX，动态 | 平铺调用，最自由 |
| 数据来源可读性 | 差（要顺着包装链找） | 好（就在眼前） | 最好（就在调用那行） |
| TypeScript 支持 | 差 | 中 | **好** |
| 适用组件 | 函数 + class | 函数 + class | **仅函数组件** |
| ref 处理 | 需要转发 | 不需要 | 不需要 |
| 当前推荐度 | 特定场景 | 特定场景 | **默认首选** |

### 6. 关键追问：Hooks 时代，HOC 和 Render Props 还有用吗

**有用，但范围收窄了。** 答不出下面这些，就容易说出"HOC 已经过时了"这种绝对化的错话。

**还得用 HOC 的场景：**

1. **错误边界**——React 至今**没有** `useErrorBoundary`，捕获渲染错误只能靠 class 组件的 `componentDidCatch` / `getDerivedStateFromError`。想把它做成可复用的东西，就得包一层 HOC（`react-error-boundary` 就是这么干的）；
2. **包装你改不了源码的组件**（三方库组件、路由组件），而且你要改的是"它渲染什么"，不只是给它喂数据；
3. **横切关注点**：埋点、权限拦截、feature flag、性能打点——这类"给一批组件统一加一层"的需求，HOC 比在每个组件里加一行 Hook 更好管；
4. **老代码和老库**：Redux 的 `connect`、React Router v5 的 `withRouter`，存量项目里遍地都是。

**还得用 Render Props 的场景：**

1. **Headless UI 库**：Downshift、React Table 早期的 API 都是 Render Props——库负责行为和无障碍，渲染完全交给你；
2. **虚拟列表**：`renderItem={(item, index) => ...}` 本质上就是 Render Props，库负责"算可见区间"，你负责"每一行长啥样"；
3. **需要把内部状态暴露给 children 的容器组件**。

### 7. 加分项：真正取代它们的是「组合组件 + Hook」

严格说，现在组件库（Radix UI、Headless UI、Arco）的主流 API 不是上面三者，而是**组合组件（Compound Components）**：用 Context 共享隐式状态，用 Hook 消费。

{% raw %}
```tsx
// 像 HTML 的 <select> / <option> 一样：父管状态，子隐式通信，结构由调用方自由拼装
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Trigger value="a">基本信息</Tabs.Trigger>
    <Tabs.Trigger value="b">权限配置</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Panel value="a">…</Tabs.Panel>
  <Tabs.Panel value="b">…</Tabs.Panel>
</Tabs>
```
{% endraw %}

所以更准确的说法是：**Hooks 取代的是"逻辑复用"这一层，而"灵活的组件 API"由组合组件接管**——这也解释了为什么 Headless UI 库既用 Context 又导出 Hook。

## 其实你每天都在用

1. **登录态拦截**：`<Route>` 外面套一层 `withAuth`，或者在组件里 `const { user } = useAuth()`——同一件事的两代写法。
2. **Redux 的 `connect(mapState, mapDispatch)(Comp)`**：最经典的 HOC，现在基本被 `useSelector` / `useDispatch` 取代了。
3. **React Router v5 的 `withRouter`**：给组件注入 `history / location / match`，v6 起全改成 `useNavigate` / `useLocation`。
4. **虚拟列表的 `renderItem`**：React Window / react-virtualized 全是 Render Props。
5. **`useRequest` / `useSWR` / `useQuery`**：一个 Hook 顶掉以前"请求 HOC"的活儿。
6. **表单库的 `useForm`**：antd 的 `Form.useForm()`，替代了早期 `Form.create()(Comp)` 这个 HOC。
7. **埋点 HOC**：`withTracker(Page)`，进页面自动上报 PV，业务组件零侵入。
8. **`react-error-boundary` 的 `withErrorBoundary`**：因为错误边界只能用 class，这是 HOC 至今赖着不走的核心原因。

## 常见误解（FAQ）

**❌ 误区1："Hooks 出来之后，HOC 和 Render Props 就彻底没用了。"**

错得最典型的一句。**错误边界至今没有 Hook 版本**，只能用 class 组件实现，要复用就得 HOC。另外三方库 API、横切埋点、包装不可修改的组件，都还是 HOC 的地盘。准确说法是"逻辑复用首选 Hook，但 HOC / Render Props 仍有不可替代的场景"。

**❌ 误区2："Render Props 就是给 children 传一段 JSX。"**

不是。传的是**函数**，组件在内部调用它并把自己的状态当参数喂出来。`<Panel><Header /></Panel>` 叫普通 children（插槽），`<Panel>{(open) => <Header open={open} />}</Panel>` 才叫 Render Props。用 `render` 还是 `children` 做 prop 名无所谓，关键是那个值是函数。

**❌ 误区3："自定义 Hook 能让多个组件共享同一份状态。"**

**共享的是逻辑，不是状态。** 每次 `useCounter()` 都是一份独立的状态，两个组件各调一次就是两份互不相干的数据。要真共享，得把状态放进 Context 或外部 store，再让 Hook 去消费——这也是所有状态库的标准做法。

**❌ 误区4："HOC 里把我需要注入的 props 传进去就行了，父组件传的不用管。"**

必须透传 `{...props}`。你不透传，父组件给的 `className`、`style`、`onClick` 全被吞掉，表现为"样式和事件莫名其妙失效"，而且极难排查。

**❌ 误区5："封装一个 `useMount` 很实用，比写 `useEffect(..., [])` 清爽。"**

官方明确不推荐。这类"生命周期马甲" Hook 唯一的作用是换个名字，反而掩盖了依赖数组，让人误以为它和外部状态无关。Hook 应该按用例命名（`useChatRoom`），让你看到名字就知道它在干嘛。

**❌ 误区6："HOC 和 Render Props 能互相改写，所以没本质区别。"**

能力上等价，但**组合时机不同**：HOC 是静态组合，写组件时就定死了增强关系；Render Props 是动态组合，运行时拿到数据再决定渲染什么。需要"根据内部状态动态决定渲染内容"时，Render Props 更自然；需要"给一批组件统一加一层"时，HOC 更省事。

## 一句话总结

**三者都是"组合优于继承"的产物，区别只在为了复用逻辑付出了多少结构代价：HOC 包一层组件、Render Props 传一个函数、自定义 Hook 只调一次函数——所以新代码优先用自定义 Hook（无层级、无命名冲突、类型和调试都最好），HOC 留给错误边界和不可修改的第三方组件，Render Props 留给需要把渲染权交给调用方的 Headless / 虚拟列表场景。**
