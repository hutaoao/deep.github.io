---
layout: post
title: "Loader与Plugin开发深度解析"
date: 2026-08-01 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: ["Webpack", "Loader", "Plugin", "自定义Loader", "自定义Plugin"]
math: true
mermaid: true
---

## 一句话概括

Webpack 的 Loader 是"内容翻译器"（文件进，文件出），Plugin 是"生命周期钩子"（在编译各阶段注入自定义逻辑）——Loader 管"每个模块变成什么"，Plugin 管"构建过程中额外做什么"，两者加起来是 Webpack 扩展体系的全部。

## 核心知识点

### 1. Loader：一个函数，一个管道

```js
// 最简 loader：把所有 console.log 替换为 console.warn
module.exports = function(source) {
  return source.replace(/console\.log/g, 'console.warn');
};

// 同步返回（两种方式等价）
module.exports = function(source) { return transform(source); };
module.exports = function(source) { this.callback(null, transform(source)); };

// 异步 loader：用 this.async()
module.exports = function(source) {
  const callback = this.async(); // 告诉 webpack 这是异步的
  fetchRemoteTransform(source).then(result => callback(null, result));
};
```

**Loader 的"管道链"**：`use: ['style-loader', 'css-loader', 'less-loader']` 执行顺序是 `less-loader → css-loader → style-loader`。前一个 loader 的输出是后一个的输入，像 Unix 的 pipe。

### 2. Pitch 机制：Loader 的"熔断"

```js
// 普通阶段：从右到左（less → css → style）
// Pitch 阶段：从左到右（style → css → less）

module.exports = function(source) { /* normal */ };
module.exports.pitch = function(remainingRequest) {
  // 如果 pitch 返回了非 undefined 值，跳过后续 loader 的 normal 阶段
  // style-loader 就是用 pitch 来"拦截" css-loader 的输出并注入 DOM
  return `
    var style = document.createElement('style');
    style.innerHTML = require(${JSON.stringify(remainingRequest)});
    document.head.appendChild(style);
  `;
};
```

**典型使用场景**：`style-loader` 的 pitch 阶段把 `css-loader` 的结果直接注入 DOM 而非传给下一个 loader。如果没有 pitch，style-loader 必须等 css-loader 返回 CSS 字符串后再操作，而 pitch 可以直接在模块代码中内联 `require`。

### 3. 获取 webpack 配置和上下文

```js
// this 上有丰富的 API——不只是 source 转换器
module.exports = function(source) {
  this.cacheable();             // 声明结果可缓存
  const options = this.getOptions(); // 获取 loader 配置
  const filePath = this.resourcePath; // 当前文件路径
  this.emitFile('output.txt', content); // 额外输出文件
  this.addDependency(filePath);  // 添加 watch 依赖

  // 输出 source map 支持
  this.callback(null, result, sourceMap);
  return result;
};

// webpack.config.js 中传配置
{ test: /\.md$/, use: [{ loader: 'md-loader', options: { headingLevel: 2 } }] }
```

### 4. Plugin：Tapable 钩子全掌握

```js
class MyPlugin {
  apply(compiler) {
    // 钩子类型：SyncHook / SyncBailHook / AsyncParallelHook / AsyncSeriesHook
    // 注册方式：tap(同步) / tapAsync(回调) / tapPromise(Promise)

    // 在编译开始前修改 asset 名
    compiler.hooks.compilation.tap('MyPlugin', compilation => {
      compilation.hooks.processAssets.tap({ name: 'MyPlugin', stage: webpack.Compilation.PROCESS_ASSETS_STAGE_OPTIMIZE }, () => {
        for (const chunk of compilation.chunks) {
          chunk.files.forEach(file => {
            compilation.updateAsset(file, old => {
              return new webpack.sources.RawSource(old.source().replace(/foo/g, 'bar'));
            });
          });
        }
      });
    });

    // 构建完成后生成分析报告
    compiler.hooks.done.tap('MyPlugin', stats => {
      const time = ((stats.endTime - stats.startTime) / 1000).toFixed(2);
      console.log(`✅ 构建完成: ${time}s, ${stats.compilation.modules.size} 个模块`);
    });
  }
}
```

**常用钩子速查**：

| 钩子 | 时机 | 用途 |
|------|------|------|
| `compiler.hooks.initialize` | 初始化完 | 设置插件内部状态 |
| `compiler.hooks.compilation` | Compilation 创建 | 注册 compilation 级别钩子 |
| `compilation.hooks.buildModule` | 每个模块开始构建 | 记录模块构建耗时 |
| `compilation.hooks.processAssets` | Asset 处理阶段 | 修改最终输出内容、生成额外文件 |
| `compiler.hooks.emit` | 输出到磁盘前 | 最后修改 assets 的机会 |
| `compiler.hooks.done` | 构建完成 | 日志、通知、性能报告 |

### 5. 实战：手写一个 Bundle Analyzer 插件

```js
class SimpleAnalyzerPlugin {
  apply(compiler) {
    compiler.hooks.done.tap('SimpleAnalyzer', stats => {
      const modules = stats.compilation.modules;
      const report = {};
      for (const mod of modules) {
        const size = mod.size(); // 模块大小（字节）
        if (size > 0) {
          const name = mod.identifier().split('node_modules/').pop() || mod.identifier();
          report[name] = (report[name] || 0) + size;
        }
      }
      // 按大小排序，输出 Top 10
      const top = Object.entries(report).sort((a, b) => b[1] - a[1]).slice(0, 10);
      console.log('\n📦 Bundle 体积分析 Top 10:');
      top.forEach(([name, size], i) =>
        console.log(`  ${i + 1}. ${name}: ${(size / 1024).toFixed(1)}KB`)
      );
    });
  }
}
```

## 其实你每天都在用

1. **`@import './variables.css'` 能被解析**——`css-loader` 的职责之一就是解析 CSS 中的 `@import` 和 `url()`，把它们也变成模块依赖，参与依赖图。你写的 CSS import 和 JS import 在 webpack 看来是同一种东西。
2. **Vue SFC（单文件组件）的 `<template>` + `<script>` + `<style>` 三合一**——`vue-loader` 把一个 `.vue` 文件拆成三块，分别交给 `template-compiler`、`babel-loader`、`css-loader` 处理，最后拼回一个组件模块。这是"一个 loader 产出多条 pipeline"的经典案例。
3. **`process.env.NODE_ENV` 在生产构建中被替换为 `"production"`**——`DefinePlugin` 在编译阶段做字符串替换，而不是运行时检查。结果是代码中的 `if (process.env.NODE_ENV === 'development')` 分支被 Tree Shaking 直接移除，生产包体不含开发工具代码。
4. **CSS Modules 的 `:local(.title)` 编译为 `.title_abc123`**——`css-loader` 的 `modules` 选项在代码生成阶段做类名哈希，同时导出 `{ title: 'title_abc123' }` 给 JS。你写的 `styles.title` 能工作，全靠這個 loader。
5. **CI 构建后自动上传 source map 到 Sentry**——`@sentry/webpack-plugin` 在 `compiler.hooks.done`（或 `afterEmit`）阶段把生成的 `.map` 文件上传，不影响构建流程本身。这是 Plugin 模式的标准应用：构建完成 → 副作用。

## 常见误解（FAQ）

- **❌ 误区：「Loader 和 Plugin 选哪个取决于个人喜好」**
  真相：职责完全不同。Loader 处理**单个文件**的内容转换（把 `.ts` 变成 `.js`），Plugin 处理**构建流程**的扩展（生成额外文件、修改 asset、注入变量）。一个判断标准：如果你的逻辑需要来自多个模块的信息（如"所有 chunk 的总大小"），那就不是 loader 能做、必须是 plugin。

- **❌ 误区：「Loader 的 `this.callback` 和直接 `return` 一样」**
  真相：`return` 只能传 transformed source。`this.callback(err, source, sourceMap, meta)` 可以传 sourceMap + 自定义 meta 给下一个 loader。如果你写的 loader 需要传递 AST 或其他中间数据给后续 loader，必须用 callback。

- **❌ 误区：「自定义 Plugin 只要实现 `apply` 就行了，不用管 Tapable 类型」**
  真相：钩子类型决定了执行顺序和行为。`SyncHook` 是同步串行，`AsyncParallelHook` 是异步并行（TerserPlugin 用这个来并行压缩 chunk），选了错误的 hook 类型可能导致"明明注册了但顺序不对"或"异步操作被当作同步跳过"。

- **❌ 误区：「插件开发门槛太高，不如直接用社区方案」**
  真相：大部分实用插件代码不到 50 行。`DefinePlugin` 的核心代码不到 100 行，`SimpleAnalyzerPlugin` 只要 15 行。关键是理解你的需求对应哪个 hook——找到对的钩子，代码量不是问题。

## 一句话总结

Loader 是"单一职责的翻译官"（给我一个文件，还你另一个文件），Plugin 是"全流程的观察者 + 参与者"（给我编译器的 hook 列表，我可以在任何阶段插入逻辑）——掌握这两个概念的边界和它们的协作方式，你就能让 Webpack 做任何你想做的事。
