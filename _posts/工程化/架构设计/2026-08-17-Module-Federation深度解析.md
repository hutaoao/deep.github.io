---
layout: post
title: "Module Federation深度解析"
date: 2026-08-17 00:00:00 +0800
categories: ["工程化", "架构设计"]
tags: [Module Federation, Webpack, 远程模块, 联邦模块]
---

## 一句话概括

Module Federation（模块联邦）是 Webpack 5 的核心特性，它让多个**独立构建**的应用在**运行时**共享彼此的模块——不用发 npm 包、不用对齐版本、甚至不用关心对方用什么技术栈。它把"代码共享"从"构建时静态打包"升级成了"运行时动态加载"，是微前端的又一种实现路径。

## 核心知识点

### 1. 基础配置：exposes 与 remotes

Module Federation 的核心配置就三个概念：`name`（联邦名）、`exposes`（暴露的模块）、`remotes`（消费的远程模块）：

```javascript
// ===== 应用 A：暴露模块（远程方）=====
// webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  output: { publicPath: 'http://localhost:3001/' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'products_app',              // 全局唯一名
      filename: 'remoteEntry.js',        // 远程入口文件
      exposes: {
        './ProductCard': './src/components/ProductCard.jsx',
      },
      shared: { react: { singleton: true } },
    }),
  ],
};

// ===== 应用 B：消费远程模块（宿主方）=====
// webpack.config.js
new ModuleFederationPlugin({
  name: 'host_app',
  remotes: {
    products: 'products_app@http://localhost:3001/remoteEntry.js',
  },
  shared: { react: { singleton: true } },
});
```

### 2. 运行时消费远程模块

配置好后，宿主应用可以像 `import` 本地模块一样 `import` 远程模块：

{% raw %}
```javascript
// 宿主应用里直接 import 远程模块
import ProductCard from 'products/ProductCard'; // 'products' 是 remotes 里定义的 key

function Home() {
  return <ProductCard product={{ id: 1, name: 'iPhone' }} />;
}
// Webpack 在运行时从 http://localhost:3001/remoteEntry.js 加载该模块
```
{% endraw %}

### 3. shared：共享依赖避免重复打包

`shared` 是最容易踩坑的配置，它的作用是让 React/Vue 这类公共依赖在整个联邦里只加载一份：

```javascript
shared: {
  react: {
    singleton: true,        // 全局只允许一份 React 实例
    requiredVersion: '^18.0.0', // 版本约束
    eager: false,            // 非急加载（用到时才加载）
  },
  'react-dom': { singleton: true },
}
// singleton: true 的意义：多个应用如果各自打包 React，
// 会出现"多个 React 实例"导致 hooks 报错，singleton 强制共享一份
```

### 4. 与微前端框架的本质区别

Module Federation 和 qiankun/single-spa 是两种思路：前者是**模块级**共享（运行时加载模块），后者是**应用级**调度（路由驱动挂载卸载）：

```javascript
// qiankun：应用级，按路由切换整个子应用
registerMicroApp('app1', { entry: '//cdn/app1', activeWhen: '/app1' });

// Module Federation：模块级，在组件里直接消费远程模块
const RemoteButton = React.lazy(() => import('app1/Button')); // 只加载一个组件
```

## 其实你每天都在用

1. **微前端里的共享组件库**：A 团队维护的"用户卡片"组件，B 团队不用 npm install，直接 `import 'shared/UserCard'` 就能用，改了立刻生效——这就是模块联邦的日常。
2. **设计系统 / 组件平台的运行时下发**：公司统一的按钮、表单组件通过 remoteEntry 下发，所有业务应用引用同一份，风格统一且实时更新。
3. **灰度发布单个组件**：不重新发版整个应用，只切换远程模块的地址，就能让部分用户看到新版组件——这是模块联邦独有的灵活度。
4. **大型 SPA 的"拆包共享"**：主应用和多个子应用共享同一份 React 运行时（`singleton`），避免每个 bundle 都带 100KB 的 React。
5. **微服务的"前端镜像"**：就像后端服务通过 RPC 互相调用，前端应用通过 remoteEntry 互相"调用模块"，本质都是运行时协作而非构建时耦合。

## 常见误解（FAQ）

**❌ 误区一：Module Federation 是 Webpack 专属，别的构建工具用不了。**
错误。Webpack 5 是原生实现，但 Rspack（字节的构建工具）也实现了兼容的 Module Federation，Vite 则有 `@originjs/vite-plugin-federation` 等社区方案。它不是 Webpack 独有，而是"运行时模块共享"这个标准思路。

**❌ 误区二：`shared` 配置了 `singleton: true` 就万事大吉。**
错误。`singleton` 只保证"运行时只加载一份"，但如果两个应用的 React 版本不兼容（如 18 和 19），强制共享会直接报错或出现诡异 bug。`requiredVersion` 的版本协商和 fallback 策略同样关键，版本对齐是团队治理问题，不是配置能一劳永逸的。

**❌ 误区三：远程模块可以随意循环依赖。**
错误。A 暴露模块给 B、B 又暴露模块给 A 的循环依赖会导致加载死锁或初始化顺序混乱。设计联邦拓扑时要避免环形依赖，通常建议"单向暴露"或"公共模块下沉到独立联邦"。

**❌ 误区四：Module Federation 能完全替代微前端框架。**
错误。它俩解决不同层次的问题：模块联邦是"模块级共享"，适合共享组件/工具函数；微前端框架是"应用级调度"，适合整个业务域的独立部署。两者常被组合使用（微前端基座 + 模块联邦共享依赖），而非互相替代。

## 一句话总结

Module Federation 把"代码复用"从 npm 包的"构建时耦合"解放成了"运行时按需加载"——它的威力在于独立构建 + 运行时合体，但代价是版本治理和依赖拓扑需要团队刻意设计，否则会从"解药"变成"新坑"。
