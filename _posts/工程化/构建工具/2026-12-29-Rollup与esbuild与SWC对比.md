---
layout: post
title: "Rollup 与 esbuild / SWC 对比"
date: 2026-12-29 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: [Rollup, esbuild, SWC, Rolldown, TreeShaking, 构建工具选型]
description: >
  这三个工具常被放在一张表里比较，但它们根本不在同一层：Rollup 是打包器，esbuild 是打包器 + 转译器，SWC 主要是转译器。搞混层次是面试第一个扣分点。
---

## 一句话概括

面试官问"Rollup 和 esbuild / SWC 有什么区别"，最容易翻车的答法是列一张速度表然后说"esbuild 最快，所以选 esbuild"。**这三个工具不是同一层的替代品**：

- **Rollup**：JS 写的**模块打包器**，ESM 优先，"输出最干净"是它的立身之本
- **esbuild**：Go 写的**打包器 + 转译器 + 压缩器**，全流程一站，"快"是它的全部卖点
- **SWC**：Rust 写的**转译器**（Babel 的替代品），主战场是别人的 loader/compiler 插件位

一句话记：**Rollup 管"模块图"，esbuild 管"速度"，SWC 管"语法降级"。**

所以真正该背的不是"谁快"，而是三件事：**为什么它们快、Tree-shaking 差在哪、各自明确不做什么**。这三条答清楚，选型题自然就通了。

## 核心知识点

### 1. 先分层：谁干谁的活

层次搞错，后面全是错。看这张表：

| 工具 | 语言 | 本体是什么 | 主要能力 | 典型出现位置 |
| --- | --- | --- | --- | --- |
| **Rollup** | JavaScript | 模块打包器 | 打包 + Tree-shaking + 多种输出格式 | 发 npm 库；Vite 8 之前的生产构建 |
| **esbuild** | Go | 打包器 + 转译器 | 打包 / 转译 / 压缩 三合一 | 独立脚本工具；工具链底层；Vite 8 之前 dev 与依赖预构建 |
| **SWC** | Rust | **转译器** | 语法降级 + 压缩（转译阶段不做 Tree-shaking） | `swc-loader`（webpack/Rspack）、`@swc/jest`、Next.js 编译器、Parcel |

几个容易混的点：

- **SWC 不是"打包器"**。它虽然也能打包，但生态重心在**转译**——它被设计成嵌到别人的构建流程里当"语法转换那一环"，而不是自己管整条流水线。
- **esbuild 既能打包也能只转译**。`esbuild.build()` 是打包，`esbuild.transform()` 是单文件转译，两个 API 别混着说。
- **Rollup 不做语法降级**（把 ES6+ 转成 ES5 是 Babel/SWC 的活），它假定输入已经是 ESM。这也是它插件列表里常年挂 `@rollup/plugin-babel` 的原因。

### 2. 为什么快：三个可以复述的原因

别只答"因为 Go / Rust 比 JS 快"。esbuild 官方 FAQ 给了很实在的解释，摘三个最有信息量的：

**① 语言与内存模型的关系，不只是"编译成原生码"。** 官方原话大意是：Go 在同一进程内**线程间共享内存**，而 JavaScript 必须在线程之间**序列化数据**；Go 的堆是所有线程共享的，JS 每个线程一个堆。作者按自己的测试估算，这"几乎把 JS worker 线程的可行并行度砍掉了一半"——因为一半核心在给另一半收垃圾。另一个角度：跑打包器的 JS VM 是第一次见到这份代码、没有任何优化提示，**你在解析代码的同时 node 还在解析打包器自己**。

**② 三个阶段里能并行的全并行。** esbuild 内部大致分 **parsing → linking → code generation** 三步，其中解析和代码生成占了绝大部分工作且完全可并行，linking 天然是串行的。多入口共享同一批依赖时，因为共享内存，工作可以直接复用。

**③ 最硬的一条：整个 AST 只走三遍。** 官方明确列了这三遍：

```text
第 1 遍：词法分析、语法分析、建立作用域、声明符号
第 2 遍：绑定符号、压缩语法、JSX/TS → JS、ESNext → ES2015
第 3 遍：压缩标识符、压缩空白、生成代码与 source map
```

其他打包器会把这几步拆成独立的 pass，并且在库与库之间**来回转换数据表示**（`string → TS → JS → string`，再 `string → JS → 老语法 JS → string`，再 `→ 压缩后 JS → string`）。往返越多，内存搬得越多，CPU 缓存命中率越低。esbuild 的做法是**趁 AST 还在缓存里热着就一路做完**。

再加上"全部自研"这条：官方点名说 TypeScript 官方编译器**即使在关闭类型检查时似乎仍在跑类型检查器**，而 esbuild 自己写的 TS parser 没有这个包袱。

官方 benchmark 的数字（大致规模的前端代码库）：`esbuild 0.39s` / `parcel 2 14.91s` / `rollup + terser 34.10s` / `webpack 5 41.21s`。

SWC 那边的官方口径是 **"比 Babel 单线程快 20 倍，四核快 70 倍"**，道理同源：Rust + 多线程 + 不做类型检查。

### 3. Tree-shaking：最容易被追问的能力差异

这一节是选型的真正分水岭，也是**可以实测出差异的地方**。同一份源码，用 Rollup 4.64.3 和 esbuild 0.28.2 各打一遍：

```js
// 输入：main.js
import * as ns from './ns.js';     // ns.js 里导出了 A / B / onlyUsedViaNamespace

console.log(ns['A']);              // 字符串字面量键，两边都能静态解析出来

export function withDeadLocal(x) {
  const neverUsed = 'dead string in function';   // 函数体内没人用
  const alsoNever = 1 + 2;                       // 同上
  return x * 2;
}
console.log(withDeadLocal(3));
```

（顺带一条：上面那个命名空间访问**两个打包器都处理掉了**——`B` 和 `onlyUsedViaNamespace` 都被删，`ns['A']` 被直接解析成 `A`。所以真正的差距不在这一行。）

两个打包器的产物差别很明确：

```text
===== Rollup =====
const A = 1;
console.log(A);

function withDeadLocal(x) {        // ← 函数体内的死局部变量也删了
  return x * 2;
}
console.log(withDeadLocal(3));
export { withDeadLocal };

===== esbuild =====
var A = 1;
console.log(A);
function withDeadLocal(x) {
  const neverUsed = "dead string in function";   // ← 原样留着
  const alsoNever = 1 + 2;                       // ← 原样留着
  return x * 2;
}
console.log(withDeadLocal(3));
export { withDeadLocal };
```

原因写在官方文档里：**esbuild 对自己的定位是 "declaration-level dead code removal"（声明级死代码消除）**，而 Rollup 做的是跨模块的图分析 + scope hoisting，能一路删到函数体内部。

esbuild 还有一条更值得背的规则：**它的副作用判定是"保守"的**。官方举的例子很典型：

| 表达式 | 能否被删 | 原因（官方给的理由） |
| --- | --- | --- |
| `12.34`、`"abcd"`、`{ a: a }` | ✅ 无副作用 | 字面量，删了不影响任何行为 |
| `"ab" + cd` | ❌ 不能 | 字符串拼接会调用 `toString()`，可能有副作用 |
| `foo.bar` | ❌ 不能 | 成员访问可能触发 getter |
| 引用一个全局标识符 | ❌ 不能 | 该全局不存在时会抛 `ReferenceError` |

想让 esbuild 敢删，得自己上注解——`/* @__PURE__ */`，**它只能放在 `new` 表达式或调用表达式前面**：

```js
// 告诉打包器：这个调用没有副作用，没人用就删掉
const gammaTable = /* @__PURE__ */ (() => {
  const table = new Uint8Array(256);
  for (let i = 0; i < 256; i++) table[i] = Math.pow(i / 255, 2.2) * 255;
  return table;
})();
```

顺带一个可以当加分点的细节：**esbuild 在生成 JSX 时会自动给 `jsx()` / `jsxDEV()` 调用加这个注解**，所以未使用的 JSX 元素能被摇掉。另外，`/* @__PURE__ */` 不只 esbuild 认，Terser / UglifyJS 也认（Webpack、Parcel 用的就是它们），所以这个注解是**跨工具可移植**的。

还有两条边界：

- esbuild 的 Tree-shaking **依赖 ESM 的 `import` / `export`，对 CommonJS 无效**
- 它**默认只在 `--bundle` 或 `format: iife` 时开启**（可以显式设 `treeShaking: true`）

压缩环节也有差距。实测把上面的 esbuild 产物再过一遍 Terser 5.51.2：

```text
esbuild minify:true      → console.log(1);function t(o){let s="dead string in function";return o*2}...
esbuild 产物 + Terser    → function o(o){return 2*o}console.log(1),console.log(o(3));...
```

**Terser 把 esbuild 遗留的那个字符串赋值也删掉了。** 所以"esbuild 压缩比 Terser 快几十倍"和"esbuild 压缩率不如 Terser"是同一件事的两面。

### 4. 能力边界：esbuild 明确不做什么

这部分建议直接背，因为面试官问"那能不能全都用 esbuild"的时候，答"能"就完了。

**① 不做 TypeScript 类型检查。** 官方原话是：esbuild 解析 TS 语法并**丢弃类型注解**，但"不做任何类型检查，你仍然需要并行跑 `tsc -noEmit`"。所以 TS 项目里 esbuild 和 `tsc` 是**分工**关系，不是替代关系。

**② 不做 ES5 降级。** 官方文档小标题就是 "ES5 is not supported well"，正文写"把 ES6+ 语法转成 ES5 目前还不支持"。实测（`target: 'es5'` 传入含 `const` / `class` 的源码）**直接报错**：

```text
Transform failed with 5 errors:
<stdin>:2:0: ERROR: Transforming const to the configured target environment ("es5") is not supported yet
<stdin>:4:0: ERROR: Transforming class syntax to the configured target environment ("es5") is not supported yet
<stdin>:4:27: ERROR: Transforming object literal extensions to the configured target environment ("es5") is not supported yet
...
```

但官方给了一个**反直觉的配套建议**，这条很能体现深度：

> 如果你的输入本来就是 ES5 代码，**`target` 仍然要设成 `es5`**。

因为不设的话，esbuild 会为了"更短"而往你的 ES5 代码里塞 ES6 语法。实测：

```text
输入：      var o = { x: x }; var s = "a\nb";

target es5     →  var o={x:x},s="a\nb";      ← 保持了 ES5 写法 ✅
target esnext  →  var o={x},s=`a<换行>b`;     ← 引入了 ES6 简写和模板字符串 ❌
```

而**纯 ES5 语法的代码在 `target: es5` 下是正常通过的**（不报错）——报错只发生在"需要降级"的时候。

**③ 官方写明不进 core 的东西**（也就是说别指望以后会加）：

```text
其他前端语言（Vue / Svelte / Elm / Angular）
TypeScript 类型检查（"just run tsc separately"）
自定义 AST 操作的 API
Hot Module Replacement
Module Federation
```

**④ 项目自己的成熟度定位。** 官方 FAQ 里两句原话值得知道：作者说把 esbuild 视为 **"a late-stage beta"**，因为**还没到 1.0**、且"缺少一些重要功能（主要是 **code splitting 还很原始**）"；另外 **"这个工具主要由我一个人在构建"**。roadmap 一节还写着：**当前不做积极的特性开发**（因为作者手上的项目没有大型前端代码库了），只做维护和定期发布。

顺便一句容易被忽略的：**esbuild 没有内置 HTML 入口**（不生成 HTML）、**没有内置 HMR**——这两条正是当年 Vite 被造出来的直接原因。

### 5. SWC 的定位，以及 2026 年必须知道的工具链变化

**SWC 就是"更快的 Babel"。** 官方首页列了它在谁手里：Next.js、Parcel、Deno，企业侧 Vercel、字节、腾讯、Shopify、Trip.com 等。它的用法是 `.swcrc`：

```json
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": { "syntax": "typescript", "tsx": true },
    "transform": { "react": { "runtime": "automatic" } },
    "target": "es2015"
  },
  "module": { "type": "es6" }
}
```

**这里有个很容易答错、但极其实用的细节：`jsc.target` 的官方默认值是 `"es5"`。** 这恰恰解释了为什么 SWC 能直接顶替 Babel——**默认就做降级**（对比 esbuild 根本不做）。也解释了为什么它能被塞进 webpack 里当 `swc-loader`：因为下游要的"把新语法变成老语法"这件事它默认就干了。

SWC 不做的两件事和 esbuild 一样：**不做 TS 类型检查**（那是 tsc / vue-tsc 的活，`@swc-node` 的提速只是替掉了转译）；**转译阶段不做 Tree-shaking**（交给上层打包器，这也正是"SWC 不是打包器"的证明）。

然后是**这一节最重要的、也是很多八股答案已经过期的地方**：

> **Vite 8（stable 2026-03-12）把 esbuild 和 Rollup 一起换掉了。**

老答案说"Vite 用 esbuild 做开发、Rollup 做生产"，这在 Vite 7 及以前是对的。Vite 8 变成了**单一 Rust 打包器 Rolldown**：

| 位置 | Vite 7 及以前 | **Vite 8** |
| --- | --- | --- |
| 生产构建 | Rollup | **Rolldown**（Rust，兼容 Rollup 插件 API） |
| 依赖预构建 | esbuild | **Rolldown** |
| JS/TS 转换 | esbuild | **Oxc** |
| 默认 CSS 压缩 | esbuild | **Lightning CSS** |

**这不是道听途说，看一眼 `package.json` 就能验证**（本机 `vite@8.3.4` 的实际依赖）：

```text
vite 8.3.4 的 dependencies：
  rolldown ~1.2.12 / lightningcss ^1.33.0 / postcss ^8.5.29 / picomatch / tinyglobby
  → 没有 rollup 依赖，也没有 esbuild 依赖
```

官方给的理由也很清楚：**两个打包器 = 两条转换管线 + 两套插件系统**，中间的胶水代码要一直对齐，边缘 case 会不断累积。统一之后插件生态反而兼容（Rolldown 实现的是 Rollup 插件 API），官方宣称 **10–30x faster than Rollup**。顺带还有一个变化：Vite 8 的默认 `build.target` 升到了 Chrome 111 / Edge 111 / Firefox 114 / Safari 16.4。

其他框架的现状也顺手记一句：**Next.js 16 起 Turbopack 成为默认打包器**（要退回 webpack 就 `next build --webpack`）；Rspack 走的是"webpack 兼容 + Rust 提速"的迁移路线。

**面试怎么答才不出错**：先讲经典架构（Vite 7 的 esbuild + Rollup 分工，理由是"dev 要快、build 要输出质量"），然后补一句"不过 Vite 8 已经统一成 Rolldown 了"——**答旧架构不失分，主动补现状是加分**。

### 6. 怎么选：把"哪个好"换成"你在包什么"

选型题的正确打开方式是先分类，再看约束：

| 你要做的事 | 选择 | 核心理由 |
| --- | --- | --- |
| 发一个 npm 库 | **Rollup** | 一次构建多格式产出 + `external` 排除 peer deps + 输出最干净 |
| 现代 Web 应用 | **Vite**（8+ = Rolldown + Oxc） | 开发体验 + 生态完整 |
| 只需要"转译"这一环 | **SWC** | 嵌进 webpack/Rspack/Jest 当 loader，默认降到 ES5 |
| 一次性脚本 / 工具链底层 / 只求快 | **esbuild** | 一个二进制、零配置、极快 |
| 大型遗留 webpack 项目 | **Rspack** | 配置兼容，迁移成本最低 |
| Next.js 应用 | Turbopack + SWC | 框架默认 |

库打包有三条硬知识，都是实测出来的，很能撑场面：

**① 想要 UMD / IIFE，就不能有动态 import。** Rollup 在开启代码分割的构建里用 `format: 'umd'` 会直接抛错：

```text
Error: Invalid value "umd" for option "output.format"
       - UMD and IIFE output formats are not supported for code-splitting builds.
```

因为 UMD/IIFE 是"一个自足的文件"，而代码分割的本质是"多个文件互相 `import`"——两者在模型上就矛盾。

**② 同一次构建里，不同格式的产物形态差别很大。** Rollup 实测 `import()` 会被按格式改写：

```text
// 源码
export async function loadA() { const m = await import('./a.js'); return m.a; }

// ESM 产物：保留字面动态 import，切成独立 chunk
async function loadA() { const m = await import('./a-DjCriV_3.js'); return m.a; }

// CJS 产物：改写成 Promise + require
async function loadA() {
  const m = await Promise.resolve().then(function () { return require('./a-DpYTsmnR.js'); });
  return m.a;
}
```

**③ 别指望"打自己的库能摇掉东西"。** 库的入口就是公开 API，**所有导出都是被使用的**，所以 Tree-shaking 在自己这一侧收益为零——实测把库按 IIFE 打出来，`used`、`unusedHeavy`、`UNUSED_CONST` 全都原样保留在 `exports.xxx =` 里。

**Tree-shaking 是给"用你库的人"准备的能力**，所以库开发者要做的是另一件事：`package.json` 里写对 `sideEffects` 和 `exports`，并且保留 ESM 入口，让下游的打包器有东西可摇。

## 其实你每天都在用

- 你在 `vite.config.js` 里写 `build.rollupOptions`，这个配置名从 Vite 2 一直留到现在，就是因为底层曾经是 Rollup（Vite 8 里它是兼容层）。
- 你写了 `const x = someFactory();` 结果没用到，打包后还在——因为 `someFactory()` 被当成有副作用，得手动加 `/* @__PURE__ */`。
- 你给组件库加 `"sideEffects": false`，用户那边的产物就变小了——这就是"Tree-shaking 是给使用者"的具体体现。
- 你的 TS 项目里 `vite build` 通过了但类型报了一堆错——因为构建那个环节**只擦类型、不查类型**。
- 你在 Jest 里配 `@swc/jest`（或在 webpack 里配 `swc-loader`），测试/构建从几十秒掉到几秒——这就是 SWC 的位置：替掉 Babel 那一环。
- 你发一个组件库，产物目录里同时有 `.mjs` / `.cjs` / `.umd.js`——老实的做法就是 Rollup 一次构建产出多格式。
- 你升级到 Next.js 16 之后构建快了一大截，`node_modules` 里少了一堆 webpack 相关包——Turbopack 变成默认了。
- 你看到 `node_modules` 里的 `@swc/core` 带一个平台专属的 `.node` 二进制（不是纯 JS），那就是"编译成原生码"的实物证据——换平台要重装，所以它必须按平台分发。

## 常见误解（FAQ）

**❌ 误区一："esbuild 最快，所以就用 esbuild 打包一切"**

它是"打包 + 转译 + 压缩"里最快的，但**能力上明确有洞**：不做 TS 类型检查、不做 ES5 降级（`target: 'es5'` 直接报错）、没有 HMR、没有 HTML 入口、code splitting 官方自评"pretty primitive"。所以它的真实位置是"**工具链底层的那台高性能发动机**"——被 Vite（8 以前）当 dev 与依赖预构建用，而不是自己去管整条流水线。

**❌ 误区二："SWC 是打包器，和 Rollup / esbuild 三选一"**

不是。SWC 的本体是**转译器**，它的主战场是"插到别人的构建流程里当语法转换那一环"（`swc-loader`、`@swc/jest`、Next.js 编译器、Parcel 的 Parse/Transform 阶段）。它当然有打包能力，但生态重心不在那。**它和 Babel 是同一层的竞品，而不是 Rollup 的竞品。**

**❌ 误区三："Tree-shaking 是打包器都能做好的事"**

不是同一个水平。官方文档里 esbuild 自己把这件事定义为 **declaration-level dead code removal（声明级）**；实测同一份源码，Rollup 能把**函数体内**未使用的局部变量也删掉，esbuild 保留。原因是两者的副作用判定强度不同：`foo.bar`、`"ab" + cd`、引用未声明的全局标识符，esbuild 都算"有副作用"。**这是"输出干净程度"和"构建速度"的取舍，不是 bug。**

**❌ 误区四："ES5 项目里 esbuild 的 target 随便设"**

`target` 一定要设 `es5`——**不是为了降级（它降不了），而是为了防止 esbuild 往你的 ES5 代码里塞 ES6 语法**。实测：不设时 `{ x: x }` 会被缩成 `{ x }`、`"a\nb"` 会变成模板字符串（因为更短），设了才保持原样。这条官方专门写了，但很少人知道。

**❌ 误区五："Vite 用 esbuild 做开发、Rollup 做生产"**

这是 **Vite 7 及以前**的架构，而且是很好的答案——但现在要补一句：**Vite 8（2026-03-12 stable）已统一成单一 Rust 打包器 Rolldown**，JS 转换换成 Oxc，CSS 压缩换成 Lightning CSS。官方给的理由是"两个打包器 = 两套插件系统 + 无穷的对齐胶水"。面试里**先答经典架构、再补现状变化**是最稳的答法。

**❌ 误区六："库打包也得靠 Tree-shaking 把产物变小"**

库的入口就是公开 API，**所有导出都是"用到的"**，所以自己这一侧摇不掉任何东西（实测 IIFE 产物把所有导出原样留着）。库能做的只有三件事：保留 ESM 入口、写准 `sideEffects`、把 peer deps 放进 `external`。**真正摇的人是下游的使用方。**

**❌ 误区七："既然 esbuild 压缩快几十倍，就不用 Terser 了"**

速度是真的，压缩率有差距也是真的。实测同一份 esbuild 产物，`minify: true` 会留下一些可删的语句，再过一遍 Terser 5.51.2 才全删干净。所以历史上有不少项目是"**esbuild 转译 + Terser 压缩**"的组合——用速度换回一点体积。

## 一句话总结

**Rollup、esbuild、SWC 的差别不在"谁快"，而在三件事：层次不同（打包器 / 打包器+转译器 / 转译器）、Tree-shaking 强度不同（Rollup 能删到函数体内，esbuild 只到声明级且副作用判定保守）、能力边界不同（esbuild 明确不做类型检查、不做 ES5 降级、不做 HMR）——而 2026 年答题时，别忘了 Vite 8 已经用 Rolldown + Oxc 把这两条管线合并成了一条。**
