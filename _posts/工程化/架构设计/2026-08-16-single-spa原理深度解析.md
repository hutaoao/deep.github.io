---
layout: post
title: "single-spa原理深度解析"
date: 2026-08-16 00:00:00 +0800
categories: ["工程化", "架构设计"]
tags: [single-spa, 应用注册, 路由劫持, 生命周期]
math: true
mermaid: true
---

## 一句话概括

single-spa 是微前端路由层的"元框架"——它不提供沙箱隔离和样式隔离，只负责根据 URL 路由规则决定加载/卸载哪个子应用。作为微前端的祖师爷框架，single-spa 通过应用注册、路由劫持（hash + history）、和生命周期契约三个核心机制，用不到 5KB 的代码量解决了"什么样的应用该在什么时候出现"这个核心调度问题。本文从源码层面解析 single-spa 的路由系统和生命周期引擎。

## 背景与意义

如果说 qiankun 是"开箱即用的微前端解决方案"，那 single-spa 就是"给你造房子的地基和砖头"。它只做一件事：监听路由变化，决定子应用的挂载和卸载，把 JS 隔离和 CSS 隔离的问题留给使用者自行解决。理解 single-spa 更重要的是理解它背后"路由驱动"的微前端哲学——这也是后来所有微前端框架（qiankun、Garfish、Micro App）的共同基础。面试中，single-spa 相关的题目通常聚焦于它的核心机制和与 qiankun 的对比："single-spa 和 qiankun 有什么区别？""single-spa 的 JS Entry 和 HTML Entry 有什么本质不同？"

## 概念与定义

### 应用注册 (Register Application)
将子应用注册到 single-spa 中，提供生命周期函数和激活规则（activeWhen）。注册后 single-spa 会管理该应用的状态。

### 路由劫持 (Routing Interception)
single-spa 劫持 `hashchange`、`popstate` 事件，并重写 `history.pushState` 和 `history.replaceState`，使得任何 URL 变化都能触发路由重算。

### 生命周期契约 (Lifecycle Contract)
子应用必须导出 `bootstrap`、`mount`、`unmount` 三个生命周期函数（可选 `update`），single-spa 在合适的时机按顺序调用这些函数。

### 应用状态机 (Application Status)
single-spa 维护每个应用的状态（NOT_LOADED → LOADING → LOADED → MOUNTING → MOUNTED → UNMOUNTING → UNMOUNTED → SKIP_BECAUSE_BROKEN 等）。

## 核心知识点拆解

### 1. 应用注册与状态机

single-spa 的核心是一个精心设计的状态机，管理着每个子应用的生命周期：

```javascript
// single-spa 应用状态机源码级实现（简化版）
// 定义应用的所有可能状态
const NOT_LOADED = 'NOT_LOADED';         // 初始状态：尚未加载
const LOADING_SOURCE_CODE = 'LOADING';   // 正在加载子应用的源代码
const NOT_MOUNTED = 'NOT_MOUNTED';       // 已加载但未挂载
const MOUNTED = 'MOUNTED';               // 已挂载到 DOM
const UNMOUNTING = 'UNMOUNTING';         // 正在卸载
const LOAD_ERROR = 'LOAD_ERROR';         // 加载失败
const SKIP_BECAUSE_BROKEN = 'BROKEN';    // 出错后跳过

// 状态转换规则：
// NOT_LOADED → LOADING_SOURCE_CODE → NOT_MOUNTED → MOUNTED
//                                                              ↓
//                    LOAD_ERROR ← 任意状态 → SKIP_BECAUSE_BROKEN
//                    MOUNTED → UNMOUNTING → NOT_MOUNTED

// 应用注册函数
function registerApplication(appNameOrConfig, appOrLoadApp, activeWhen, customProps) {
  // 解析参数（支持多种调用方式）
  const registration = sanitizeArguments(
    appNameOrConfig, appOrLoadApp, activeWhen, customProps
  );

  const { appName, loadApp, activeWhen, customProps } = registration;

  // 创建应用对象
  const app = {
    name: appName,
    status: NOT_LOADED,          // 初始状态
    loadApp,                     // 加载函数（返回 Promise）
    activeWhen,                  // 激活规则函数
    customProps,                 // 自定义属性
    bootstrap: null,             // 加载后设置
    mount: null,
    unmount: null,
    unload: null,
    timeouts: {
      bootstrap: { global: 4000, app: 1000 },
      mount: { global: 3000, app: 1000 },
      unmount: { global: 3000, app: 1000 },
      unload: { global: 3000, app: 1000 },
    },
    loadPromise: null,           // 加载 promise（用于去重）
  };

  // 添加到应用注册表
  apps.push(app);

  // 立即重新路由（检查新的子应用是否需要激活）
  reroute();
}

// 判断应用是否应该激活
function shouldBeActive(app) {
  try {
    return app.activeWhen(window.location);
  } catch (e) {
    console.error(`[single-spa] 应用 "${app.name}" 的 activeWhen 函数异常:`, e);
    return false;
  }
}

// 获取需要挂载和需要卸载的应用列表
function getAppChanges() {
  const appsToUnmount = [];  // 当前 MOUNTED 但 shouldBeActive() === false
  const appsToMount = [];    // 当前 NOT_MOUNTED 但 shouldBeActive() === true
  const appsToLoad = [];     // 当前 NOT_LOADED 且 shouldBeActive() === true

  apps.forEach((app) => {
    const appShouldBeActive = shouldBeActive(app);

    switch (app.status) {
      case LOAD_ERROR:
      case SKIP_BECAUSE_BROKEN:
      case NOT_LOADED:
      case LOADING_SOURCE_CODE:
        if (appShouldBeActive) {
          appsToLoad.push(app);
        }
        break;
      case NOT_MOUNTED:
        if (appShouldBeActive) {
          appsToMount.push(app);
        }
        break;
      case MOUNTED:
        if (!appShouldBeActive) {
          appsToUnmount.push(app);
        }
        break;
    }
  });

  return { appsToUnmount, appsToMount, appsToLoad };
}

// 使用示例
import { registerApplication, start } from 'single-spa';

// 方式 1: 加载函数返回生命周期对象
registerApplication({
  name: 'app1',
  app: () => import('./app1/app1.js'),  // 加载函数
  activeWhen: '/app1',                   // 激活规则
  customProps: { domElement: '#app1-container' },
});

// 方式 2: 直接传入生命周期对象
registerApplication({
  name: 'app2',
  app: {
    bootstrap: () => Promise.resolve(),
    mount: () => Promise.resolve(),
    unmount: () => Promise.resolve(),
  },
  activeWhen: ['/app2', '/other'],  // 多个路径
});

// 方式 3: 使用函数判断激活条件
registerApplication({
  name: 'app3',
  app: () => System.import('app3'),
  activeWhen: (location) => {
    return location.pathname.startsWith('/app3') && 
           !location.pathname.includes('/admin');
  },
});

// 启动 single-spa
start();
```

### 2. 路由劫持机制

single-spa 的路由劫持是其最核心但最容易被忽视的机制：

```javascript
// single-spa 路由劫持源码实现
(function () {
  // ===== 原始方法保存 =====
  const originalPushState = window.history.pushState;
  const originalReplaceState = window.history.replaceState;
  const originalAddEventListener = window.addEventListener;
  const originalRemoveEventListener = window.removeEventListener;

  // ===== 事件监听器管理 =====
  let popStateListeners = [];
  let hashChangeListeners = [];
  let capturedEventListeners = { popstate: [], hashchange: [] };

  // ===== 1. 劫持 popstate 事件 =====
  // 拦截 addEventListener 对 popstate 的注册
  window.addEventListener = function (eventType, listener, options) {
    if (eventType === 'popstate') {
      // single-spa 内部处理：使用自己的列表管理
      popStateListeners.push({ listener, options });
      // 不注册到真实 window 上
      return;
    }
    // 其他事件正常注册
    originalAddEventListener.call(window, eventType, listener, options);
  };

  window.removeEventListener = function (eventType, listener) {
    if (eventType === 'popstate') {
      popStateListeners = popStateListeners.filter(
        (entry) => entry.listener !== listener
      );
      return;
    }
    originalRemoveEventListener.call(window, eventType, listener);
  };

  // ===== 2. 劫持 history.pushState / replaceState =====
  window.history.pushState = function (data, title, url) {
    // 调用原始方法（浏览器地址栏会更新）
    originalPushState.call(this, data, title, url);
    
    // 触发 single-spa 的重路由
    // 注意：pushState 本身不会触发 popstate 事件
    urlReroute(new PopStateEvent('popstate'));
  };

  window.history.replaceState = function (data, title, url) {
    originalReplaceState.call(this, data, title, url);
    urlReroute(new PopStateEvent('popstate'));
  };

  // ===== 3. 劫持浏览器原生的 popstate（前进后退按钮触发） =====
  // 为 window 注册一个真正的 popstate 监听器
  // 当用户点击浏览器前进/后退时触发
  originalAddEventListener.call(window, 'popstate', (event) => {
    // single-spa 内部的路由处理
    urlReroute(event);
    
    // 通知用户注册的 popstate 监听器
    popStateListeners.forEach((entry) => {
      try {
        entry.listener.call(window, event);
      } catch (e) {
        console.error('[single-spa] popstate 监听器异常:', e);
      }
    });
  });

  // ===== 4. 拦截 hashchange 事件 =====
  window.addEventListener('hashchange', (event) => {
    // 触发重路由
    urlReroute(event);
    
    // 通知用户注册的 hashchange 监听器
    hashChangeListeners.forEach((entry) => {
      try {
        entry.listener.call(window, event);
      } catch (e) {
        console.error('[single-spa] hashchange 监听器异常:', e);
      }
    });
  });

  // ===== 5. 核心路由函数 =====
  let isRerouting = false;
  let appChangeUnderway = false;
  let peopleWaitingOnAppChange = [];

  function urlReroute(event) {
    // 防止重入（如果已经在路由中，排队等待）
    if (isRerouting) {
      return new Promise((resolve, reject) => {
        peopleWaitingOnAppChange.push({ resolve, reject, event });
      });
    }

    isRerouting = true;

    try {
      // 获取需要装入和卸载的应用
      const { appsToUnmount, appsToMount, appsToLoad } = getAppChanges();
      
      // 1. 先卸载不需要的应用
      if (appsToUnmount.length > 0) {
        unmountApps(appsToUnmount);
      }
      
      // 2. 加载需要激活的应用
      if (appsToLoad.length > 0) {
        loadApps(appsToLoad);
      }
      
      // 3. 挂载已经加载好的应用
      if (appsToMount.length > 0) {
        mountApps(appsToMount);
      }
    } finally {
      isRerouting = false;
      
      // 处理排队中的请求
      if (peopleWaitingOnAppChange.length > 0) {
        const next = peopleWaitingOnAppChange.shift();
        next.resolve(urlReroute(next.event));
      }
    }
  }

  // 外部可以通过 start() 启动监听
  // start 之后，single-spa 才真正开始劫持路由
})();
```

### 3. 生命周期调度引擎

生命周期调度是 single-spa 中利用 Promise 链实现的状态转换引擎：

```javascript
// single-spa 生命周期调度引擎
// 核心：以可重试（retryable）的 Promise 链驱动状态机转换

// ===== 生命周期标志配置 =====
const lifeCycleFns = ['bootstrap', 'mount', 'unmount', 'update'];
const defaultTimeouts = {
  bootstrap: { global: 4000, app: 1000 },
  mount: { global: 3000, app: 1000 },
  unmount: { global: 3000, app: 1000 },
  unload: { global: 3000, app: 1000 },
};

// ===== 加载应用（NOT_LOADED → NOT_MOUNTED）=====
async function loadApp(app) {
  // 状态验证
  if (app.status !== NOT_LOADED && app.status !== LOAD_ERROR) {
    return app;
  }

  app.status = LOADING_SOURCE_CODE;

  try {
    // 调用加载函数获取子应用暴露的生命周期
    const userApp = await reasonableTime(
      app.loadApp(),
      `loading application ${app.name}`,
      app.timeouts.load
    );

    // 验证生命周期接口
    if (!userApp.bootstrap || !userApp.mount || !userApp.unmount) {
      throw new Error(
        `子应用 ${app.name} 必须导出 bootstrap, mount, unmount 方法`
      );
    }

    // 将子应用暴露的生命周期绑定到 app 对象
    app.bootstrap = flattenLifecyclesArray(userApp.bootstrap);
    app.mount = flattenLifecyclesArray(userApp.mount);
    app.unmount = flattenLifecyclesArray(userApp.unmount);
    app.update = flattenLifecyclesArray(userApp.update);

    // 状态转换
    app.status = NOT_MOUNTED;
    return app;
  } catch (error) {
    app.status = LOAD_ERROR;
    throw error;
  }
}

// ===== 挂载应用（NOT_MOUNTED → MOUNTED）=====
async function mountApp(app) {
  if (app.status !== NOT_MOUNTED) {
    return app;
  }

  app.status = MOUNTING;

  try {
    // 提供自定义 props（包含基座传递的信息）
    const props = getProps(app);
    
    // 调用子应用的 mount 方法
    await performLifecycle(
      app.mount,
      props,
      app,
      'mount'
    );

    app.status = MOUNTED;
    return app;
  } catch (error) {
    app.status = LOAD_ERROR;
    throw error;
  }
}

// ===== 卸载应用（MOUNTED → NOT_MOUNTED）=====
async function unmountApp(app) {
  if (app.status !== MOUNTED) {
    return app;
  }

  app.status = UNMOUNTING;

  try {
    const props = getProps(app);
    
    await performLifecycle(
      app.unmount,
      props,
      app,
      'unmount'
    );

    app.status = NOT_MOUNTED;
    return app;
  } catch (error) {
    app.status = LOAD_ERROR;
    throw error;
  }
}

// ===== 生命周期执行函数 =====
// 支持多函数串联（子应用可以导出数组）
async function performLifecycle(fns, props, app, lifecycleName) {
  // 确保生命周期函数被包装为数组
  const lifecycles = Array.isArray(fns) ? fns : [fns];
  
  for (const lifecycle of lifecycles) {
    if (lifecycle.length === 0) {
      // 无参数的生命周期函数
      await reasonableTime(
        lifecycle(),
        `${app.name} ${lifecycleName}`,
        app.timeouts[lifecycleName]
      );
    } else {
      // 带 props 的生命周期函数
      await reasonableTime(
        lifecycle(props),
        `${app.name} ${lifecycleName}`,
        app.timeouts[lifecycleName]
      );
    }
  }
}

// ===== 超时控制工具 =====
function reasonableTime(promiseOrFn, description, timeouts) {
  return new Promise((resolve, reject) => {
    let finished = false;
    
    // 应用级超时
    const appTimeout = setTimeout(() => {
      if (!finished) {
        finished = true;
        reject(new Error(`${description} 超时 (${timeouts.app}ms)`));
      }
    }, timeouts.app);
    
    // 全局级超时
    const globalTimeout = setTimeout(() => {
      if (!finished) {
        finished = true;
        reject(new Error(`${description} 全局超时 (${timeouts.global}ms)`));
      }
    }, timeouts.global);

    Promise.resolve(
      typeof promiseOrFn === 'function' ? promiseOrFn() : promiseOrFn
    ).then((result) => {
      if (!finished) {
        finished = true;
        clearTimeout(appTimeout);
        clearTimeout(globalTimeout);
        resolve(result);
      }
    }).catch((error) => {
      if (!finished) {
        finished = true;
        clearTimeout(appTimeout);
        clearTimeout(globalTimeout);
        reject(error);
      }
    });
  });
}

// ===== 辅助函数 =====
function getProps(app) {
  const customProps = typeof app.customProps === 'function'
    ? app.customProps(app.name, window.location)
    : app.customProps;
  
  return {
    ...customProps,
    name: app.name,
    // single-spa 内置 API
    singleSpa: {
      mountParcel: mountParcel, // 子应用挂载自己的 parcel
    },
  };
}

function flattenLifecyclesArray(lifecycleOrArray) {
  if (!lifecycleOrArray) return [];
  return Array.isArray(lifecycleOrArray) 
    ? lifecycleOrArray 
    : [lifecycleOrArray];
}
```

### 4. Parcel 系统

Parcel 是 single-spa 中一个强大的概念——它是可以独立挂载/卸载的 UI 组件粒度的微前端单元：

```javascript
// single-spa Parcel 系统
// Parcel 与子应用的区别：Parcel 不由路由驱动，由父组件手动控制

// ===== 创建 Parcel =====
function createParcel(config) {
  const parcel = {
    // Parcel 自己的状态
    status: NOT_LOADED,
    name: config.name || `parcel-${Date.now()}`,
    // 生命周期
    bootstrap: flattenLifecyclesArray(config.bootstrap || defaultBootstrap),
    mount: flattenLifecyclesArray(config.mount),
    unmount: flattenLifecyclesArray(config.unmount),
    update: flattenLifecyclesArray(config.update),
    // 加载函数（可选）
    loadApp: config.app || null,
    // 自定义 props
    customProps: config.customProps || {},
  };

  // Parcel 的公共 API
  return {
    mount: (additionalProps = {}) => mountParcel(parcel, additionalProps),
    unmount: () => unmountParcel(parcel),
    update: (customProps) => updateParcel(parcel, customProps),
    getStatus: () => parcel.status,
  };
}

// ===== Parcel 与子应用的嵌套关系 =====
// 子应用可以在 mount 中挂载自己的 Parcel
function mount(props) {
  const { mountParcel, domElement } = props;
  
  // 挂载一个 React 组件作为 Parcel
  const parcel = mountParcel({
    name: 'react-header',
    mount: () => {
      return new Promise((resolve) => {
        ReactDOM.render(<Header />, domElement, resolve);
      });
    },
    unmount: () => {
      return new Promise((resolve) => {
        ReactDOM.unmountComponentAtNode(domElement, resolve);
      });
    },
  });
  
  // Parcel 可以被手动控制
  // parcel.unmount() // 在子应用内部按需卸载
}

// Parcel 嵌套场景（3 层）：
// 主应用 (single-spa 宿主)
//   └── 购物车子应用 (mount 时挂载)
//         ├── 商品列表组件 (React Parcel)
//         ├── 购物车小计 (Vue Parcel)
//         └── 推荐模块 (Svelte Parcel)
```

## 实战案例：从零搭建 single-spa 微前端

```javascript
// ============ 1. 主应用入口 ============
// main-app/src/index.js

import { registerApplication, start } from 'single-spa';
import { constructRoutes, constructApplications } from 'single-spa-layout';

// 方案 A: 手动注册（推荐）
registerApplication({
  name: 'react-nav',
  app: () => System.import('react-nav'),  // 使用 SystemJS 或 import()
  activeWhen: '/',
});

registerApplication({
  name: 'app-products',
  app: () => import('./apps/products/products.app.js'),
  activeWhen: (location) => {
    // 仅当路径以 /products 开头且不是 /products/manage 时激活
    return location.pathname.startsWith('/products') &&
           !location.pathname.startsWith('/products/manage');
  },
  customProps: {
    apiBaseUrl: 'https://api.example.com/v1',
  },
});

registerApplication({
  name: 'app-checkout',
  app: () => import('./apps/checkout/checkout.app.js'),
  activeWhen: '/checkout',
  customProps: (name, location) => {
    // 动态 props（每次路由变化时重新计算）
    return {
      isTest: location.search.includes('test=true'),
      user: getUserFromCookie(),
    };
  },
});

// 启动——设置各种选项
start({
  // URL 变化时强制 reroute
  urlRerouteOnly: true,
  // 控制台日志级别
  warning: {
    timeout: false,           // 不警告超时
    unresolvable: true,       // 警告无法解析的模块
  },
});

// 方案 B: 使用 single-spa-layout（声明式布局）
// main-app/src/index.html
// <template id="app-layout">
//   <nav id="nav-container" />
//   <div id="content">
//     <route path="/products">
//       <application name="app-products"></application>
//     </route>
//     <route path="/checkout">
//       <application name="app-checkout"></application>
//     </route>
//   </div>
// </template>

const routes = constructRoutes(document.querySelector('#app-layout'));
const applications = constructApplications({
  routes,
  loadApp: ({ name }) => import(`./apps/${name}/${name}.app.js`),
});

applications.forEach(registerApplication);
start();

// ============ 2. 子应用适配（React）============
// app-products/src/products.app.js

import React from 'react';
import ReactDOM from 'react-dom';
import rootComponent from './root.component.js';

// 生命周期必须返回 Promise
export const bootstrap = () => {
  console.log('products 应用已加载');
  return Promise.resolve();
};

export const mount = (props) => {
  console.log('products 应用已挂载', props);
  
  const { domElement, name, singleSpa } = props;
  
  // 在指定 DOM 节点渲染
  ReactDOM.render(
    React.createElement(rootComponent, { name, singleSpa }),
    domElement || document.getElementById('products-container')
  );
  
  return Promise.resolve();
};

export const unmount = (props) => {
  console.log('products 应用已卸载');
  
  const { domElement } = props;
  ReactDOM.unmountComponentAtNode(
    domElement || document.getElementById('products-container')
  );
  
  return Promise.resolve();
};

// ============ 3. Webpack 构建配置 ============
// webpack.config.js（子应用专用）

const webpack = require('webpack');
const packageJson = require('./package.json');

module.exports = {
  output: {
    // 关键：必须输出 UMD 格式
    libraryTarget: 'umd',
    library: packageJson.name,
    // 使用唯一的 jsonp 函数名，避免多个子应用冲突
    jsonpFunction: `webpackJsonp_${packageJson.name}`,
    // 告诉 webpack 不在 UMD 内包裹模块代码
    globalObject: 'window',
  },
  externals: {
    // 共享的依赖可以声明为外部依赖
    // 由主应用提供（减少打包体积）
    'react': 'React',
    'react-dom': 'ReactDOM',
    'single-spa': 'singleSpa',
  },
  devServer: {
    // 必须开启 CORS
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
    // 开发环境下作为独立应用时
    historyApiFallback: true,
  },
};

// ============ 4. 子应用 Vue 适配 ============
// app-checkout/src/checkout.app.js

import Vue from 'vue';
import singleSpaVue from 'single-spa-vue';
import App from './App.vue';
import router from './router';

const vueLifecycles = singleSpaVue({
  Vue,
  appOptions: {
    render: (h) => h(App),
    router,
  },
});

export const bootstrap = vueLifecycles.bootstrap;
export const mount = vueLifecycles.mount;
export const unmount = vueLifecycles.unmount;
```

## 底层原理

### single-spa 的"假持久化"设计

single-spa 的 `start()` 函数调用后，它会查抄当前页面的 URL 和已注册的应用列表，但并不会立即挂载应用。这是因为它采用"先注册，后启动"的两阶段设计。在 `registerApplication` 和 `start()` 之间的窗口期，URL 变化不会被 single-spa 感知，因为路由劫持机制尚未激活。

启动后，single-spa 会在第一个 `reroute` 调用时计算应该加载哪些应用。关键的是，`start` 时传入的 `urlRerouteOnly` 参数控制着：是否只有当 URL 变化时才 reroute。如果设置为 `false`，`start()` 会立即 reroute 一次，完成首屏挂载。

### 为什么 single-spa 不提供沙箱？

这是一个架构哲学问题。single-spa 的设计者 Joel Denning 认为："沙箱应该由框架层而非路由层提供。" 他的理由是：浏览器环境本身就是一个巨大的共享沙箱——你无法阻止子应用调用 `document.body.appendChild()`。任何试图"完全隔离"的方案要么带来性能开销（Proxy），要么带来兼容性问题（iframe），要么给用户错误的安全感。因此 single-spa 选择了零信任策略——你应该通过组织规范（Code Review、团队契约）而非技术方案来解决污染问题。

## 高频面试题解析

### 面试题 1：single-spa 和 qiankun 的核心区别是什么？

**答案要点：** single-spa 是一个"微前端路由层"，只负责应用注册、路由调度和生命周期管理，不处理 JS 隔离、CSS 隔离和子应用加载方式的优化。qiankun 基于 single-spa，增加了：1）HTML Entry（子应用无需修改构建配置）；2）JS 沙箱（Proxy/快照隔离）；3）样式隔离（Scoped CSS/Shadow DOM）；4）全局状态通信；5）预加载策略。简言之，single-spa 是引擎，qiankun 是整车。

### 面试题 2：single-spa 如何劫持路由？为什么不用监听所有 URL 变化？

**答案要点：** single-spa 通过三种机制劫持路由：1）重写 history.pushState/replaceState——每当调用时触发 reroute；2）监听 popstate 事件——浏览器前进/后退触发；3）监听 hashchange——hash 路由变化。之所以不监听完整的 URL 变化，是因为浏览器没有提供通用的 URL 变化事件——`hashchange` 只监听到 hash 部分的变化，`popstate` 只监听到历史记录的前进后退。因此必须补充对 pushState/replaceState 的劫持。

### 面试题 3：single-spa 的 activeWhen 写法有哪些？`'/app'` 和 `['/app1', '/app2']` 有什么区别？

**答案要点：** activeWhen 支持三种形式：1）字符串路径前缀——`'/app'` 等价于 `(location) => location.pathname.startsWith('/app')`；2）字符串数组——`['/app1', '/app2']` 表示路径以任一字符串开头即激活（OR 关系）；3）函数——完全自定义，接收 location 参数。注意字符串匹配是基于 startsWith 的，所以 `'/app'` 会匹配 `/app`、`/app/products`、`/application` 等。如果要精确匹配，使用自定义函数。

### 面试题 4：single-spa 的应用超时是如何工作的？为什么不使用 while(true) 轮询？

**答案要点：** single-spa 为每个生命周期阶段设置了两个超时：app 级超时和 global 级超时。app 级超时（如 1000ms）是单个生命周期调用的软限制，global 级超时（如 4000ms）是所有调用的硬限制。超时机制通过 `Promise.race` 实现——在对应时间内未 resolve 就 reject。不使用轮询的原因：1）轮询浪费 CPU；2）Promise.race 自然适配了 Promise 生命周期（bootstrap/mount/unmount 都返回 Promise）；3）轮询无法准确捕获"生命周期尚未完成"这种状态。

### 面试题 5：single-spa 的 Parcel 与子应用有什么区别？什么场景下适合使用 Parcel？

**答案要点：** Parcel 比子应用"更轻"——它不由路由驱动，而是由父组件手动控制生命周期。Parcel 适合在子应用内部嵌入另一个技术栈的小组件。例如：React 子应用中嵌入一个 Vue 组件作为购物车图标，这个 Vue 组件就是一个 Parcel。区别在于：1）Parcel 没有路由激活规则；2）Parcel 可以嵌套（子应用内部挂载子应用内部的 Parcel）；3）Parcel 由开发者手动调用 mount/unmount。

## 总结与扩展

### 知识体系图

```
single-spa 核心
├── 应用注册（registerApplication）
│   ├── appName（唯一标识）
│   ├── loadApp（加载函数 → 返回生命周期）
│   ├── activeWhen（激活规则）
│   │   ├── 字符串前缀（startsWith）
│   │   ├── 字符串数组（OR）
│   │   └── 自定义函数（location → bool）
│   └── customProps（传递给子应用的属性）
├── 路由劫持
│   ├── 重写 history.pushState/replaceState
│   ├── 拦截 popstate 事件
│   ├── 监听 hashchange 事件
│   └── urlReroute（核心调度函数）
├── 状态机
│   ├── NOT_LOADED → LOADING → NOT_MOUNTED
│   ├── NOT_MOUNTED → MOUNTING → MOUNTED
│   ├── MOUNTED → UNMOUNTING → NOT_MOUNTED
│   └── LOAD_ERROR / SKIP_BECAUSE_BROKEN
├── 生命周期契约
│   ├── bootstrap（引导，一次）
│   ├── mount（挂载，每次激活）
│   ├── unmount（卸载，每次离开）
│   └── update（更新，可选）
├── Parcel 系统
│   ├── 手动生命周期控制
│   ├── 组件级微前端
│   └── 支持嵌套
└── 辅助库
    ├── single-spa-react（React 适配）
    ├── single-spa-vue（Vue 适配）
    ├── single-spa-angular（Angular 适配）
    └── single-spa-layout（声明式布局）
```

### 延伸阅读

1. **single-spa 官方源码**: GitHub 搜索 `single-spa/single-spa` — 核心源码仅 ~2000 行
2. **single-spa 官方文档**: single-spa.js.org — 详细的示例和文档
3. **Joel Denning 的技术博客**: 深入理解 single-spa 的设计哲学
4. **single-spa-layout**: 声明式微前端路由配置库
5. **Module Federation vs single-spa 对比**: 两种微前端实现路线的对比分析
