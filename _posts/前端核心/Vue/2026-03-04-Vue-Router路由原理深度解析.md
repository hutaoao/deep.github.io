---
layout: post
title: "Vue Router路由原理"
date: 2026-03-04
tags: [Vue3, Vue Router, 路由, SPA]
categories: [前端核心, Vue]
---

## 一句话概括

Vue Router 的核心是**把 URL 变化映射为组件切换**——通过 Hash 或 History 两种模式劫持浏览器地址变化，用路由匹配器（matcher）算出当前 URL 对应哪棵组件树，再由 `<router-view>` 渲染出来，中间经过一条可拦截的导航守卫管线。

## 核心知识点

### 1. Hash 模式 vs History 模式

```typescript
// Hash 模式 —— 监听 hashchange
// URL: /#/user/123，hash 变化不触发服务器请求
window.addEventListener('hashchange', () => {
  const path = location.hash.slice(1) // '/user/123'
  router.push(path)
})

// History 模式 —— 监听 popstate + pushState
// URL: /user/123，跟普通 URL 一样
// 但需要服务器配置 fallback：所有路由返回 index.html
window.addEventListener('popstate', () => {
  router.push(location.pathname)
})

// router.push 内部调用：
history.pushState({}, '', '/user/123')
// pushState 本身不触发 popstate，手动触发路由更新
```

**选择建议**：History 模式 URL 干净（没有 `#`），适合面向用户的 C 端应用，但需要后端配置 fallback；Hash 模式零配置部署，管理后台够用。

### 2. 路由匹配与嵌套

Vue Router 会把路由配置编译成一棵**路由记录树（RouteRecord）**，匹配时深度遍历找最精确的匹配项。

```typescript
const routes = [
  {
    path: '/user/:id',
    component: User,
    children: [
      { path: 'profile', component: UserProfile },  // 匹配 /user/:id/profile
      { path: 'posts', component: UserPosts },       // 匹配 /user/:id/posts
    ],
  },
]

// router.resolve('/user/42/profile') 返回:
// { matched: [User路由记录, UserProfile路由记录] }
// 匹配到的组件在 <router-view> 中按层级嵌套渲染
```

### 3. 动态路由与路由参数

```typescript
// 定义
{ path: '/user/:id(\\d+)', component: User }

// 组件内获取（两种方式）
import { useRoute } from 'vue-router'
const route = useRoute()
// route.params.id   →  "42"
// route.query.page  →  "1"

// 参数变化但组件不重新创建时用 watch
watch(() => route.params.id, (newId) => {
  fetchUser(newId)
})
```

### 4. 导航守卫管线

完整流程：`beforeEach → beforeEnter → beforeRouteEnter → resolve → afterEach`。任何守卫中 `return false` 或 `next(false)` 都能取消导航。

```typescript
// 全局前置守卫 —— 最常见的「登录拦截」
router.beforeEach((to, from) => {
  const isLoggedIn = useAuthStore().token
  if (to.meta.requiresAuth && !isLoggedIn) {
    return '/login'  // 重定向到登录页
  }
})

// 路由独享守卫
{ path: '/admin', component: Admin, beforeEnter: () => { /* ... */ } }

// 组件内守卫（在 setup 里用）
import { onBeforeRouteLeave } from 'vue-router'
onBeforeRouteLeave((to, from) => {
  if (hasUnsavedChanges.value) {
    return confirm('有未保存的修改，确定离开？')
  }
})
```

### 5. 路由懒加载

```javascript
// ❌ 同步导入 —— 全部打在首屏包里
import UserProfile from './UserProfile.vue'

// ✅ 动态 import —— 访问 /user 时才加载
const UserProfile = () => import('./UserProfile.vue')

// 配合 webpackChunkName 控制分包
const UserProfile = () => import(/* webpackChunkName: "user" */ './UserProfile.vue')
```

Vite/Rollup 里 `/* webpackChunkName */` 不生效，用文件路径自动命名。懒加载后的组件在首次访问前完全不加载，是首屏性能优化的第一刀。

## 「其实你每天都在用」

1. **登录拦截**：`router.beforeEach` 检查 token，没登录跳 `/login`——几乎是每个后台系统的第一个功能。
2. **面包屑导航**：`route.matched` 拿到层级路由记录数组，渲染 `首页 > 用户管理 > 编辑`。
3. **页面标题更新**：`router.afterEach` 里 `document.title = to.meta.title`，Seo 和浏览器标签页标题同步。
4. **keep-alive + 路由元信息**：`route.meta.keepAlive` 控制组件缓存，返回列表不丢状态。
5. **滚动行为**：`scrollBehavior` 让从列表切到详情再返回时停在原来位置，不用从头滚。

## 常见误解（FAQ）

**❌ 误区：History 模式部署到 Nginx 不用额外配置。** 必须配置 `try_files $uri $uri/ /index.html`，否则刷新非根路径（如 `/user/123`）时 Nginx 会尝试找这个物理文件找不到返回 404。这是 History 模式最常见的线上故障。

**❌ 误区：`route.params` 变化时组件会重新创建。** 默认不会。`/user/1` → `/user/2` 走了同组件匹配，Vue 复用同一实例。需要用 `watch` 监听参数变化或在 `beforeRouteUpdate` 里处理，或者给 `<router-view>` 加 `:key="$route.fullPath"` 强制重建。

**❌ 误区：导航守卫中 `next` 必须调用。** Vue Router 4 已经可选 `next`——你可以直接 `return path` 来重定向，或者 `return false` 取消导航。老代码里的 `next()` 仍然能用，但新代码推荐 return 风格。

**❌ 误区：`router.push` 和 `router.replace` 只是浏览器历史记录的差别。** 还有导航守卫和 `scrollBehavior` 的差异。`replace` 不触发 `scrollBehavior`（因为不算页面进入），如果你希望替换后也滚到顶部，需要在守卫里手动处理。

## 一句话总结

Vue Router 本质就是三件事——URL 映射表、守卫管线做权限控制、`<router-view>` 做动态渲染——把「页面切换」从服务端搬到了浏览器，换来无白屏的流畅体验。
