---
layout: post
title: "React Router路由原理"
date: 2026-03-24
categories: ["前端核心", "React"]
tags: ["React", "React Router", "路由", "SPA", "面试"]
---

## 一句话概括

React Router 核心就三件事：BrowserRouter 监听 URL 变化（History API / hashchange），Routes 用评分算法匹配路径，RouteContext + Outlet 把匹配结果渲染到正确位置。精炼下来就是"URL ↔ 组件映射"的声明式实现。

## 核心知识点

### 1. BrowserRouter vs HashRouter 的本质区别

```typescript
// BrowserRouter：基于 History API，URL 干净但需要服务端配合
window.history.pushState(null, '', '/about');  // URL变但不刷新
window.addEventListener('popstate', handler);   // 监听前进后退

// HashRouter：基于 location.hash，兼容所有浏览器但 URL 带 #
window.location.hash = '/about';               // 触发 hashchange
window.addEventListener('hashchange', handler);
```

BrowserRouter 直接刷新 `/about` 会 404，因为浏览器向服务端发了请求。**必须配 Nginx `try_files $uri /index.html` 把非静态资源的请求都回退到 index.html**。

### 2. 路径评分算法（为什么 Routes 不需要关心顺序）

```javascript
// React Router 的评分逻辑
function computeScore(path) {
  let score = 0;
  for (const seg of path.split('/').filter(Boolean)) {
    if (seg.startsWith(':'))  score += 1;   // 动态段 :id
    else if (seg === '*') score -= 1;     // 通配符
    else                    score += 10;   // 静态段 users
  }
  return score;
}

// /users/list   → 10+10=20 分
// /users/:id    → 10+1=11 分
// /:a/:b        → 1+1=2 分
// *             → -1 分
// 匹配时选分数最高的，和声明顺序无关！
```

这是 v6 Routes 替代 v5 Switch 的关键。v5 是"匹配第一个符合的"（依赖顺序），v6 是"匹配分数最高的"（不依赖顺序）。

### 3. 三层 Context 架构

```typescript
// React Router 内部的三层 Context
<NavigationContext>    // 提供 navigate() 方法
  <LocationContext>    // 提供当前 location 和 action
    <RouteContext>      // 提供当前匹配结果和 outlet（子路由元素）
      {children}
```

`useNavigate` → 读 NavigationContext；`useLocation` → 读 LocationContext；`useParams` / `Outlet` → 读 RouteContext。

### 4. Outlet 的嵌套渲染原理

```jsx
// Outlet 就是一个 "路由占位符"
function Outlet() {
  const routeContext = useContext(RouteContext);
  return routeContext.outlet;  // 当前匹配的下一级子元素
}

// Routes 用 reduceRight 从深层到浅层构建 Provider 嵌套
// 如果匹配到 /dashboard/settings：
// 最内层 Provider → outlet = <Settings/>
// 外层 Provider   → outlet = <DashboardLayout>（其内部 <Outlet/> 渲染 <Settings/>）
```

这就是为什么嵌套路由的 UI 层级能直接对应文件目录结构——每个 `<Route>` 节点的 element 里放 `<Outlet />`，子路由自动填入。

### 5. 导航守卫

```jsx
import { unstable_useBlocker as useBlocker } from 'react-router-dom';

function Editor() {
  const blocker = useBlocker(
    ({ nextLocation }) => hasUnsavedChanges
  );

  useEffect(() => {
    if (blocker.state === 'blocked') {
      if (confirm('有未保存的更改，确定离开？')) {
        blocker.proceed();
      } else {
        blocker.reset();
      }
    }
  }, [blocker]);

  return <textarea onChange={...} />;
}
```

底层原理：navigate 时先检查 blocker，被拦截就暂停更新并设置 `state='blocked'`，等用户决定。

## 其实你每天都在用

- **SPA 页面切换不刷新**：`<Link to="/about">` 点击后 URL 变了、内容变了，但浏览器没发 HTTP 请求——背后是 `pushState` + React 重渲染
- **直接访问深层路由**：刷新 `/dashboard/analytics` 页面正常显示——服务端 `try_files` 把请求交给 index.html，前端 Router 初始化时从当前 URL 解析路径匹配组件
- **嵌套布局复用**：`<Route path="dashboard" element={<Layout/>}><Route path="settings" element={<Settings/>}/></Route>` → 切换子路由时 Layout 组件不销毁，只换 Outlet 内容
- **路由传参**：`/user/:id` → `useParams()` 拿到 `{ id: '42' }`，比 query string 更 RESTful

## 常见误解

- **❌ 误区：「HashRouter 也能服务端渲染」** 不能。# 后面的内容不会发送给服务端，服务端永远只看到根路径。SSR 必须用 BrowserRouter。

- **❌ 误区：「Link 组件和 a 标签完全一样」** 不一样。Link 拦截点击事件，调用 `navigate()` 而非 `window.location.href`，不触发完整页面刷新。a 标签会导致白屏重载。

- **❌ 误区：「v6 的 Routes 和 v5 的 Switch 只是改名」** 不是。最大的变化是路径匹配从"声明顺序"改为"评分制"，同时嵌套路由不再需要手动在父组件中写 `<Route>` 和 `<Switch>`。

- **❌ 误区：「路由变化就是组件卸载再挂载」** 同级路由切换时是新组件挂载，但嵌套路由中父布局组件不会卸载——只有 `<Outlet/>` 内的内容会切换。

## 一句话总结

前端路由的本质 = URL 变化检测 + 路径匹配引擎 + 组件渲染占位——React Router 把这三件事全封装进了三级 Context，让你用写 JSX 的方式描述整个应用的页面结构。
