---
layout: post
title: "single-spa原理深度解析"
description: >
  single-spa 是微前端的路由调度引擎，不负责沙箱与样式隔离，只监听 URL 变化决定哪个子应用挂载卸载。
  用应用注册、路由劫持、生命周期契约三件套以极小体积奠定路由驱动微前端的哲学，是 qiankun 等框架的共同底座。
date: 2026-08-16 00:00:00 +0800
categories: ["工程化", "架构设计"]
tags: [single-spa, 应用注册, 路由劫持, 生命周期]
---

## 一句话概括

single-spa 是微前端的"路由调度引擎"——它不负责沙箱隔离和样式隔离，只做一件事：**监听 URL 变化，决定哪个子应用该挂载、哪个该卸载**。它用应用注册、路由劫持、生命周期契约三件套，以极小的体积奠定了"路由驱动微前端"的哲学，是 qiankun、Garfish 等所有后续框架的共同底座。

## 核心知识点

### 1. 生命周期契约：bootstrap/mount/unmount

single-spa 与子应用之间的"合同"是三个生命周期函数，子应用必须导出，single-spa 在合适时机按序调用：

```javascript
// 子应用必须导出生命周期函数
export async function bootstrap() {} // 只执行一次，做初始化
export async function mount(props) {
  // 挂载到 DOM，返回 Promise
  ReactDOM.createRoot(props.container).render(<App />);
}
export async function unmount(props) {
  // 卸载，清理 DOM 和副作用
  ReactDOM.unmountComponentAtNode(props.container);
}
// single-spa 不关心子应用内部用什么框架，只要导出这三个函数
```

### 2. 路由劫持：监听一切 URL 变化

single-spa 劫持 `hashchange`、`popstate`，并重写 `pushState`/`replaceState`，确保任何路由变化都能触发重算：

```javascript
// 路由劫持核心（简化）
window.addEventListener('hashchange', reroute);
window.addEventListener('popstate', reroute);

// 重写 history 方法，让编程式导航也被拦截
const originalPushState = window.history.pushState;
window.history.pushState = function (...args) {
  const result = originalPushState.apply(this, args);
  reroute(); // 手动 pushState 也要触发重算
  return result;
};
```

### 3. 状态机：管理每个应用的生命周期

single-spa 用一个状态机管理每个应用，这是它的核心设计精髓：

```javascript
const NOT_LOADED = 'NOT_LOADED';       // 未加载
const LOADING = 'LOADING_SOURCE_CODE'; // 加载中
const NOT_MOUNTED = 'NOT_MOUNTED';     // 已加载未挂载
const MOUNTED = 'MOUNTED';             // 已挂载
const UNMOUNTING = 'UNMOUNTING';       // 卸载中

// 状态转换：NOT_LOADED → LOADING → NOT_MOUNTED → MOUNTED → UNMOUNTING → NOT_MOUNTED

function getAppChanges(apps) {
  const appsToLoad = [], appsToMount = [], appsToUnmount = [];
  apps.forEach(app => {
    const active = app.activeWhen(window.location);
    switch (app.status) {
      case NOT_LOADED:
        if (active) appsToLoad.push(app); break;
      case NOT_MOUNTED:
        if (active) appsToMount.push(app); break;
      case MOUNTED:
        if (!active) appsToUnmount.push(app); break;
    }
  });
  return { appsToLoad, appsToMount, appsToUnmount };
}
```

### 4. 应用注册与激活规则

通过 `registerApplication` 注册应用，`activeWhen` 决定激活规则：

```javascript
import { registerApplication, start } from 'single-spa';

registerApplication({
  name: 'app1',
  app: () => import('./app1/app1.js'), // 懒加载
  activeWhen: '/app1',                  // 路径匹配
});

registerApplication({
  name: 'app2',
  app: () => import('./app2/app2.js'),
  activeWhen: location => location.pathname.startsWith('/app2'), // 函数匹配
});

start(); // 开始监听路由
```

## 其实你每天都在用

1. **浏览器多标签页切换**：每个标签页就是一个"应用"，切换时 inactive 页面被挂起、active 页面继续运行——single-spa 的 `mount/unmount` 就是这种"按需激活"的网页版。
2. **单页应用里的路由守卫**：你写的 `vue-router` / `react-router` 的 `beforeEach` 钩子，本质上和 single-spa 的 `activeWhen` 一样，都是"根据 URL 决定做什么"。
3. **后台系统的菜单懒加载**：点开某个菜单才加载对应模块的代码（`() => import()`），single-spa 的按需 `loadApp` 就是这个思路的框架化。
4. **微前端基座的路由分发**：你访问 `/order` 进入订单模块、访问 `/user` 进入用户模块，背后的基座就是用 single-spa 的调度逻辑决定谁挂载谁卸载。
5. **qiankun / Garfish 这些框架的调度层**：它们内部都跑着 single-spa 那套状态机，你"感觉不到"是因为被封装成了更易用的 API。

## 常见误解（FAQ）

**❌ 误区一：single-spa 和 qiankun 是并列的两种微前端方案。**
错误。是"地基"和"房子"的关系——single-spa 只做路由调度，qiankun 在它之上加了沙箱隔离、HTML Entry 等。说"二选一"不准确，qiankun 内部就依赖 single-spa。

**❌ 误区二：single-spa 能解决子应用的样式和变量冲突。**
错误。single-spa 明确不处理隔离，JS 全局变量、CSS 样式污染都需要你自己解决（或用 qiankun 这类增强框架）。这是 single-spa 最常被误解的一点。

**❌ 误区三：子应用必须打包成 UMD 才能用 single-spa。**
错误。UMD 是 JS Entry 时代的常见做法，但 single-spa 真正要求的是"导出生命周期函数"。用 ES Module + `() => import()` 的懒加载方式同样可以接入，只是需要处理模块格式（通常配合 `systemjs` 或打包工具）。

**❌ 误区四：`activeWhen` 只能匹配路径字符串。**
错误。`activeWhen` 可以是字符串、前缀字符串、路径数组，也可以是一个接收 `location` 返回布尔值的函数——函数形式最灵活，可以做任意复杂的激活逻辑（如"登录用户才激活"）。

## 一句话总结

single-spa 用"注册 + 路由劫持 + 状态机"三件套，回答了微前端最核心的调度问题——"谁该在什么时候出现"，它故意不做隔离，反而让它成为所有微前端框架共同信任的底层基石。
