---
layout: post
title: "Vue Router路由原理"
date: 2026-07-01
tags: [Vue3, Vue Router, 路由, SPA]
categories: [前端核心, Vue]
---

## 一句话概括

Vue Router 是 Vue.js 生态中官方的前端路由解决方案，它通过监听浏览器地址变化，将 URL 与组件树建立映射关系，在**不刷新页面**的前提下实现视图的切换与状态管理，其核心是 Hash 与 History 两种底层模式的选择、一套高效的路由匹配算法以及严格可预测的导航守卫管线。

## 背景与意义

### 从 MPA 到 SPA 的范式转变

在单页应用（SPA）出现之前，Web 应用普遍采用多页应用（MPA）架构。每次页面跳转都意味着一次完整的 HTTP 请求-响应周期：浏览器发送请求、服务端渲染完整 HTML、客户端重新加载 CSS/JS 资源。这种模式的用户体验存在明显短板——页面切换存在白屏等待、状态难以跨页保持、交互连贯性差。

SPA 的出现改变了这一切。应用启动时加载完整的 JavaScript 包，后续的所有页面切换都在客户端完成，仅通过与服务端的 API 交互获取数据。但 SPA 带来了一个新问题：**如何在客户端管理"页面"的概念？** 用户期望的浏览器前进/后退按钮、可收藏的书签地址、可分享的 URL——这些在 SPA 中无法天然获得。

### 前端路由的核心使命

前端路由的诞生正是为了解决这个矛盾。它需要在客户端模拟传统多页应用的路由行为，同时保持 SPA 的流畅体验。具体来说，前端路由需要完成以下工作：

1. **URL 与视图的绑定**：改变 URL 时，呈现对应的组件
2. **浏览历史的维护**：支持浏览器的前进/后退导航
3. **不触发服务端请求**：URL 变化不引起页面重载
4. **导航控制**：提供守卫机制拦截、重定向或取消导航
5. **状态同步**：URL 参数与组件状态的双向同步

Vue Router 作为 Vue 生态的官方路由方案，完美地解决了上述问题，并且与 Vue 的响应式系统深度集成，提供了声明式的路由配置、嵌套路由、命名视图、路由元信息等丰富特性。

## 概念与定义

在深入源码之前，我们先梳理 Vue Router 的核心概念：

### 路由记录（RouteRecord）

路由记录是路由配置的基本单元，包含路径（path）、组件（component）、子路由（children）等信息。Vue Router 内部维护着一个扁平化的路由记录列表，用于快速匹配。

```typescript
// RouteRecord 的核心字段
interface RouteRecord {
  path: string;           // 路径模式，如 '/users/:id'
  regex: RegExp;          // 编译后的正则表达式
  components: Record<string, Component>;
  name?: string;          // 命名路由
  alias?: string | string[];
  redirect?: RouteRecordRedirect;
  meta?: RouteMeta;       // 路由元信息
  beforeEnter?: NavigationGuard;
  children?: RouteRecord[];  // 嵌套子路由
}
```

### 路由位置（RouteLocation）

路由位置是对当前 URL 的结构化描述，包含了路径、参数、查询字符串、哈希等解析后的信息。

```typescript
interface RouteLocation {
  path: string;
  fullPath: string;       // 完整路径（含查询和哈希）
  name?: string | null;
  params: Record<string, string>;  // 路径参数
  query: Record<string, string | string[]>;
  hash: string;
  matched: RouteRecord[]; // 匹配到的路由记录链（用于嵌套路由）
  meta: RouteMeta;        // 合并后的元信息
  redirectedFrom?: RouteLocation;
}
```

### 路由守卫（Navigation Guard）

守卫是导航过程中的钩子函数，可以在导航的不同阶段执行逻辑。守卫本质上是一个**函数管线**，按严格的顺序执行。

### 路由器（Router）

路由器是核心控制器，管理路由表、历史状态、导航流程，对外暴露 `push`、`replace`、`addRoute`、`beforeEach` 等 API。

## 最小示例

```vue
<!-- 1. 安装依赖 -->
<!-- npm install vue-router@4 -->

<!-- 2. 定义路由配置 -->
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'
import User from '../views/User.vue'

const routes = [
  {
    path: '/',
    name: 'home',
    component: Home
  },
  {
    path: '/users/:id',
    name: 'user',
    component: User,
    props: true
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

export default router

<!-- 3. 注册路由 -->
// main.ts
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App)
app.use(router)
app.mount('#app')

<!-- 4. 模板中使用 -->
<!-- App.vue -->
<template>
  <nav>
    <RouterLink to="/">首页</RouterLink>
    <RouterLink to="/users/42">用户42</RouterLink>
  </nav>
  <!-- 路由出口 -->
  <RouterView />
</template>
```

这个最小示例涵盖了 Vue Router 的完整工作流：配置路由 → 注入应用 → 声明链接 → 渲染视图。但更值得我们关注的是，当用户点击路由链接时，Vue Router 内部经历了怎样的流程。

## 核心知识点拆解

### 1. Hash 模式与 History 模式

这是 Vue Router 最基础的底层设计决策。两种模式的核心区别在于：**如何在不刷新页面的前提下改变 URL**。

#### Hash 模式

Hash 模式利用 URL 中 `#` 及其后面的部分（hash）不会触发服务端请求这一特性。当 hash 变化时，浏览器不会向服务端发送请求，但会触发 `hashchange` 事件。

**实现原理：**

```typescript
function createWebHashHistory() {
  // 获取当前路径（去掉 # 号）
  function getCurrentLocation() {
    const href = window.location.href
    const hashIndex = href.indexOf('#')
    return hashIndex === -1 ? '/' : href.slice(hashIndex + 1) || '/'
  }

  // 监听 hash 变化
  window.addEventListener('hashchange', () => {
    // 通知路由器更新路由
    routerHistoryListener(getCurrentLocation())
  })

  // 推入新状态
  function push(to: string) {
    window.location.hash = to
  }

  // 替换当前状态（不增加历史记录）
  function replace(to: string) {
    const i = window.location.href.indexOf('#')
    window.location.replace(
      window.location.href.slice(0, i >= 0 ? i : 0) + '#' + to
    )
  }

  return { getCurrentLocation, push, replace }
}
```

**优点：** 兼容性好，无需服务端配合，在所有浏览器（包括 IE）中都能正常工作。**缺点：** URL 中包含 `#` 不够美观，且 hash 部分不会被服务端接收到，不利于 SEO。

#### History 模式

History 模式利用 HTML5 提供的 History API（`pushState` 和 `replaceState`）来操作浏览器的历史栈，同时配合 `popstate` 事件监听 URL 变化。

```typescript
function createWebHistory() {
  function getCurrentLocation() {
    return window.location.pathname + window.location.search + window.location.hash
  }

  // 监听浏览器前进/后退
  window.addEventListener('popstate', () => {
    routerHistoryListener(getCurrentLocation())
  })

  // pushState 推入新状态
  function push(to: string) {
    const currentState = history.state || {}
    history.pushState(currentState, '', to)
    // 手动触发通知
    routerHistoryListener(to)
  }

  // replaceState 替换当前状态
  function replace(to: string) {
    history.replaceState(history.state || {}, '', to)
    routerHistoryListener(to)
  }

  return { getCurrentLocation, push, replace }
}
```

**关键细节：** `pushState` 和 `replaceState` **不会触发** `popstate` 事件。因此通过 `push`/`replace` 进行程序化导航时，路由器需要**手动触发**导航流程。`popstate` 事件只在用户点击浏览器的前进/后退按钮或调用 `history.back()`、`history.go()` 时触发。

**优点：** URL 美观，与传统 URL 无异。**缺点：** 需要服务端配合做 fallback（将所有路由指向 index.html），否则直接访问子路径会返回 404。

#### 两种模式的选择与降级

```typescript
// Vue Router 4 中的历史模式创建
function createRouter(options) {
  // 根据用户传入的 history 对象区分模式
  const history = options.history

  // Vue Router 内部统一调用 history.push/replace
  // 不关心底层是 hash 还是 history API
}
```

最佳实践：一般情况下推荐使用 History 模式。当部署在静态文件服务器或 CDN 上且无法做服务端 rewrite 时，回退到 Hash 模式。

### 2. 导航守卫管线

导航守卫是 Vue Router 最核心的**流程控制**机制。它的设计思想类似于 Web 框架中的中间件（Middleware）或过滤器（Filter）——多个守卫函数串联成一个管线，每个守卫可以放行、重定向或取消导航。

#### 守卫类型

```typescript
// 全局前置守卫
router.beforeEach(async (to, from) => {
  if (to.meta.requiresAuth && !isLoggedIn()) {
    return '/login'  // 重定向
  }
})

// 全局解析守卫（在所有组件内守卫和异步路由组件解析之后调用）
router.beforeResolve(async (to, from) => {
  // 用于数据预取
  await prefetchData(to)
})

// 全局后置守卫（导航已确认，DOM 即将更新）
router.afterEach((to, from, failure) => {
  // 埋点上报、页面标题更新等
  document.title = to.meta.title
  sendAnalytics(to.fullPath)
})

// 路由独享守卫（在路由配置中定义）
{
  path: '/users/:id',
  component: User,
  beforeEnter: (to, from) => {
    // 仅当进入此路由时调用
  }
}

// 组件内守卫（在组件选项中定义）
const UserComponent = {
  beforeRouteEnter(to, from, next) {
    // 导航确认前调用，此时组件实例尚未创建
    // 通过 next(vm => {}) 回调访问组件实例
  },
  beforeRouteUpdate(to, from, next) {
    // 路由参数变化但组件复用时调用
  },
  beforeRouteLeave(to, from, next) {
    // 离开当前路由时调用，用于提示保存未提交的修改
  }
}
```

#### 管线的执行顺序

Vue Router 的导航管线具有严格的顺序，理解这个顺序对于正确使用守卫至关重要：

```
1. 导航被触发（点击链接、router.push等）
2. 调用即将离开的组件内的 beforeRouteLeave 守卫
3. 调用全局的 beforeEach 守卫
4. 调用与即将进入路由匹配的 beforeEnter 守卫
5. 解析异步路由组件
6. 调用即将进入的组件内的 beforeRouteEnter 守卫
7. 调用全局的 beforeResolve 守卫
8. 导航被确认
9. 调用全局的 afterEach 守卫
10. 触发 DOM 更新
11. 调用 beforeRouteEnter 中传给 next 的回调函数
```

#### 管线实现源码分析

```typescript
// Vue Router 内部的核心导航管道
async function navigate(to: RouteLocationNormalized, from: RouteLocationNormalized): Promise<any> {
  // guards 是 Guard 实例的数组（每个包含 enter/leave 守卫）
  const guards = extractComponentsGuards(
    to.matched.map(record => record.components.default),
    from.matched.map(record => record.components.default)
  )

  // 1. 离开守卫
  for (const guard of guards.leavingRecords) {
    await guard.beforeRouteLeave(to, from)
  }

  // 2. 全局前置守卫
  for (const guard of router.beforeEachGuards) {
    const result = await guard(to, from)
    if (result !== undefined) {
      return result  // 返回重定向地址或 false
    }
  }

  // 3. 路由独享守卫
  const toRecords = to.matched
  for (const record of toRecords) {
    if (record.beforeEnter) {
      const result = await record.beforeEnter(to, from)
      if (result !== undefined) return result
    }
  }

  // 4. 解析异步组件
  await Promise.all(to.matched.map(record =>
    record.components && loadAsyncComponent(record.components.default)
  ))

  // 5. 进入守卫（组件内）
  for (const guard of guards.enteringRecords) {
    await guard.beforeRouteEnter(to, from)
  }

  // 6. 全局解析守卫
  for (const guard of router.beforeResolveGuards) {
    const result = await guard(to, from)
    if (result !== undefined) return result
  }

  // 导航确认——到这里为止导航不可逆
  return true
}
```

**设计亮点：** 每个守卫函数可以是普通函数或 async 函数，Vue Router 通过 `await` 统一处理异步情况。守卫函数的返回值具有语义：`undefined` 表示放行，字符串或 `RouteLocation` 表示重定向，`false` 表示取消导航。这种设计比旧的 `next()` 回调模式更加直观且不易出错。

### 3. 动态路由添加

Vue Router 允许在运行时通过 `router.addRoute()` 动态添加路由，这在大型 SPA 中非常实用——尤其是权限控制、模块懒加载等场景。

```typescript
// 基本用法：添加顶级路由
router.addRoute({
  path: '/admin',
  component: () => import('./views/Admin.vue'),
  meta: { requiresAuth: true }
})

// 添加嵌套子路由（挂在指定父路由下）
router.addRoute('parent', {
  path: 'child',
  component: ChildComponent
})

// 移除动态添加的路由
router.removeRoute('admin')

// 获取所有路由记录
router.getRoutes()
```

**实现原理：** `addRoute` 调用后，Vue Router 内部会做以下事情：

1. 将传入的配置标准化为 `RouteRecord` 对象
2. 如果是嵌套路由，建立父子关联，在父路由的 `children` 数组中添加
3. 用该路由的 `path` 重新编译生成 `RouteRecordMatcher`（正则匹配器）
4. 将新的匹配器插入到路由匹配器数组的正确位置（考虑优先级）
5. **旧路由记录缓存失效**——下次导航时会用更新后的匹配器列表

```typescript
// Vue Router 4 中的 addRoute 简化源码
function addRoute(parentOrRoute: string | RouteRecord, route?: RouteRecord) {
  // 1. 处理参数
  let parent: RouteRecord | undefined
  if (typeof parentOrRoute === 'string') {
    parent = matcher.keys.get(parentOrRoute)?.record
  }

  // 2. 标准化并创建匹配器
  const record = route !== undefined ? route : parentOrRoute
  const matcher = createRouteRecordMatcher(record, parent)

  // 3. 插入匹配器列表（按优先级排序）
  if (hasPath(record) && record.path !== '/') {
    insertMatcher(matcher)
  }

  // 4. 处理别名
  record.alias?.forEach(alias => {
    const aliasMatcher = createRouteRecordMatcher(
      { ...record, path: alias, alias: undefined },
      parent
    )
    insertMatcher(aliasMatcher)
  })

  return () => removeMatcher(matcher)
}
```

**注意陷阱：** 动态添加路由后，如果当前 URL 匹配的是旧路由，Vue Router 不会自动重新解析当前导航。你需要在适当时机手动调用 `router.replace(router.currentRoute.value.fullPath)` 来重新匹配。

### 4. 路由匹配算法（RouteRecordMatcher）

路由匹配是 Vue Router 性能的关键。想象一下，当应用中定义了上百个路由，用户访问 `/users/42/posts/5` 时，路由器需要找到对应的组件链。Vue Router 使用了一套**基于优先级排序的线性扫描 + 正则匹配**算法。

#### 路径的编译

Vue Router 内部使用 `path-to-regexp` 库（Vue Router 4 内置了修改版 `@vue-router/path-to-regexp`），将声明式路径模式编译为正则表达式：

```
/user/:id       → /^\/user\/([^\/]+?)(?:\/)?$/i

/user/:id?      → /^\/user(?:\/([^\/]+?))?(?:\/)?$/i（参数可选）

/user/:id(\\d+) → /^\/user\/(\d+)(?:\/)?$/i（参数约束）

/files/*path    → /^\/files\/(.*)(?:\/)?$/i（通配匹配）
```

#### RouteRecordMatcher 的实现

```typescript
class RouteRecordMatcher {
  record: RouteRecord
  parent: RouteRecordMatcher | undefined
  children: RouteRecordMatcher[]
  regex: RegExp              // 编译后的正则
  keys: { name: string }[]   // 参数名列表

  constructor(record: RouteRecord, parent?: RouteRecordMatcher) {
    this.record = record
    this.parent = parent
    this.children = []

    // 编译路径为正则
    const compiled = pathToRegexp(record.path)
    this.regex = compiled.regex
    this.keys = compiled.keys
  }

  // 测试路径是否匹配
  test(path: string): boolean {
    return this.regex.test(path)
  }

  // 从路径中提取参数
  parse(path: string): Record<string, string> | null {
    const match = path.match(this.regex)
    if (!match) return null

    const params: Record<string, string> = {}
    for (let i = 1; i < match.length; i++) {
      const key = this.keys[i - 1]?.name
      if (key) params[key] = match[i]
    }
    return params
  }
}
```

#### 匹配流程

```typescript
// 简化版的匹配逻辑
function resolve(path: string): RouteLocation {
  const pathWithoutQuery = path.split('?')[0]

  // 按照优先级依次检查每个匹配器
  for (const matcher of matchers) {
    if (matcher.test(pathWithoutQuery)) {
      // 收集匹配链（当前 + 所有父级，用于嵌套路由）
      const matched: RouteRecordMatcher[] = []
      let current: RouteRecordMatcher | undefined = matcher
      while (current) {
        matched.unshift(current)  // 从根到叶子
        current = current.parent
      }

      // 提取参数
      const params = matcher.parse(pathWithoutQuery)

      return {
        matched: matched.map(m => m.record),
        params,
        path: pathWithoutQuery,
        // ...
      }
    }
  }

  // 未匹配到任何路由
  throw createRouterError('No match')
}
```

#### 优先级规则

Vue Router 使用**得分（score）**来确定匹配器的优先级，得分高的先被检查：

1. **静态路径** > **动态路径**（`/users/list` 优先于 `/users/:id`）
2. **参数数量少** > **参数数量多**
3. **有约束的参数** > **无约束的参数**（`/user/:id(\\d+)` 优先于 `/user/:id`）
4. **嵌套深度浅** > **嵌套深度深**

```typescript
// 优先级计算示意
function computeScore(path: string): number {
  const segments = path.split('/').filter(Boolean)
  let score = segments.length * 1000  // 路径越深，基数越大

  for (const segment of segments) {
    if (segment.startsWith(':')) {
      score -= 100  // 动态段扣分
      if (segment.includes('(')) {
        score += 10 // 有约束则加回
      }
    }
  }
  return score
}
```

这种设计确保了**最精确的匹配结果**——当存在多个可能匹配的路径时，得分最高的胜出。

## 实战案例

### 案例：权限控制的完整实现

```typescript
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/auth'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    {
      path: '/',
      component: () => import('@/layouts/DefaultLayout.vue'),
      children: [
        {
          path: '',
          name: 'home',
          component: () => import('@/views/Home.vue'),
        },
        {
          path: 'dashboard',
          name: 'dashboard',
          component: () => import('@/views/Dashboard.vue'),
          meta: { requiresAuth: true, roles: ['admin', 'editor'] }
        },
        {
          path: 'settings',
          component: () => import('@/views/Settings.vue'),
          meta: { requiresAuth: true },
          children: [
            {
              path: 'profile',
              name: 'profile',
              component: () => import('@/views/Profile.vue')
            },
            {
              path: 'security',
              name: 'security',
              component: () => import('@/views/Security.vue'),
              meta: { roles: ['admin'] }
            }
          ]
        }
      ]
    },
    { path: '/login', name: 'login', component: () => import('@/views/Login.vue') },
    { path: '/403', name: 'forbidden', component: () => import('@/views/Forbidden.vue') },
    { path: '/:pathMatch(.*)*', name: 'not-found', component: () => import('@/views/NotFound.vue') }
  ]
})

// 全局前置守卫——权限校验
router.beforeEach(async (to, from) => {
  const auth = useAuthStore()

  // 1. 判断是否需要登录
  const requiresAuth = to.matched.some(record => record.meta.requiresAuth)

  // 2. 如果需要登录但未登录
  if (requiresAuth && !auth.isLoggedIn) {
    return {
      name: 'login',
      query: { redirect: to.fullPath } // 登录后回跳
    }
  }

  // 3. 已登录到登录页，直接重定向到首页
  if (to.name === 'login' && auth.isLoggedIn) {
    return { name: 'home' }
  }

  // 4. 角色权限校验
  const requiredRoles = to.matched
    .flatMap(record => (record.meta.roles as string[]) || [])
  if (requiredRoles.length > 0) {
    const hasRole = requiredRoles.some(role => auth.hasRole(role))
    if (!hasRole) {
      return { name: 'forbidden' }
    }
  }
})

// 全局后置守卫——页面标题与埋点
router.afterEach((to, from) => {
  // 更新页面标题
  const title = to.meta.title as string
  if (title) {
    document.title = `${title} - My App`
  }

  // 页面访问埋点
  if (typeof window._paq !== 'undefined') {
    window._paq.push(['trackPageView'])
    window._paq.push(['setDocumentTitle', document.title])
  }

  // 滚动到顶部（除了保持滚动位置的页面）
  if (!to.meta.keepAlive) {
    window.scrollTo(0, 0)
  }
})

// 动态注册路由——根据用户权限动态加载模块
export function registerModuleRoutes(moduleName: string, routes: RouteRecord[]) {
  const existing = router.hasRoute(moduleName)
  if (existing) {
    router.removeRoute(moduleName)
  }
  routes.forEach(route => router.addRoute(moduleName, route))
}

// 组件内守卫——离开时保存表单状态
// Dashboard.vue
export default {
  beforeRouteLeave(to, from, next) {
    if (this.hasUnsavedChanges) {
      const answer = window.confirm('有未保存的更改，确定要离开吗？')
      if (!answer) {
        return false
      }
    }
    next()
  }
}
```

### 案例：scrollBehavior 的深度定制

```typescript
const router = createRouter({
  history: createWebHistory(),
  routes: [/* ... */],
  scrollBehavior(to, from, savedPosition) {
    // savedPosition 是浏览器「前进/后退」时保存的滚动位置
    if (savedPosition) {
      return savedPosition  // 恢复
    }

    // 处理 hash 锚点
    if (to.hash) {
      return {
        el: to.hash,
        behavior: 'smooth'  // 平滑滚动
      }
    }

    // 只对不同路由做顶部滚动，同一路由保持位置
    if (to.path !== from.path) {
      return { top: 0 }
    }

    // 返回空对象表示不改变滚动位置
    return {}
  }
})
```

**scrollBehavior 的实现原理：** Vue Router 在导航确认、DOM 更新完成之后，调用用户配置的 `scrollBehavior` 函数。函数返回一个描述滚动位置的对象，路由器使用 `window.scrollTo` 或 `el.scrollIntoView` 来执行滚动。如果返回 `false`，则跳过滚动处理。

```typescript
// 内部实现简化版
async function updateScroll(router: Router) {
  const behavior = router.options.scrollBehavior
  if (!behavior) return

  const to = router.currentRoute.value
  const from = router.previousRoute.value
  const savedPosition = router.savedScrollPosition?.[to.fullPath]

  const position = await behavior(to, from, savedPosition)
  if (position === false) return

  await nextTick()  // 确保 DOM 已更新

  if (position) {
    if ('el' in position) {
      const el = document.querySelector(position.el)
      el?.scrollIntoView({ behavior: position.behavior as ScrollBehavior })
    } else {
      window.scrollTo({
        left: position.left || 0,
        top: position.top || 0,
        behavior: position.behavior as ScrollBehavior
      })
    }
  }
}
```

## 底层原理

### 完整的导航解析流程

下面的伪代码梳理了 Vue Router 4 从 `router.push` 到视图更新的完整流程：

```typescript
// 1. 触发导航
Router.prototype.push = function(to: RawLocation) {
  return this.history.push(to)  // history.pushState 或 location.hash = to
}

// 2. 解析目标路由
// history 层接收到地址变化后，调用 router.resolve
function resolveRoute(location: string): NavigationResult {
  // 根据 path 解析为 RouteLocationNormalized
  const resolved = matcher.resolve(location)

  // 处理重定向
  if (resolved.redirect) {
    return handleRedirect(resolved.redirect, location)
  }

  return resolved
}

// 3. 运行守卫管线
async function guardPipeline(to, from): Promise<NavigationResult> {
  let result: NavigationResult | undefined

  // beforeRouteLeave（依次从最深到最浅）
  for (const record of [...from.matched].reverse()) {
    result = await runGuardOnComponent(record, 'beforeRouteLeave', to, from)
    if (result !== undefined) return result
  }

  // beforeEach（全局）
  result = await runGuardList(router.beforeEachGuards, to, from)
  if (result !== undefined) return result

  // beforeEnter（依次从最浅到最深）
  for (const record of to.matched) {
    if (record.beforeEnter) {
      result = await record.beforeEnter(to, from)
      if (result !== undefined) return result
    }
  }

  // 解析异步组件
  await Promise.all(to.matched.map(loadRouteComponent))

  // beforeRouteEnter
  for (const record of [...to.matched].reverse()) {
    result = await runGuardOnComponent(record, 'beforeRouteEnter', to, from)
    if (result !== undefined) return result
  }

  // beforeResolve
  result = await runGuardList(router.beforeResolveGuards, to, from)
  if (result !== undefined) return result

  // 导航确认
  return NavigationResult.CONFIRMED
}

// 4. 更新视图
function updateRoute(to: RouteLocation) {
  // 内部更新 currentRoute（响应式数据）
  router.currentRoute.value = to

  // 触发 RouterView 的重新渲染
  triggerReactivity()

  // 执行 afterEach 守卫
  runGuardList(router.afterEachGuards, to, from)

  // 处理 scrollBehavior
  updateScroll(router)
}
```

### 响应式集成

Vue Router 与 Vue 响应式系统的集成是它的一个关键设计。`currentRoute` 是一个 `ref` 对象：

```typescript
const currentRoute = shallowRef<RouteLocationNormalizedLoaded>(initialRoute)

// 当路由变化时
currentRoute.value = newRoute

// 所有引用 currentRoute 的组件自动重新渲染
// RouterView 组件内部：
setup() {
  const route = inject(routerViewLocationKey)!
  return () => {
    const current = route.value
    const matched = current.matched
    // 根据 matched 渲染对应的组件层级
  }
}
```

正是因为使用了 `shallowRef`，**路由变化时只有使用 `RouterView` 和 `useRoute()` 的组件会重新渲染**，而不是整个应用。这种细粒度的响应式更新是 Vue Router 高性能的关键。

### 嵌套路由的实现

嵌套路由是 Vue Router 最强大的功能之一。当路由嵌套时，多个 `RouterView` 组件按层级关系渲染。

```vue
<!-- 顶层 App.vue -->
<template>
  <RouterView />  <!-- 渲染 layout 组件 -->
</template>

<!-- layouts/DefaultLayout.vue -->
<template>
  <header>导航栏</header>
  <aside>侧边栏</aside>
  <main>
    <RouterView />  <!-- 渲染子路由组件 -->
  </main>
</template>
```

在渲染层面，每个 `RouterView` 组件通过 `inject` 获取当前路由在所在层级的匹配状态：

```typescript
// RouterView 组件实现（简化）
const RouterView = defineComponent({
  name: 'RouterView',
  setup(props, { slots }) {
    const injectedRoute = inject(routerViewLocationKey)!
    const depth = inject(routerViewDepthKey, 0)  // 当前深度

    // 每一层 RouterView 都提供下一层的 depth
    provide(routerViewDepthKey, depth + 1)

    return () => {
      const route = injectedRoute.value
      const matched = route.matched[depth]  // 取当前层级的记录

      if (!matched) {
        // 无匹配，渲染 fallback 或空白
        return slots.default?.() || h('div')
      }

      // 渲染匹配的组件
      const component = matched.components.default
      return h(component)
    }
  }
})
```

这种基于 depth 的匹配机制，使得嵌套路由的组件渲染完全由数据结构驱动，无需开发者手动关联。

## 高频面试题解析

### Q1：router.push 和 router.replace 的区别是什么？

`router.push` 会向浏览器历史栈中添加一条新记录，用户可以通过「后退」按钮返回到之前的路由。`router.replace` 则替换当前的历史记录，不会增加新的记录。底层对应的是 `history.pushState` 和 `history.replaceState` 的区别。

**实际场景：** 在登录成功后跳转首页使用 `replace` 而非 `push`，防止用户通过「后退」按钮回到登录页。

### Q2：Vue Router 的两种模式如何选择？

**Hash 模式：** 兼容性好，不需要服务端配置，适合部署在静态文件服务器上的应用。URL 带 `#` 号，不够美观。
**History 模式：** URL 美观，需要服务端配置所有路由指向 `index.html`（SPA fallback）。当部署在 CDN 或无法配置服务端的环境时不可用。

**决策建议：** 能配服务端就用 History，否则用 Hash。Vue Router 4 支持在运行时创建路由器时灵活选择。

### Q3：addRoute 动态添加的路由如何持久化？

`addRoute` 添加的路由仅存在于当前运行时的内存中，页面刷新后会丢失。如果需要持久化，应在应用初始化时从服务端获取路由配置并进行动态添加。通常做法：

```typescript
// router/index.ts
export async function setupRouter(app: App) {
  const router = createRouter({ /*...*/ })

  // 从服务端获取动态路由
  if (isAuthenticated()) {
    const remoteRoutes = await fetch('/api/user/routes')
    remoteRoutes.forEach(route => router.addRoute(route))
  }

  app.use(router)
  return router
}
```

### Q4：路由守卫中 next() 函数和 return 的区别？

Vue Router 3 使用 `next()` 三参数风格，容易导致忘记调用或被多次调用的问题。Vue Router 4 改为**基于 return 的统一风格**：

- 不 return 或 return `undefined`：放行
- return `RouteLocation` 或字符串：重定向到该地址
- return `false`：取消当前导航
- 抛出错误：导航失败

```typescript
// ✅ Vue Router 4 推荐写法
router.beforeEach((to, from) => {
  if (!isLoggedIn() && to.name !== 'login') {
    return '/login'  // 清晰的重定向
  }
  // 不 return 表示放行
})

// ❌ Vue Router 3 的旧写法（不再推荐）
router.beforeEach((to, from, next) => {
  if (!isLoggedIn() && to.name !== 'login') {
    next('/login')
  } else {
    next()
  }
})
```

### Q5：路由变化时，Vue 组件是如何更新的？

这个过程涉及三个关键步骤：

1. **路由解析与守卫管线**：确认导航最终被批准
2. **响应式更新**：`router.currentRoute.value` 被赋值为新的路由位置，这是一个 `shallowRef`
3. **RouterView 重新渲染**：`RouterView` 组件的渲染函数读取 `currentRoute.value.matched`，根据当前 `depth` 选取对应的组件记录，渲染正确的组件

由于 `RouterView` 使用了 Vue 的渲染函数而非模板，它能在不创建额外包裹元素的情况下完成组件切换，并且配合 `<Transition>` 可以实现动画效果。

### Q6：什么是 RouteRecordMatcher？它是如何工作的？

`RouteRecordMatcher` 是路由匹配的核心数据结构。每个路由记录对应一个匹配器，匹配器持有：

- **编译后的正则**：用于快速检测路径是否匹配
- **参数键名列表**：用于从匹配结果中提取命名参数
- **父子关系**：用于嵌套路由的链式匹配
- **评分值**：用于排序，确保最精确的路由优先匹配

匹配器在 `addRoute` 时按 `insertMatcher` 插入排序好的数组，保证**最精确的匹配最先被命中**。

## 总结与扩展

Vue Router 作为 Vue 生态的官方路由方案，其核心设计可以用一张图概括：

```
用户操作 → URL变化 → 路由解析 → 守卫管线 → 组件更新
   ↑                                              │
   └──────────────────────────────────────────────┘
                   响应式驱动
```

从 Hash/History 模式的选择到守卫管线的实现，从动态路由的添加到复杂的匹配算法，Vue Router 的设计充分体现了**可预测性**与**声明式**的理念。它与 Vue 的响应式系统深度绑定，使得路由变化后视图的更新完全自动化，开发者只需关注业务逻辑。

### 进一步学习的路径

1. **源码阅读**：从 `createRouter` 入手，追踪 `push` → `navigate` → `resolve` → `update` 的完整链路。Vue Router 4 的源码用 TypeScript 重写，类型清晰，非常适合学习。
2. **性能优化**：理解路由匹配算法后，可以优化大型应用的路由配置——将精确路径排在通用路径前面，避免大量通配路由。
3. **扩展阅读**：对比 React Router v6 的设计——React Router 使用更加声明式的嵌套路由（`<Outlet />` 组件），Vue Router 则对 `path` 做了更多约定式的编译优化。
4. **Vite 与路由懒加载**：Vue Router 4 与 Vite 深度配合，`() => import()` 天然支持代码分割，每个路由页面打包为独立的 chunk。

### 参考资源

- [Vue Router 官方文档](https://router.vuejs.org/)
- [Vue Router GitHub 仓库](https://github.com/vuejs/router)
- [path-to-regexp](https://github.com/pillarjs/path-to-regexp)
- [MDN: History API](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
- [MDN: hashchange 事件](https://developer.mozilla.org/en-US/docs/Web/API/Window/hashchange_event)
