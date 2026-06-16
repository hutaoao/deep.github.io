---
layout: post
title: "Module Federation深度解析"
date: 2026-08-17 00:00:00 +0800
categories: ["工程化", "架构设计"]
tags: [Module Federation, Webpack, 远程模块, 联邦模块]
math: true
mermaid: true
---

## 一句话概括

Module Federation（模块联邦）是 Webpack 5 最重磅的特性，它允许多个独立构建的应用在运行时动态共享和消费彼此的模块——不需要三方包发布、不需要修改子应用构建、甚至不需要知道其他应用的技术栈。它以一种"分布式运行时"的方式实现了微前端的核心目标：独立构建、独立部署、运行时合体。本文从容器应用、远程模块、共享依赖、异步边界等核心概念出发，深入 Module Federation 的源码实现和工程化落地。

## 背景与意义

在 Module Federation 出现之前，前端代码共享只有三条路：发布 npm 包需要走 package.json 版本管理和发布流程；微前端（如 qiankun / single-spa）需要在更高维度（路由）上进行应用拆分；代码复制则是对工程化的亵渎。Module Federation 提供了一个"第四选择"——**在运行时**从一个远程地址加载一个模块，就像 `import` 一个本地模块一样自然。

对于大型项目而言，Module Federation 的影响是革命性的：团队 A 将一个共享组件作为"联邦模块"暴露出来，团队 B 在运行时直接引用，无需 npm install、无需版本对齐、可以灰度切换版本。面试中，Module Federation 的主题通常围绕核心概念和配置展开："Module Federation 的 shared 配置如何工作？""如何用 Module Federation 做微前端？""Module Federation 和 qiankun 的区别是什么？"

## 概念与定义

### 联邦模块 (Federated Module)
一个被暴露（expose）为远程可加载的模块。其他应用可以在运行时通过联邦容器加载它。

### 容器应用 (Container Application)
一个特殊的 Webpack 构建产物，它是"模块的出口"——将本地模块暴露给其他应用，同时也可以从其他应用消费模块。

### 远程模块 (Remote Module)
从容器应用中加载的模块。远程模块在运行时通过异步 `import()` 加载，Webpack 自动处理所有依赖。

### 共享依赖 (Shared Dependencies)
通过 `shared` 配置声明公共依赖（如 React、Vue），Webpack 确保整个应用中只加载一份副本，避免依赖的重复打包。

### 异步边界 (Async Boundary)
当应用从一个联邦应用加载远程模块时，天然形成了一个异步边界。Webpack 在这个边界处插入代码来处理远程模块的加载、缓存和 fallback。

## 核心知识点拆解

### 1. Module Federation 基础配置

一切从配置开始。以下的配置展示了两个独立应用如何通过 Module Federation 实现运行时模块共享：

```javascript
// ===== 应用 A：商品应用（用于暴露共享组件）=====
// apps/products/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  mode: 'production',
  output: {
    publicPath: 'http://localhost:3001/',  // CDN 地址
    uniqueName: 'products_app',
  },
  plugins: [
    new ModuleFederationPlugin({
      // 1. 联邦应用名称（全局唯一）
      name: 'products_app',
      
      // 2. 暴露给其他应用的模块
      exposes: {
        // key: 其他应用引用时使用的路径
        // value: 本地模块的路径
        './ProductCard': './src/components/ProductCard.jsx',
        './ProductList': './src/components/ProductList.jsx',
        './ProductAPI': './src/api/productApi.js',
        './store': './src/store/productStore.js',
      },
      
      // 3. 共享依赖——避免重复打包
      shared: {
        react: {
          singleton: true,          // 全局只加载一份 React
          requiredVersion: '^18.0.0', // 版本约束
          eager: false,             // 非急加载（等用到时再加载）
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-router-dom': {
          singleton: true,
        },
      },
    }),
  ],
};

// ===== 应用 B：订单应用（用于消费商品应用暴露的模块）=====
// apps/orders/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  mode: 'production',
  output: {
    publicPath: 'http://localhost:3002/',
    uniqueName: 'orders_app',
  },
  plugins: [
    new ModuleFederationPlugin({
      name: 'orders_app',
      
      // 1. 指定远程容器应用
      remotes: {
        // key: 应用中使用的别名
        // value: 远程容器应用的地址
        // 格式: 容器名称@远程入口URL
        'products_app': 'products_app@http://localhost:3001/remoteEntry.js',
      },
      
      // 2. 订单应用也可以暴露自己的模块
      exposes: {
        './OrderCard': './src/components/OrderCard.jsx',
      },
      
      // 3. 共享依赖（必须与远程应用的 shared 配置兼容）
      shared: {
        react: {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        'react-dom': {
          singleton: true,
          requiredVersion: '^18.0.0',
        },
        // 共享的类型定义包
        'lodash': {
          singleton: false,  // 不要求单例
        },
      },
    }),
  ],
};
```

### 2. 在代码中消费远程模块

配置完成后，在代码中引用远程模块就像引用 npm 包一样简单：

```javascript
{% raw %}
// ===== 在订单应用中引用商品模块 =====
// apps/orders/src/pages/ProductSelectionPage.jsx

import React, { useState, Suspense } from 'react';

// 动态异步加载远程模块
// Webpack 在构建时识别出 'products_app/ProductCard' 是远程模块
// 自动在编译产物中插入远程加载代码
const ProductCard = React.lazy(() => import('products_app/ProductCard'));
const ProductList = React.lazy(() => import('products_app/ProductList'));

// 加载时的 fallback 组件
function LoadingFallback() {
  return (
    <div className="loading-container">
      <div className="spinner" />
      <p>正在加载商品模块...</p>
    </div>
  );
}

// 错误情况下的 fallback
function ErrorFallback({ error, retry }) {
  return (
    <div className="error-boundary">
      <h3>商品模块加载失败</h3>
      <p>{error.message || '未知错误'}</p>
      <button onClick={retry}>重试</button>
    </div>
  );
}

export default function ProductSelectionPage() {
  const [hasError, setHasError] = useState(false);
  const [error, setError] = useState(null);

  // 重试逻辑
  const handleRetry = () => {
    setHasError(false);
    setError(null);
    // 重新加载组件
    window.location.reload();
  };

  // Error Boundary
  if (hasError) {
    return <ErrorFallback error={error} retry={handleRetry} />;
  }

  return (
    <div className="product-selection">
      <h1>选择商品下单</h1>
      
      {/* 使用远程模块：就像使用本地组件一样 */}
      <Suspense fallback={<LoadingFallback />}>
        <ProductList 
          onSelect={(product) => {
            console.log('选中商品:', product);
            // 处理选中的商品
          }}
          filter={{ inStock: true, maxPrice: 5000 }}
        />
      </Suspense>
    </div>
  );
}

// 另一种方式：直接 import 远程模块的导出
// 使用 Webpack 5 的新语法

// 异步动态导入（更灵活，支持运行时决策）
async function loadRemoteModule() {
  try {
    // Webpack 在构建时替换为远程加载代码
    const { ProductAPI } = await import('products_app/ProductAPI');
    
    // 调用远程模块的 API
    const products = await ProductAPI.getProducts({ 
      page: 1, 
      pageSize: 20,
      category: 'electronics',
    });
    
    return products;
  } catch (error) {
    console.error('远程模块加载失败:', error);
    // fallback 到本地备份数据
    return [];
  }
}

// 也支持静态 import（需要在 webpack config 中设置 eager）
// import { store } from 'products_app/store';
{% endraw %}
```

### 3. Shared 依赖的版本裁决

Module Federation 最精妙的地方在于 shared 的版本调度。当两个应用都声明了 `react: ^18.0.0`，Webpack 会加载最匹配的版本：

```javascript
// Module Federation 共享依赖的版本裁决逻辑（源码分析）
// webpack/lib/container/FallbackModule.js

// 伪代码描述 shared 模块的加载策略：

class SharedModuleResolver {
  constructor(sharedConfigs) {
    // 所有应用声明的共享配置
    this.sharedConfigs = sharedConfigs;
  }

  // 决定使用哪个版本的依赖
  resolveShared(dependencyName) {
    const configs = this.getConfigsFor(dependencyName);
    
    // 1. 收集所有可用版本
    const availableVersions = this.collectVersions(configs);
    
    // 2. 按版本优先级排序（优先使用 matching 数量多的）
    const sorted = this.sortByPreference(availableVersions, configs);
    
    // 3. 选择第一个满足所有约束的版本
    for (const candidate of sorted) {
      if (this.meetsAllConstraints(candidate, configs)) {
        return candidate;
      }
    }
    
    // 4. 如果没有完全满足的版本：
    // 如果 singleton: true，则使用冲突版本（打印警告）
    // 如果 singleton: false，则加载多个副本
    return this.handleVersionMismatch(sorted, configs);
  }

  // 版本范围检查
  meetsAllConstraints(version, configs) {
    return configs.every((config) => {
      if (!config.requiredVersion) return true;
      
      // semver 版本范围匹配
      // 例如 '^18.0.0' 匹配 >=18.0.0 <19.0.0
      return semver.satisfies(version, config.requiredVersion);
    });
  }

  // 处理版本不匹配
  handleVersionMismatch(sorted, configs) {
    const singletonConfig = configs.find(c => c.singleton);
    
    if (singletonConfig) {
      // 单例模式：使用最高版本，但打印警告
      const highestVersion = sorted[0];
      console.warn(
        `[Module Federation] 依赖 ${configs[0].name} 版本冲突:` +
        `请求 ${configs.map(c => c.requiredVersion).join(', ')}，` +
        `实际使用 ${highestVersion}`
      );
      return highestVersion;
    }
    
    // 非单例模式：每个应用使用自己的版本
    return null; // 表示 fallback 到各自的版本
  }
}

// 实际配置中的版本策略示例
sharedConfigs = {
  // 策略 1: 单例 + 严格版本（推荐）
  'react': {
    singleton: true,
    requiredVersion: '^18.0.0',
    version: '18.2.0',
  },
  
  // 策略 2: 单例 + 宽松版本
  'react-dom': {
    singleton: true,
    requiredVersion: false,  // 任何版本都行
  },
  
  // 策略 3: 非单例（每个应用独立）
  'moment': {
    singleton: false,
    // 每个应用会加载自己的 moment
  },
  
  // 策略 4: 急加载（随初始 chunk 一起加载）
  'core-js': {
    eager: true,  // 立即加载，不懒加载
  },
};
```

### 4. 动态远程加载

Module Federation 不仅支持在 webpack 配置中静态声明 remotes，还支持运行时动态添加远程容器：

```javascript
// 动态远程加载——运行时决定引用哪个远程应用
// 这在微前端场景中非常关键（版本切换、灰度、A/B 测试）

// ===== 动态远程工具函数 =====
// 使用 Webpack 提供的容器 API

async function loadDynamicRemote(name, url) {
  // 1. 检查该远程是否已加载
  if (window[`__federation_loaded_${name}__`]) {
    return window[`__federation_loaded_${name}__`];
  }

  // 2. 动态加载远程容器的入口文件
  await new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = url;
    script.onload = resolve;
    script.onerror = () => reject(new Error(`远程入口加载失败: ${url}`));
    document.head.appendChild(script);
  });

  // 3. 获取远程容器对象（由 remoteEntry.js 暴露）
  const container = window[name];
  
  // 4. 初始化容器（传入共享作用域）
  await container.init(__webpack_share_scopes__.default);
  
  // 5. 标记已加载
  window[`__federation_loaded_${name}__`] = container;
  
  return container;
}

// 运行时动态加载远程模块
async function importDynamicRemote(container, modulePath) {
  // container.get('./ProductCard') 返回一个 module factory
  const factory = await container.get(modulePath);
  // 调用 factory 获得模块的导出
  const Module = factory();
  return Module;
}

// 完整使用示例
async function dynamicallyUseRemoteModule() {
  // 场景：根据用户角色加载不同的商品推荐模块
  const userRole = getUserRole(); // 'vip' | 'normal'
  
  // 不同角色使用不同的远程服务
  const remoteConfig = userRole === 'vip' 
    ? { name: 'vip_recommend_app', url: 'http://cdn.vip.com/remoteEntry.js' }
    : { name: 'normal_recommend_app', url: 'http://cdn.normal.com/remoteEntry.js' };
  
  try {
    // 1. 动态加载远程容器
    const container = await loadDynamicRemote(
      remoteConfig.name, 
      remoteConfig.url
    );
    
    // 2. 从容器中获取模块
    const RecommendList = await importDynamicRemote(
      container, 
      './RecommendList'
    );
    
    // 3. 使用模块
    return RecommendList;
  } catch (error) {
    console.error('远程推荐模块加载失败:', error);
    // fallback 到本地默认推荐
    return null;
  }
}

// 配合 React Suspense 的封装
function createRemoteComponent(remoteName, modulePath) {
  const RemoteComponent = React.lazy(async () => {
    const container = await loadDynamicRemote(remoteName, remoteEntryUrl);
    const Module = await importDynamicRemote(container, modulePath);
    return { default: Module.default || Module };
  });
  
  return (props) => (
    <Suspense fallback={<div>加载远程组件...</div>}>
      <RemoteComponent {...props} />
    </Suspense>
  );
}
```

### 5. Module Federation 作为微前端方案

Module Federation 本身就是一个"无框架、无约定"的微前端引擎。它无需 single-spa 那种生命周期协议，也无需 qiankun 的沙箱，因为：

```javascript
// Module Federation 的微前端模式

// ===== 基座应用：暴露容器，加载远程子应用 =====
// host/webpack.config.js
new ModuleFederationPlugin({
  name: 'host_app',
  remotes: {
    'product_app': 'product_app@http://localhost:3001/remoteEntry.js',
    'checkout_app': 'checkout_app@http://localhost:3002/remoteEntry.js',
    'auth_app': 'auth_app@http://localhost:3003/remoteEntry.js',
  },
  shared: {
    react: { singleton: true, requiredVersion: '^18.0.0' },
    'react-dom': { singleton: true },
    'react-router-dom': { singleton: true },
  },
});

// ===== 基座应用：路由分发 =====
// host/src/App.jsx
import React, { Suspense } from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

// 远程子应用作为异步组件
const ProductApp = React.lazy(() => import('product_app/ProductApp'));
const CheckoutApp = React.lazy(() => import('checkout_app/CheckoutApp'));

export default function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/products">商品</Link>
        <Link to="/checkout">购物车</Link>
      </nav>
      
      <Suspense fallback={<div>加载中...</div>}>
        <Routes>
          <Route path="/products/*" element={<ProductApp />} />
          <Route path="/checkout/*" element={<CheckoutApp />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// 优势对比：
// ✅ 无需生命周期契约（直接 import 即可）
// ✅ 共享依赖自动处理版本冲突
// ✅ 内置异步加载和代码分割
// ✅ 可以暴露任意颗粒度的模块（从单个函数到整个应用）
// ❌ 没有沙箱隔离（需要团队约定）
// ❌ 子应用间的样式可能冲突
// ❌ 只有 Webpack 用户能用（不支持 Vite/Rspack）
```

## 实战案例：多应用联邦模块集成

```javascript
// ============ 场景：电商平台的 3 个团队 ============
// 
// Team A: 商品应用 (port 3001) - 暴露 ProductCard, ProductList
// Team B: 购物车应用 (port 3002) - 消费 ProductCard, 暴露 CartModule  
// Team C: 统一入口 (port 3000) - 加载所有子应用

// ===== Team A: 商品应用的联邦配置 =====
// apps/products/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;
const deps = require('./package.json').dependencies;

module.exports = {
  output: { publicPath: 'auto' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'products',
      filename: 'remoteEntry.js',
      exposes: {
        './ProductCard': './src/components/ProductCard',
        './ProductList': './src/components/ProductList', 
        './ProductRouter': './src/ProductRouter',
      },
      shared: {
        react: { singleton: true, requiredVersion: deps.react },
        'react-dom': { singleton: true, requiredVersion: deps['react-dom'] },
        'react-router-dom': { singleton: true },
      },
    }),
  ],
};

// ===== Team B: 购物车应用的联邦配置 =====
// apps/cart/webpack.config.js
module.exports = {
  output: { publicPath: 'auto' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'cart',
      filename: 'remoteEntry.js',
      // 消费商品应用的模块
      remotes: {
        products: 'products@http://localhost:3001/remoteEntry.js',
      },
      // 暴露购物车模块
      exposes: {
        './CartModule': './src/CartModule',
        './AddToCartButton': './src/components/AddToCartButton',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};

// ===== Team B 购物车中引用商品卡片 =====
// apps/cart/src/components/CartItem.jsx
import React from 'react';

// 远程加载商品卡片组件
const ProductCard = React.lazy(() => import('products/ProductCard'));

export default function CartItem({ product, quantity }) {
  return (
    <div className="cart-item">
      <Suspense fallback={<div>加载商品信息...</div>}>
        <ProductCard 
          product={product}
          showPrice={true}
          compact={true}
        />
      </Suspense>
      <div className="cart-item-quantity">
        <span>数量: {quantity}</span>
      </div>
    </div>
  );
}

// ===== Team C: 主入口应用（消费所有子应用）=====
// apps/host/webpack.config.js
module.exports = {
  output: { publicPath: 'auto' },
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        products: 'products@http://localhost:3001/remoteEntry.js',
        cart: 'cart@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
        'react-router-dom': { singleton: true },
      },
    }),
  ],
};

// ===== 主入口：路由配置 =====
// apps/host/src/App.jsx
import React, { Suspense, useState } from 'react';

const ProductRouter = React.lazy(() => import('products/ProductRouter'));
const CartModule = React.lazy(() => import('cart/CartModule'));

function App() {
  const [currentPage, setCurrentPage] = useState('products');
  
  return (
    <div className="app">
      <header className="app-header">
        <nav>
          <button onClick={() => setCurrentPage('products')}>商品</button>
          <button onClick={() => setCurrentPage('cart')}>购物车</button>
        </nav>
      </header>
      
      <main className="app-content">
        <Suspense fallback={<div className="loading">加载子应用...</div>}>
          {currentPage === 'products' && <ProductRouter />}
          {currentPage === 'cart' && <CartModule />}
        </Suspense>
      </main>
    </div>
  );
}

// ===== 验证：构建后的产物结构 =====
// 构建后产物（商品应用）:
// dist/
//   remoteEntry.js    ← 联邦入口，Webpack 自动生成
//   src_components_ProductCard_jsx.js  ← 暴露的模块
//   src_components_ProductList_jsx.js
//   vendors-node_modules_react_index_js.js  ← 共享依赖
//   ...

// 构建后产物（主入口）:
// dist/
//   main.js            ← 主入口 bundle
//   ...
// 主应用不会打包 ProductCard/ProductList 等远程模块

// ===== 应用加载流程 =====
// 1. 用户访问主应用 (localhost:3000)
// 2. 主应用加载 main.js
// 3. 用户点击"购物车"按钮
// 4. 需要加载 cart/CartModule
// 5. Webpack 先检查 shared 依赖（react已加载，复用）
// 6. 加载 cart 的 remoteEntry.js
// 7. 从 cart 的容器中获取 CartModule
// 8. CartModule 内部引用 products/ProductCard
// 9. 再加载 products 的 remoteEntry.js（如果尚未加载）
// 10. 从 products 容器获取 ProductCard
// 11. 渲染完成
```

## 底层原理

### Module Federation 源码级分析

Module Federation 的实现可以分解为几个关键的 Webpack Plugin：

1. **ContainerPlugin**：创建容器的 plugin。它生成一个"模块容器"——一个包含 `init()` 和 `get()` 方法的对象。

2. **ContainerReferencePlugin**：处理 `remotes` 配置的 plugin。当代码中 `import('products_app/ProductCard')` 时，该 plugin 将导入路径解析为从远程容器获取。

3. **SharePlugin**：处理 `shared` 配置的 plugin。生成共享作用域（share scope）的初始化代码。

4. **FallbackModule**：当共享依赖版本不匹配时的 fallback 机制。

```javascript
// Webpack Module Federation 的运行时核心代码（简化版）
// 这是 remoteEntry.js 中实际运行的逻辑

// ===== 远程容器对象 =====
// 每个联邦应用在运行时暴露这样一个容器
window.products_app = {
  // 初始化方法（传入共享作用域）
  init(shareScope) {
    // 存储共享作用域的引用
    this.shareScope = shareScope;
    
    // 检查共享依赖是否满足
    // 如果主应用已经提供了 React 18，这里不再重复加载
    if (shareScope.react) {
      this.overridableReact = shareScope.react;
    }
    
    return Promise.resolve();
  },
  
  // 获取方法（获取暴露的模块）
  get(modulePath) {
    // modulePath 例如 './ProductCard'
    
    switch (modulePath) {
      case './ProductCard':
        // 返回一个 factory 函数，调用后获得模块
        return () => {
          // 在这里解析模块，使用共享的 React 而不是本地加载的
          // __webpack_require__ 是 webpack 内部的 require 实现
          return __webpack_require__('./src/components/ProductCard.jsx');
        };
        
      case './ProductList':
        return () => {
          return __webpack_require__('./src/components/ProductList.jsx');
        };
        
      default:
        return () => {
          throw new Error(`模块 ${modulePath} 未被暴露`);
        };
    }
  },
};

// ===== 消费端的模块加载 =====
// 当在代码中写 import('products_app/ProductCard') 时，
// Webpack 将其编译为：

// 1. 先确保远程容器已加载
const remoteContainerPromise = loadScript(
  'http://localhost:3001/remoteEntry.js'
);

// 2. 初始化容器并获取模块
const moduleFactoryPromise = remoteContainerPromise
  .then(() => window.products_app)
  .then(container => container.init(__webpack_share_scopes__.default))
  .then(() => window.products_app.get('./ProductCard'));

// 3. 最终返回模块导出
export default moduleFactoryPromise.then(factory => factory());
```

### Module Federation 的"零运行时"本质

相比 qiankun 的沙箱和有 single-spa 的路由层，Module Federation 是"零运行时"的——它只在构建阶段由 Webpack 插件处理，运行时只是标准的 ESM 异步加载。这意味着：

1. **不需要安装额外的运行时框架**——只需要 Webpack 5
2. **没有性能开销**——除了远程模块的 HTTP 请求延迟外，没有额外的函数调用开销
3. **自然集成**——与 React.lazy、Suspense、动态 import 等原生 API 无缝配合
4. **TypeScript 友好**——可以通过 `@module-federation/typescript` 插件生成远程模块的类型

## 高频面试题解析

### 面试题 1：Module Federation 中的 shared 配置的 singleton 选项影响什么？

**答案要点：** `singleton: true` 表示全局只加载一份依赖实例。当多个应用都依赖 React 但版本不同时，Webpack 会尝试使用满足所有版本要求的最高版本。但如果版本完全不兼容（如 React 17 vs React 18），Webpack 仍然只加载一份（遵循 singleton 合约），但会在控制台打印版本冲突警告，可能导致运行时错误。`singleton: false` 允许每个应用加载自己的版本。通常推荐将关键 UI 框架（React、Vue）设为 singleton，工具库（lodash、moment）不设 singleton。

### 面试题 2：Module Federation 是如何保证 remote 模块的依赖不重复的？

**答案要点：** 核心是"共享作用域"（Share Scope）机制。当远程容器初始化时，`init(shareScope)` 接收主容器传递的共享依赖映射。远程模块在 require 某个依赖时，先检查 share scope 中是否存在，存在则复用，不存在才从自己的 chunk 中加载。例如：React 在主应用已经加载了 18.2.0，远程容器初始化时，主应用将 `{ 'react': { '18.2.0': {...} } }` 传给远程容器。远程容器中的模块加载时，会优先使用共享的 React 而非自身携带的。

### 面试题 3：Module Federation 与 qiankun（或 single-spa）的核心区别是什么？可以替代吗？

**答案要点：** 核心区别：Module Federation 是"模块级别的共享"，qiankun 是"应用级别的隔离"。Module Federation 不提供沙箱，不提供样式隔离，不提供生命周期管理，但它提供了更细粒度的模块共享和更自然的异步加载体验。两者可以共存：用 qiankun 做应用级（路由级）的拆分，用 Module Federation 做组件级（模块级）的共享。两者不是竞争关系，而是互补关系。

### 面试题 4：Module Federation 中 remoteEntry.js 的作用是什么？它的加载流程是怎样的？

**答案要点：** remoteEntry.js 是每个联邦应用的"目录文件"，类似于容器的索引。它包含：1）容器的 init 和 get 方法；2）暴露模块的映射表；3）共享依赖的版本信息。加载流程：消费应用发现需要远程模块 → 动态创建 script 标签加载 remoteEntry.js（利用浏览器缓存，不会重复加载）→ 调用 `container.init(shareScope)` 初始化 → 调用 `container.get(modulePath)` 获取模块的 factory → 执行 factory 获得模块导出。

### 面试题 5：Module Federation 的局限是什么？什么情况下不推荐使用？

**答案要点：** 局限：1）仅支持 Webpack 5（不支持 Vite / esbuild / Turbopack）；2）没有沙箱隔离（子应用可以污染全局 window）；3）没有样式隔离（CSS 冲突需要自行处理）；4）远程模块的调试不如本地模块方便（Source Map 需要额外配置）；5）动态远程加载时，类型推导需要额外工具支持。不推荐使用的场景：对安全隔离有严格要求（如嵌入第三方不可信代码）、团队全部使用 Vite 或 esbuild 构建。

## 总结与扩展

### 知识体系图

```
Module Federation
├── 核心概念
│   ├── 容器（Container）- 模块的出口和入口
│   ├── 暴露（Expose）- 将本地模块暴露给外部
│   ├── 引用（Remote）- 远程引用其他应用的模块
│   └── 共享（Shared）- 依赖的版本协商
├── 配置详解
│   ├── name（全局唯一应用名）
│   ├── filename（remoteEntry 文件名）
│   ├── exposes（暴露的模块映射）
│   ├── remotes（引用的远程应用）
│   └── shared（共享依赖策略）
│       ├── singleton（是否单例）
│       ├── requiredVersion（版本约束）
│       ├── eager（是否急加载）
│       └── version（指定版本）
├── 运行时机制
│   ├── Container.init(shareScope)
│   ├── Container.get(modulePath) → factory
│   ├── Share Scope 传递
│   └── 版本裁决算法
├── 微前端集成
│   ├── 路由级集成（结合 React Router）
│   ├── 组件级集成（结合 React.lazy）
│   ├── 动态远程（运行时切换远程源）
│   └── 性能优化（prefetch、缓存）
└── 局限与边界
    ├── 无沙箱隔离
    ├── 仅 Webpack 5
    ├── 类型推导需额外工具
    └── 远程调试复杂度
```

### 延伸阅读

1. **Webpack 5 Module Federation 官方文档**: webpack.js.org/concepts/module-federation
2. **Module Federation 示例仓库**: GitHub 搜索 `module-federation/module-federation-examples`
3. **Zack Jackson 的 Module Federation 论文**: 核心作者的深度分析
4. **@module-federation/typescript**: 远程模块的类型安全方案
5. **Rspack Module Federation**: 字节跳动 Rust 打包器的 Module Federation 实现
