---
layout: post
title: "Tree-shaking 原理与副作用处理"
date: 2026-12-31 00:00:00 +0800
categories: ["工程化", "构建工具"]
tags: [TreeShaking, sideEffects, 副作用, Rollup, esbuild, 构建工具]
description: >
  Tree-shaking 的难点从来不是"删未使用的导出"，而是"判断一个表达式能不能删"。副作用判定是整套优化的地基，判错了轻则产物变大，重则 polyfill、埋点、全局样式静默消失。
---

## 一句话概括

Tree-shaking 这个动作本身很简单：**从入口出发，标记所有能走到的代码，走不到的就删掉。** 真正难的是另一件事——**走不到的代码，真的可以删吗？**

举个例子。下面两行都是"没被用到"的代码，但只有一个能安全删除：

```js
const a = 'x' + 'y';        // 删掉没影响
const b = 'x' + getName();  // 删掉就不一样了
```

第二行虽然 `b` 没人用，但 `getName()` 这个函数调用**可能会改全局状态、发请求、写 localStorage**。打包器看不到 `getName` 的函数体（可能在别的文件），它只能假设"这个调用有副作用"，于是保留。

**所以副作用判定的本质是"证明无害"**：打包器必须能确凿地证明"这个表达式删掉后，运行时行为完全不变"，才敢删。证明不了就保守保留。这条规则决定了你产物体积的上限，也决定了 `sideEffects: false` 为什么是一个能删掉线上 polyfill 的危险开关。

面试里这个知识点的三层问法：①"Tree-shaking 的前提是什么"（送分，答 ESM + 无副作用）；②"`sideEffects` 字段有什么用"（中等）；③"为什么我加了 `sideEffects: false`，样式/polyfill 反而没了"（高分题，考的就是本文内容）。

## 核心知识点

### 1. 两个阶段：标记可达 → 判断副作用

先理清 Tree-shaking 到底做了几件事，很多人把它和压缩器的 DCE 混在一起：

| 阶段 | 谁做 | 干什么 | 粒度 |
|---|---|---|---|
| **标记（marking）** | 打包器（Rollup/esbuild/webpack） | 从入口遍历模块图，标记可达的导出 | 模块 / 导出级 |
| **副作用判定（treeshake）** | 打包器 | 对未标记的代码逐条判断"能否安全删除" | **语句 / 表达式级** |
| **物理删除（DCE）** | 压缩器（Terser 等） | 把标记为死的代码真的从产物里剪掉 | 语句 / 表达式级 |

**注意第二行和第三行不是一回事。** 打包器负责"判断这条语句算不算死代码"，压缩器负责"真的剪掉"。这也是为什么 `optimization.usedExports: true` 打开后，webpack 产物里会出现 `/* unused harmony export */` 注释——那只是**标记**，还没剪；得等 Terser 上场才真的消失。

这个区分有实际意义：如果你发现产物里还有未使用的导出，先看压缩器有没有开，再看标记有没有生效。

### 2. 副作用判定：打包器到底在问什么

对每个"未被引用"的语句，打包器要回答一个问题：**删掉它，运行时会不会有可观测的差异？**

判断顺序大致是这样：

1. **能静态求值吗？** 纯字面量运算（`'a' + 'b'`、`1 + 2`）→ 删了没影响，可删。
2. **是函数/类声明吗？** 声明本身没副作用，可删。
3. **里面的标识符是已知的吗？** 遇到未知标识符（可能是全局变量）→ 可能是 getter、可能抛 ReferenceError，保留。
4. **有函数调用 / `new` 吗？** 除非有 `/* @__PURE__ */` 标注，否则保留（调用可能有副作用）。
5. **是属性访问吗？** `foo.bar` 可能触发 getter，保留。

这就是为什么"纯字面量赋值"和"带函数调用的赋值"命运完全不同。区别不在语句形式，而在**能否证明无害**。

### 3. 实测：Rollup 和 esbuild 的判定差异

光讲规则太虚，我把同样一批"未被引用的模块顶层语句"分别喂给 Rollup 4.64.3 和 esbuild，看它们各自的产物（`import './side.js'` 只为副作用，语句本身无人引用）：

| 模块顶层语句 | Rollup | esbuild |
|---|---|---|
| `'abc'` | 删空 | 删空 |
| `1 + 2 * 3` | 删空 | **保留** |
| `'a' + 'b'` | 删空 | 删空 |
| `` `plain` `` | 删空 | 删空 |
| `` `${1}` `` | 删空 | 删空 |
| `` `${Math.random()}` `` | 删空 | **保留** |
| `foo()` | 保留 `foo();` | 保留 `foo();` |
| `new Date()` | 删空 | 删空 |
| `location` | 删空 | 删空 |
| `typeof window` | 删空 | 删空 |
| `/x/` | 删空 | 删空 |
| `[1, 2]` | 删空 | 删空 |
| `({ a: 1 })` | 删空 | 删空 |
| `'x'.toUpperCase()` | 删空 | **保留** |
| `() => 1` | 删空 | 删空 |
| `class A { m() { return 1 } }` | 删空 | 删空 |

这张表有几个反直觉的地方，都是面试可以拿出来讲的：

- **Rollup 几乎全删了，esbuild 留了四条。** 这就是两者产物体积差距的一个直接来源。
- **`1 + 2 * 3` 被 esbuild 保留，`'a' + 'b'` 却被删了。** 看起来矛盾——同样是字面量运算，为什么一个留一个删？因为 esbuild 对**数字**字面量的处理更谨慎（涉及数字格式化等可观察行为），而字符串拼接可以安全求值。
- **`` `${Math.random()}` `` 被 esbuild 保留。** 模板字符串里有函数调用，esbuild 不敢删；Rollup 敢——它能确认 `Math.random` 是内置方法。
- **`'x'.toUpperCase()` 被 esbuild 保留。** 方法调用一律视为有副作用，哪怕 `'x'` 是字面量——因为 `.toUpperCase` 理论上可被原型污染改写。
- **两条 "保留" 的形态不一样，值得注意。** `foo()` 在两边都是**原样保留**（因为没有能力分析它）；而 esbuild 保留 `1 + 2 * 3`、`` `${Math.random()}` ``、`'x'.toUpperCase()` 时是**保留了整个表达式语句**。

再补一个"删声明、留调用"的实测，这是理解 Tree-shaking 动作方式的关键：

```js
// 函数体内
export function f() {
  const unused = heavyCompute();   // 声明没被用到，但调用不能删
  return 42;
}

// Rollup 产物：声明没了，调用被提出来保留
function f() {
  heavyCompute();
  return 42;
}
```

**"声明可以删、调用必须留"** 这个剥离动作，就是 Tree-shaking 最核心的操作方式。它说明打包器不是在"整块删代码"，而是在**逐层剥离可证明无害的部分**，把不能证明无害的表达式剥到最内层保留下来。

对照一下就很清楚：`const unused = 1;` 这种**纯字面量赋值**是整条删掉的（声明和值都没副作用）；而 `const unused = heavyCompute();` 会退化成裸调用。**判定的最小单位不是"语句"，而是"表达式"。**

### 4. esbuild 保守判定的真正边界：有没有标识符

很多文章说"esbuild 保守，`'ab' + cd` 这种拼接不会删"。这话只对了一半，容易误导。

真正的分界线是 **表达式里有没有标识符**，不是"有没有运算符"：

| 模块顶层语句（模块内有其他被使用的导出） | esbuild 产物 | 原因 |
|---|---|---|
| `'a' + 'b';` | **删除** | 纯字面量，可静态求值 |
| `"ab" + cd;` | 保留 | `cd` 是未知标识符 |
| `const x = "ab" + cd;` | 保留 | 同上 |
| `foo.bar;` | 保留 | 可能是 getter |
| `someGlobal;` | 保留 | 可能是未知全局，删了可能吞掉 ReferenceError |

`cd` 在打包器眼里是一个**未知全局变量**，它无法排除"读 `cd` 会触发 getter 或抛错"的可能。而 `'a' + 'b'` 全是字面量，`+` 对两个字符串是纯运算，没有任何可观察行为。

**所以面试答题的正确说法是**："esbuild 只做 declaration-level 的死代码消除，对含**未知标识符**的表达式一律保守保留"——而不是笼统地说"esbuild 不做表达式分析"。区别在于，你把边界说准了，面试官就知道你真的看过产物。

### 5. `sideEffects` 字段：一个能删掉 polyfill 的开关

前面讲的是**模块内部**的语句判定。`sideEffects` 解决的是**模块级**判定：**整个模块如果导出全没被用，能不能整块删掉？**

默认情况下打包器不敢删，因为模块顶层可能有副作用（注册全局、改原型、注入样式）。`sideEffects` 就是让你**主动声明**：

```json
{
  "name": "my-lib",
  "sideEffects": false
}
```

含义是"**这个包里所有文件都没有副作用，导出没用就可以整块删**"。如果是部分文件有副作用，用数组只列出那些：

```json
{
  "sideEffects": ["./src/polyfill.js", "*.css"]
}
```

这里有个**源码级细节**值得记住。看 `@rollup/plugin-node-resolve` 的实现（v16.0.3，`dist/es/index.js`）：

```js
const finalPackageSideEffects = packageSideEffects.map((sideEffect) => {
  // 不含 / 的模式，前面自动补 **/
  if (sideEffect.includes('/')) return sideEffect;
  return `**/${sideEffect}`;
});
internalPackageInfo.hasModuleSideEffects =
  createFilter(finalPackageSideEffects, null, { resolve: pkgRoot });
```

**不含 `/` 的模式会被自动补成 `**/` 前缀**——所以 `*.css` 实际等于 `**/*.css`，能匹配任意深度的 CSS 文件。源码注释里直接引用了 webpack 文档说明这个行为。知道这一点，你就不会写出 `["dist/*.css"]` 这种只想匹配一级、结果漏掉嵌套文件的写法。

### 6. `false` 和数组的语义差别（这个最容易搞错）

这是本文最重要的一段，因为它直接对应线上事故：

```json
// ① 布尔：全包统一语义
"sideEffects": false        // 全部文件都当作「可无副作用处理」
"sideEffects": true         // 全部文件都当作「有副作用」，等于放弃优化

// ② 数组：选择性声明 —— 列出的是「有副作用」的，其余视为无副作用
"sideEffects": ["./src/polyfill.js", "*.css"]
```

**注意数组的语义方向**：数组里列出的是**保留项**，没列出的都按无副作用处理。所以 `["*.css"]` 的意思是"只有 CSS 有副作用，其他文件都可以放心删"——**不是**"只有 CSS 可以被删"。

我实测了这几种写法，效果差异很大。测试结构是这样的一个包：

```js
// node_modules/fake-lib/src/index.js
export { keep } from './pure.js';
import './side.js';          // ← 只为副作用而 import

// node_modules/fake-lib/src/side.js
globalThis.__telemetry = 'init';   // 纯副作用，没有任何导出被使用
```

业务入口只用 `keep`。测 `side.js` 的副作用能不能活下来：

| `sideEffects` 值 | Rollup | esbuild |
|---|---|---|
| `false` | ❌ 副作用被删 | ❌ 副作用被删 |
| `true` | ✅ 保留 | — |
| `["./src/side.js"]` | ⚠️ 被删 | ✅ 保留 |
| `["src/side.js"]` | ⚠️ 被删 | — |
| `["**/side.js"]` | ⚠️ 被删 | — |
| `["*.js"]` | ✅ 保留 | — |
| `["*.css"]`（只标 css） | ❌ 被删 | ❌ 被删 |

两个结论：

**第一，Rollup 和 esbuild 在数组写法上不一致。** 我通过拦截 `resolveId` 确认过：`["./src/side.js"]` 时 Rollup 拿到的 `moduleSideEffects(side.js)` **确实是 `true`**，匹配是对的。但当这个副作用模块是通过 barrel 文件（`index.js` 里 `import './side.js'`）间接引入时，Rollup 最终还是把副作用删掉了。而**入口直接 `import` 这个文件时，同样的配置就能正常保留**。

这不是配置写错，而是"选择性声明 + barrel 中转"这个组合下的行为差异。实践建议很明确：**如果你的包里有必须保留的副作用文件（polyfill、全局注册），要么用 `"sideEffects": true` 放弃这部分优化，要么确保消费方直接 import 它。**

**第二，`false` 在这两个工具里行为完全一致——都会删。** 所以 `sideEffects: false` 是一个"信任声明"：你说了没副作用，打包器就真的照做，删错了它不负责。

### 7. 为什么这个开关会造成事故

回到最常见的三个事故场景，它们的共同点是**"副作用看起来不像副作用"**：

```js
// ① polyfill：修改内置原型，没有任何导出
import 'core-js/es/array/flat';

// ② 全局样式：只为注入 CSS，没有任何导出
import './global.css';

// ③ 全局注册：给 EventEmitter 加默认行为
EventEmitter.defaultMaxListeners = 20;
```

这三个文件**导出是空的**。在打包器眼里，"没有导出的模块 + 声明无副作用" = 整块删除。于是：

- 老浏览器上 `Array.prototype.flat` 报 `is not a function`
- 页面全局样式全丢
- 线上埋点一次都没上报

而且**构建不报错、控制台不报警**——这是最难查的一类 bug。

所以业界最佳实践是：**库作者的 `sideEffects` 只写 `false` 的前提是真的检查过所有文件**；而应用开发者（不是库作者）通常根本不需要写这个字段——它主要是给**发布到 npm 的包**用的。

顺带说一个更隐蔽的坑：`"sideEffects": false` 会让 `import './global.css'` 被删掉。所以库里有 CSS 的话必须显式列出：

```json
{ "sideEffects": ["**/*.css"] }
```

（记住上面说的：写 `*.css` 也会被自动补成 `**/*.css`，效果一样。）

### 8. 函数体内 vs 模块顶层：两套完全不同的判定

还有一个高频困惑点：为什么删不掉函数体内的死代码？

```js
export function calc() {
  const unused = heavyCompute();   // 这条能删吗？
  return 42;
}
```

**模块顶层的语句**判定相对容易，因为模块只执行一次。**函数体内的语句**就复杂得多——函数可能被调用任意次，闭包可能捕获变量，`arguments` 可能被使用。

实测结果（Rollup 4.64.3）：

```js
// 输入
export function f() {
  const unusedLocal = 1;
  const unusedCall = Math.random();
  const dead = 'x' + 'y';
  return 42;
}

// Rollup 产物：全删
function f() {
  return 42;
}
```

Rollup 做到了函数体内的死变量消除。而 **esbuild 官方明确说它只做 "declaration-level dead code removal"**——即模块导出声明这一层，不深入函数体做变量级分析。

这就解释了那个经典现象：**同样的源码，Rolldown/Rollup 产物比 esbuild 小一截**。差距往往不在 `import` 语句的删除上（那个大家都做），而在函数体内部的死变量和死分支上。

另一种情况：`const unused = heavyCompute()` 这条**删不掉**——因为 `heavyCompute()` 是调用，可能有副作用。Rollup 的产物会保留调用本身：

```js
function f() {
  heavyCompute();
  return 42;
}
```

还是那句话：**删声明，留调用。**

### 9. `/* @__PURE__ */`：手动告诉打包器"这个调用无害"

既然打包器对函数调用一律保守，那怎么办？答案是用注释标注：

```js
// 打包器不知道 createApp 有没副作用，默认保留
const app = createApp(App);

// 标注后：如果 app 没被使用，整句可删
const app = /* @__PURE__ */ createApp(App);
```

`@__PURE__` 的字面意思就是"纯的"——告诉打包器"这个调用的返回值只取决于入参，没有副作用"。它**只能放在调用表达式或 `new` 表达式前面**：

```js
/* @__PURE__ */ foo();
/* @__PURE__ */ new Bar();
```

放错位置（比如放在变量声明前）是无效的。另外一个细节：**esbuild 在生成 JSX 时会自动给 `jsx()` 调用加上 `@__PURE__`**，因为 JSX 工厂函数被约定为纯函数——所以没被使用的 JSX 元素能被正常摇掉。这也是 React/Solid 项目里 `@__PURE__` 大量出现的原因。

还有一个更现代的写法：`/* @__NO_SIDE_EFFECTS__ */`，用来标注**函数声明**（不是调用），表示"这个函数被调用时也没有副作用"。相比 `@__PURE__` 要每次调用都标注，它标注一次就管所有调用点：

```js
/* @__NO_SIDE_EFFECTS__ */
export function makeId() {
  return Date.now().toString(36);
}

makeId();   // 结果没被使用 → 可安全删除
```

### 10. 怎么排查"该摇的没摇"

最后给一套实用排查思路，面试也可以作为"你实际怎么处理"来答：

**第一，先确认标记有没有生效。** webpack 下看产物有没有 `/* unused harmony export */` 注释；有注释说明标记到了，是压缩器没剪；没注释说明标记就没生效。

**第二，确认模块被正确解析成 ESM。** 最常见的失效原因是 Babel 把 `import` 转成了 `require`：

```js
// ❌ .babelrc 里用了这个 preset，Tree-shaking 直接失效
{ "presets": [["@babel/preset-env", { "modules": "commonjs" }]] }

// ✅ 交给打包器处理模块语法
{ "presets": [["@babel/preset-env", { "modules": false }]] }
```

**第三，检查 barrel 文件（`index.js` 里 `export * from ...`）。** 这是最隐蔽的一个。barrel 本身可能没问题，但当它跟 `sideEffects` 声明、循环依赖混在一起时，行为就不好预测了——我上面实测的 `["./src/side.js"]` 失效场景就属于这一类。

**第四，显式声明不够时，用工具确认。** `rollup-plugin-visualizer` / `webpack-bundle-analyzer` 能直接看到哪个包占了体积；Rollup 还能通过 `treeshake.manualPureFunctions` 或 `moduleSideEffects` 钩子精细控制。

**第五，最实用的兜底：看 ESM 产物而不是 CJS 产物。** 现在的库基本都会提供双格式，打包器优先选 `module` / `exports` 里的 ESM 入口。如果只有 CJS 产物，Tree-shaking 基本无望。

## 其实你每天都在用

- **`import { debounce } from 'lodash-es'` 只把 debounce 打进产物**：lodash-es 是纯 ESM 且标了无副作用，而 `import _ from 'lodash'` 只能整包引入近 70KB。
- **`import { Button } from 'antd'` 不再需要 babel-plugin-import**：现代 antd 直接把 `sideEffects` 和 ESM 入口配好了，按需引入由打包器完成。
- **`import './global.css'` 在 `sideEffects: false` 的库里消失了**：这是最经典的事故，样式全丢但不报错。
- **`import './polyfill.js'` 上线后老浏览器报错**：同样是空导出模块被判定为死代码整块删除。
- **`import { createApp } from 'vue'` 不会把整个 Vue 打进来**：Vue 3 的 ESM 产物按功能拆分，未使用的 API 会被摇掉。
- **写 `const config = { api: 'x' }` 在模块顶层没被用到，产物里就没了**：模块顶层未引用声明会被删（Rollup 和 esbuild 都会）。
- **`console.log` 在函数体内没被删**：函数体分析 esbuild 不做，Rollup 虽然能做但对调用一律保守。
- **自己写的工具库被同事说"摇不掉"**：多半是产物只有 CJS，或者没在 `package.json` 里声明 `sideEffects`。
- **JSX 组件没被使用时能整块消失**：因为 esbuild/Babel 给 `jsx()` 调用自动加了 `@__PURE__`。
- **项目从 webpack 换到 Vite 后产物体积变了**：底层从 webpack 的标记 + Terser 换成了 Rolldown 的深度 treeshake，函数体内优化更激进。

## 常见误解（FAQ）

**❌ 误区一："Tree-shaking 就是删掉没被 import 的导出"**

只对了一半。删未使用的导出只是**标记**阶段。真正决定产物体积的是**副作用判定**阶段——打包器能不能证明"这条语句删了无害"。删不干净通常不是标记的问题，而是判定太保守（比如遇到未知标识符、函数调用就留）。

**❌ 误区二："esbuild 不做表达式级分析，`'ab' + cd` 这种拼接一律保留"**

表述不准确，而且会答错细节。准确说法是：esbuild 只做 **declaration-level** 的死代码消除，对含**未知标识符**的表达式保守保留。我实测 `'a' + 'b'`（纯字面量）**会被删掉**，而 `"ab" + cd`（含未知标识符 `cd`）保留。**分界线是"有没有标识符"，不是"有没有运算符"。**

**❌ 误区三："`sideEffects: false` 会阻止所有 Tree-shaking"**

反了。`false` 的含义是"**没有副作用，可以放心摇**"，它是**强化** Tree-shaking 的开关。真正会阻止优化的是 `true`。搞反这个方向，就会得出完全相反的结论。

**❌ 误区四："`sideEffects: ["*.css"]` 表示只有 CSS 可以被删除"**

方向搞反了。数组里列出的是**有副作用、需要保留**的文件，没列出的都按"无副作用"处理。`["*.css"]` = "CSS 有副作用要保留，其他随便删"。所以库里如果有 polyfill 和 CSS，必须都列进去，漏一个就可能被删掉。

**❌ 误区五："写 `*.css` 只能匹配根目录的 CSS"**

不会。`@rollup/plugin-node-resolve` 里有一段明确逻辑：**不含 `/` 的模式会被自动补成 `**/` 前缀**，所以 `*.css` 实际等价于 `**/*.css`，能匹配任意层级。源码注释里还专门引用了 webpack 文档说明这是有意为之。

**❌ 误区六："`sideEffects` 写死 `false` 一定最省体积，所以无脑加上"**

这是最容易引发线上事故的做法。`false` 是**信任声明**，打包器不会再去检查。模块顶层有全局注册、原型修改、样式注入的文件会被静默删除，而且**构建不报错**。我实测 Rollup 和 esbuild 都会删。正确做法是只列真正有副作用的文件，或者干脆不写这个字段（应用项目一般不需要）。

**❌ 误区七："Rollup 和 esbuild 的 Tree-shaking 结果应该一样"**

不一样，实测差异明显。同一个 `sideEffects: ["./src/side.js"]` 声明 + barrel 中转的场景，esbuild 保留了副作用，**Rollup 反而删掉了**；而纯字面量运算 `1 + 2 * 3`，Rollup 删、esbuild 留。**换打包器产物体积变化是正常现象**，不要默认两者行为一致。

**❌ 误区八："函数体内的死代码和模块顶层一样容易被删"**

差别很大。模块顶层只执行一次，判定容易；函数体可能被调用多次、涉及闭包和 `arguments`，分析成本高。**esbuild 官方明确只做 declaration-level，不深入函数体**；Rollup 能做（我实测函数体内未使用的局部变量全被删了），这也是两者产物体积差距的来源之一。

**❌ 误区九："`/* @__PURE__ */` 放在哪都生效"**

无效位置不生效。它**只能放在调用表达式或 `new` 表达式前面**：`/* @__PURE__ */ foo()`、`/* @__PURE__ */ new Bar()`。放到变量声明前（`/* @__PURE__ */ const x = ...`）是没用的。另外注意它标注的是**这一次调用**；要对一个函数的所有调用点生效，应该用 `/* @__NO_SIDE_EFFECTS__ */` 标注函数声明。

## 一句话总结

Tree-shaking 的完整逻辑是「**从入口标记可达代码 → 对不可达代码逐条证明无害 → 交压缩器物理删除**」，其中真正的难点是第二步；面试时把「**打包器不是在删代码块，而是在剥离无害的部分——删声明、留调用**」这句话说出来，再补一句「**`sideEffects: false` 是信任声明，写错会让 polyfill 和全局样式静默消失**」，这道题就能拿高分了。
