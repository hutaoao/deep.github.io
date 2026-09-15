---
layout: post
title: "Vue 模板编译原理：AST 与 render 函数"
date: 2026-12-02 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [模板编译, AST, render函数, patchFlag, 静态提升, BlockTree]
description: >
  面试向讲透 Vue 模板编译：为什么模板不是 HTML 而是"编译器的输入语言"、
  parse → transform → generate 三阶段各自干了什么（附 3.5 实测编译产物）、
  编译期四大优化（静态缓存 / patchFlag / Block Tree / 事件缓存）如何让运行时 diff 变快，
  以及"Vue 2 的 optimize 阶段去哪了""静态提升到底提升到哪"这类高频追问的标准答法。
---

## 一句话概括

{% raw %}`<div>{{ msg }}</div>`{% endraw %} 这段东西，浏览器根本看不懂。Vue 里它叫**模板（template）**，是**编译器的输入语言**；编译器的活儿是把它翻译成一个普通的 JS 函数——`render` 函数，这个函数执行后返回虚拟 DOM（vnode）树，运行时的渲染器再拿着 vnode 去操作真 DOM。

面试问"模板编译原理"，真正想听的不是你能不能背出三个阶段的名字，而是两件事：

1. **编译发生在什么时候**（构建时还是运行时）；
2. **编译期到底做了哪些优化，让运行时 diff 更快**——这才是 Vue 和 React 都叫"虚拟 DOM"，性能表现却不一样的根本原因（官方叫**带编译时信息的虚拟 DOM**）。

记住这个总纲：**Vue 同时握着编译器和运行时两只手，所以它可以在编译期把"运行时该做什么"提前算出来并写成标记，运行时只按标记走捷径、不做判断。**

## 核心知识点

### 1. 编译发生在什么时候：构建时 vs 运行时

同一个模板，两种编译时机：

| | 构建时编译（主流） | 运行时编译 |
|---|---|---|
| 谁编译 | `@vitejs/plugin-vue` / `vue-loader` 调 `@vue/compiler-sfc` | 浏览器里现场编译 |
| 用的包 | 构建期用 `@vue/compiler-sfc`；产物里的运行时是 **runtime-only** 版 `vue`（不含编译器） | 必须引入**含编译器**的完整版（如 CDN 的 `vue.global.js`） |
| 产物 | 纯 JS（render 函数），**不含编译器** | 产物里打包进整个编译器 |
| 体积 | 小 | 编译器本身几十 KB，明显变大 |
| 场景 | 所有正式项目 | 写 demo、CDN 直引 |

为什么 .vue 文件必须构建时编译？因为它压根不是能直接跑的东西——`<template>`、`<script setup>`、`<style scoped>` 三个块都要编译。Vite 里 `@vitejs/plugin-vue` 拆开 SFC 后，把 template 块交给 `@vue/compiler-dom`（走 SSR 时交给 `@vue/compiler-ssr`）。

编译器内部是分层的三层包，这个结构面试常被顺手问一句：

```
@vue/compiler-sfc   → 处理 .vue 文件（拆块、编译 script、样式）
@vue/compiler-dom   → DOM 平台相关（HTML 标签、v-model 的 vModelText 等）
@vue/compiler-core  → 平台无关的核心流水线（90% 的"原理"都在这）
```

**记住边界**：`compiler-core` 不知道 DOM 是什么；反过来 runtime 也不 import compiler。所以 `compiler-sfc` 编译完的产物是"纯 JS 字符串"，运行时看见的只有函数。

### 2. 三阶段流水线：parse → transform → generate

`compiler-core` 的入口就是 `baseCompile`，短到可以直接背：

```js
// 简化自 compiler-core/src/compile.ts
export function baseCompile(template, options = {}) {
  const ast = baseParse(template, options)   // ① parse：模板字符串 → 模板 AST
  transform(ast, options)                    // ② transform：转换 + 打优化标记
  return generate(ast, options)              // ③ generate：AST → render 函数代码
}
```

⚠️ **和 Vue 2 最大的差别**：Vue 2 是 `parse → optimize → generate`，有个**独立的 optimize 阶段**专门去标静态节点。Vue 3 把这个阶段**合并进 transform** 了——遍历 AST 的时候顺手就把静态标记、patchFlag、Block 结构全算完，不再多走一遍树。面试答"Vue 3 编译分几步"时说"逻辑上是三步、但 Vue 2 的 optimize 已经并入 transform"，是个明确的加分点。

三段话概括每步在干嘛：

- **parse**：字符串 → 结构化的对象树（模板 AST），节点带 `type` / `tag` / `props` / `children` / `loc`；
- **transform**：遍历这棵树，把 Vue 的语义（`v-if` `v-for` `v-model` 插值）翻译成"创建 vnode 的调用"，同时**打上运行时优化标记**；
- **generate**：把转换后的 AST 序列化成 render 函数的代码字符串（**注意是字符串，不是函数**——SFC 产物最终由打包器 `new Function` 或直接写进模块里）。

### 3. parse：模板字符串怎么变成 AST

parse 本质是个**带状态机的递归下降解析器**，用一个栈维护标签层级，遇到 `<div>` 压栈、遇到 `</div>` 出栈；文本、插值、属性、指令分别生成不同类型的节点。

节点类型是个枚举（3.5 实测值，面试问到能报出来很唬人）：

| NodeTypes | 值 | 含义 |
|---|---|---|
| ROOT | 0 | 根节点 |
| ELEMENT | 1 | 元素节点 |
| TEXT | 2 | 纯文本 |
| COMMENT | 3 | 注释 |
| SIMPLE_EXPRESSION | 4 | 一个 JS 表达式（`msg`、`a + b`） |
| INTERPOLATION | 5 | 插值 `{ { msg } }` |
| ATTRIBUTE | 6 | 普通属性 |
| DIRECTIVE | 7 | 指令（`v-if`、`:id`、`@click`） |
| IF / IF_BRANCH / FOR | 9 / 10 / 11 | 结构指令转换后生成的新节点 |
| VNODE_CALL | 13 | 生成 vnode 的调用 |
| JS_CALL_EXPRESSION …… | 14~26 | generate 阶段的 JS AST（对象、数组、函数、条件、缓存…） |

从 13 往后那些 `JS_*` 就是关键信号：**transform 把"模板 AST"变成了"JS AST"**。模板 AST 描述的是模板长什么样，JS AST 描述的是 render 函数长什么样——这是理解 transform 的关键。

### 4. transform：面试重点，所有优化都在这一步

transform 的遍历模型是**洋葱模型**：`nodeTransforms` 里的插件在"进入节点"时执行，如果它返回一个函数，这个函数会在"离开节点"（子节点全部处理完之后）执行。

```js
// 简化示意：洋葱模型
traverseNode(node, context)
//  ├─ 进入：依次执行各 nodeTransform（v-if / v-for / 插值 / 元素 等）
//  │   └─ 返回的 exit 回调先存起来
//  ├─ 递归处理 children
//  └─ 离开：倒序执行 exit 回调（自底向上，父节点能看到子节点处理完的结果）
```

为什么要"自底向上"？因为**父节点的优化判断依赖子节点的结论**——比如"我整棵子树是不是全静态"，得等子节点都标完才知道。

优化点按重要性排序，前三个必须能说出来：

#### 4.1 静态节点缓存（静态提升）

模板里从不参与动态渲染的部分，没必要每次重渲染都重新创建 vnode。

官方文档现在叫**缓存静态内容（Cache Static）**，之前叫静态提升（hoistStatic）。⚠️ **3.5 有个重要变化**：静态节点的处理从"提升到模块作用域"改成了**按组件实例缓存**（官方 changelog：`compiler-core: change node hoisting to caching per instance`），原因是提升到模块级的 vnode 会被多个组件实例共用，踩过若干坑。

实测 3.5.42 的编译产物，两种形态：

```js
// 场景 A：零散静态节点（不足阈值）→ 各自缓存进渲染函数的 _cache
return function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", null, [
    _cache[0] || (_cache[0] = _createElementVNode("div", null, "foo", -1 /* CACHED */)),
    _cache[1] || (_cache[1] = _createElementVNode("div", null, "bar", -1 /* CACHED */)),
    _createElementVNode("div", null, _toDisplayString(_ctx.dynamic), 1 /* TEXT */)
  ]))
}

// 场景 B：连续 5 个以上静态元素 → 直接"预字符串化"成一个静态 vnode（HTML 串中间省略）
_cache[0] || (_cache[0] = _createStaticVNode("<div class=\"foo\">foo</div><div class=\"foo\">foo</div>……", 5))
//                                                                 第二个参数 = 这串 HTML 含几个真实节点
```

注意 `-1 /* CACHED */` 这个 patchFlag，以及 `_createStaticVNode(html, n)`：**它内部是用 `innerHTML` 一次性插入 n 个真实节点**，挂载和激活（hydration）时都跳过逐个创建。这是"预字符串化"，比一个个 vnode 更快——但只在连续静态元素数量够多时才划算（实测同层级连续 5 个元素就会触发）。

另外还有**属性键名数组提升**，这个仍然是模块作用域的常量：

```js
const _hoisted_1 = ["id", "title"]   // ← 模块级，数组只创建一次
_createElementVNode("p", { id: _ctx.id, title: _ctx.t }, "x", 8 /* PROPS */, _hoisted_1)
```

`_hoisted_1` 告诉运行时"这个节点动态属性的键是 id 和 title"——更新时只遍历这两个 key，别的不用碰。

#### 4.2 更新类型标记（patchFlag）

这是 Vue 编译优化里最核心的一个。React 在更新时不知道新树长什么样，只能**逐个 vnode 比较 props**；Vue 在编译期就知道"这个节点哪个部分会变"，于是直接**把类型编码成一个数字**进 vnode 创建调用：

```js
_createElementVNode("div", { class: _normalizeClass({ active: _ctx.active }) }, null, 2 /* CLASS */)
//                                                                                ↑ 就是这个数字
```

运行时用**位运算**检查，快到可以忽略：

```js
if (vnode.patchFlag & PatchFlags.CLASS /* 2 */) {
  // 只更新 class，绝不去碰 style / props
}
```

完整 patchFlag 表（3.5 实测导出值，值得记几个高频的）：

| 值 | 名称 | 含义 |
|---|---|---|
| 1 | TEXT | 只有文本子节点会变 |
| 2 | CLASS | 只有 class 会变 |
| 4 | STYLE | 只有 style 会变 |
| 8 | PROPS | 有动态 props（不含 class/style） |
| 16 | FULL_PROPS | **动态 key 的 props**（如 `v-bind="obj"`），没法预知比什么，必须全量比 |
| 32 | NEED_HYDRATION | 有需要激活的事件（3.5 里原名 HYDRATE_EVENTS） |
| 64 | STABLE_FRAGMENT | 子节点顺序永远不会变 |
| 128 | KEYED_FRAGMENT | 带 key 的 v-for 片段 |
| 256 | UNKEYED_FRAGMENT | 不带 key 的 v-for 片段 |
| 512 | NEED_PATCH | 需要 patch 但不是 props（ref、自定义指令） |
| 1024 | DYNAMIC_SLOTS | 动态插槽 |
| -1 | CACHED | 完全静态，**永远跳过比对** |
| -2 | BAIL | 放弃优化，退回全量 diff |

**两个负数要单独记**：`-1` 表示"完全静态、直接跳过"，`-2` 表示"这段太复杂，放弃优化退回全量 diff"。正数越大不代表越好，值本身只是一个**类型标记的比特位**。

**这些标记可以叠加**——多个动态类型会用**位或**合并成一个数字。实测对比：

```js
// <p :id="id">x</p>              → 8 = PROPS
_createElementVNode("p", { id: _ctx.id }, "x", 8 /* PROPS */, _hoisted_1)

// <p :id="id">{{ msg }}</p>      → 9 = 1(TEXT) | 8(PROPS)
_createElementVNode("p", { id: _ctx.id }, _toDisplayString(_ctx.msg), 9 /* TEXT, PROPS */, _hoisted_1)
```

两种情况下键名数组都提升成了模块级的 `const _hoisted_1 = ["id"]`——**编译期算好，运行时零成本**。而运行时那个 `&` 判断也不只判一次：`patchFlag & (PatchFlags.TEXT | PatchFlags.PROPS)` 就能分别处理。

#### 4.3 Block Tree（树结构打平）

光有 patchFlag 还不够：如果没有 block，更新时还是得**递归遍历整棵 vnode 树**才能找到那些带标记的节点。

所以 Vue 3 引入了 **Block**："内部结构稳定的一段"就是一个块。块在创建时，会把**所有带 patchFlag 的后代节点（不只是直接子节点）收集到一个扁平数组 `dynamicChildren` 里**。

{% raw %}
```text
模板结构                     编译后的 block 结构
<div>                        div (block root)
  <div>...</div>              ├─ div :id        ← 被追踪
  <div :id="id"></div>        └─ div {{ bar }}  ← 被追踪（跨层级收集）
  <div>
    <div>{{ bar }}</div>      （静态节点完全不在列表里）
  </div>
</div>
```
{% endraw %}

于是更新时**只遍历这个扁平数组，不再遍历整棵树**——官方叫"树结构打平"，静态部分被高效略过。实测产物里的标志就是 `openBlock()` + `createElementBlock()`：

```js
return (_openBlock(), _createElementBlock("div", null, [
  _createElementVNode("div", { id: _ctx.id }, null, 8 /* PROPS */, _hoisted_1),
  _createElementVNode("div", null, [ _createElementVNode("div", null, _toDisplayString(_ctx.bar), 1) ]),
  ...
]))
```

**`v-if` 和 `v-for` 会创建新的块**，因为它们会改变结构：

```js
// v-if：条件分支是一个独立 block，不成立时用注释节点占位
(_ctx.ok)
  ? (_openBlock(), _createElementBlock("p", { key: 0 }, [ /* ... */ ]))
  : _createCommentVNode("v-if", true)

// v-for：循环本身是一个 Fragment block，按有没有 key 标 128 / 256
(_openBlock(true), _createElementBlock(_Fragment, null, _renderList(_ctx.list, (i) => {
  return (_openBlock(), _createElementBlock("li", { key: i.id }, _toDisplayString(i.name), 1))
}), 128 /* KEYED_FRAGMENT */))
```

注意 v-for 那句 `_openBlock(true)` 的 `true` 是 `disableTracking`——**v-for 内部不再往上收集动态节点**，因为列表项的顺序/数量本身就可能变，交给 Fragment 统一处理更合理。这个细节面试提到会很出彩。

#### 4.4 缓存事件处理函数（cacheHandlers）

内联事件处理函数每次渲染都是新函数，会让子组件白白重渲染。编译期直接缓存它：

```js
// 开 cacheHandlers 前后
_createElementBlock("button", { onClick: () => _ctx.onClick() })          // 每次渲染新函数
_createElementBlock("button", {
  onClick: _cache[0] || (_cache[0] = (...args) => (_ctx.onClick && _ctx.onClick(...args)))
})                                                                       // 只创建一次
```

这解释了为什么 Vue 里 `@click="handleClick"` 基本不用手动 `useCallback` 之类的优化——**编译器已经免费帮你做了**。

### 5. generate：AST → render 函数代码

generate 遍历 transform 留下的 `codegenNode`（VNODE_CALL 及其 JS AST），拼出代码字符串，同时按需收集运行时要用到的 helper 函数（`createElementVNode`、`toDisplayString`、`openBlock`…）作为 import。产物长这样：

```js
const { createElementVNode: _createElementVNode, toDisplayString: _toDisplayString,
        openBlock: _openBlock, createElementBlock: _createElementBlock } = Vue

return function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("p", null, _toDisplayString(_ctx.msg), 1 /* TEXT */))
}
```

这里有个容易答错的点：**`prefixIdentifiers`**。

- `prefixIdentifiers: true`（**SFC / esm-bundler 构建的默认值**）：模板里的变量编译成 `_ctx.msg`，render 函数是**纯函数**，可以从组件实例上摘出来单独用；
- `prefixIdentifiers: false`：编译成 `with (_ctx) { ... msg ... }`。实测产物：

```js
return function render(_ctx, _cache) {
  with (_ctx) {
    const { toDisplayString: _toDisplayString, openBlock: _openBlock, createElementBlock: _createElementBlock } = _Vue
    return (_openBlock(), _createElementBlock("p", null, _toDisplayString(msg), 1 /* TEXT */))
  }
}
```

`with` 是运行时编译（CDN 直引）才用的兜底，因为**严格模式下 `with` 直接报错**，而且 `with` 会让变量查找变慢、无法被引擎优化。所以"运行时编译体积大 + 性能差，构建时编译是唯一正经选择"这两条理由是连在一起的。

顺手再记一个：`v-model` 编译成什么？

```js
// <input v-model="text" /> 的产物
_withDirectives((_openBlock(), _createElementBlock("input", {
  "onUpdate:modelValue": $event => ((_ctx.text) = $event)
}, null, 8 /* PROPS */, _hoisted_1)), [
  [_vModelText, _ctx.text]      // ← 运行时指令，负责真正同步 DOM 的值
])
```

所以 `v-model` 在编译产物里**是"一个 prop + 一个运行时指令"**，不是一个魔法语法。理解这个就理解了为什么 `.lazy` `.number` `.trim` 是改指令的 modifier 而不是改 prop。

### 6. 面试标准答题模板（可直接复述）

> Vue 3 的模板编译在 `@vue/compiler-core` 里，入口是 `baseCompile`，分三步：
> **parse** 把模板字符串用栈式解析器变成模板 AST；**transform** 遍历这棵树，把 `v-if`、`v-for`、`v-model`、插值这些模板语义翻译成创建 vnode 的调用，同时打上编译期优化标记；**generate** 把 AST 序列化成 render 函数的代码字符串。
> 跟 Vue 2 比，Vue 2 有独立的 optimize 阶段标静态节点，Vue 3 把它并进了 transform。
> 真正重要的是 transform 阶段的优化，主要有四个：**静态内容缓存**（静态节点缓存进 `_cache`，连续多个还会预字符串化成 `_StaticVNode` 用 innerHTML 插入）、**patchFlag**（把"这块只有 class 会变"编码成数字，运行时用位运算判断，跳过无关比对）、**Block Tree**（把带标记的后代节点收集到扁平数组 `dynamicChildren`，更新时只遍历这个数组而不遍历整棵树，`v-if` / `v-for` 会生成新 block）、以及**事件处理函数缓存**。
> 这套东西官方叫"带编译时信息的虚拟 DOM"——因为编译器知道模板长什么样，可以把判断提前到编译期，这是 Vue 和纯运行时虚拟 DOM（如 React）在性能策略上最大的区别。

## 其实你每天都在用

- 写 {% raw %}`<div class="a">{{ msg }}</div>`{% endraw %} 时那个 `1 /* TEXT */` 标记，你已经吃过它带来的性能红利了，只是没看见。
- 用 `<script setup>` 时模板里的 `msg` 会被编译成 `_ctx.msg`，所以 render 函数认得的是"组件实例上的属性"——这就是为什么从 `defineProps()` 解构出来的变量（3.5 之前）会丢响应性：解构那一刻取到的是快照值，不在 `_ctx` 的访问链上了。
- 连续写一堆纯静态的 `<li>` 菜单项，Vue 偷偷把它们合成一个 `_StaticVNode` 用 innerHTML 插进去，你写的十几行标签实际只有一次 DOM 操作。
- `@click="doSomething"` 之所以不用像 React 那样担心"每次渲染新函数导致子组件重渲染"，是因为编译器把它缓存进了 `_cache[0]`。
- 在 v-for 里忘了写 `:key`，编译产物就从 `128 /* KEYED_FRAGMENT */` 变成 `256 /* UNKEYED_FRAGMENT */`，两者 diff 策略完全不同。
- 打开 [template-explorer.vuejs.org](https://template-explorer.vuejs.org) 就能直接看到你写的模板编译成了什么，这是排查"为什么这段没被优化"的最快方式。
- Vite 里改一行 template，热更新的恰恰是那个 render 函数——你其实每天都在跑这个编译器。

## 常见误解（FAQ）

**❌ 误区1："模板编译就是三步，`parse → optimize → generate`"**

这是 **Vue 2** 的答案。Vue 2 确实有独立的 `optimize` 阶段（标 `static` / `staticRoot`）。Vue 3 把它并入了 transform，官方流水线是 `parse → transform → generate`。答错这一条很伤，因为它直接暴露你的知识是抄来的。

**❌ 误区2："静态提升就是把静态节点提升到模块作用域，只创建一次"**

这是 **3.4 及之前**的描述。3.5 官方 changelog 明确写着 `change node hoisting to caching per instance`——静态 vnode 现在是**缓存进渲染函数的 `_cache`**（`_cache[0] || (_cache[0] = ...)`），按实例缓存而不是模块级共享。仍然提升到模块作用域的主要是**静态属性键名数组**（`const _hoisted_1 = ["id"]`）这类轻量常量。面试时补一句"3.5 改成了按实例缓存，因为模块级 vnode 多实例共享会出问题"，档次立刻不同。

**❌ 误区3："编译产物是个函数，Vue 运行时直接调用"**

`generate` 产出的是**代码字符串**。构建时编译的情况下，这段字符串被写进模块文件（或经 `new Function` 生成），由打包器落到产物里；运行时编译（CDN 引完整版）才是真的在浏览器里 `new Function`。所以"构建时编译能减小体积"里的"减小"，减掉的是**编译器本身的代码**，不是 render 函数。

**❌ 误区4："patchFlag 就是给节点打个标记，运行时还是会比较"**

patchFlag 的价值就在于**不比较**。`vnode.patchFlag & PatchFlags.CLASS` 为真时，运行时只更新 class，**完全跳过其它 props 的 diff**；`-1 /* CACHED */` 更是直接整块跳过。配合 Block Tree 后，连"找到哪些节点要更新"这一步都不用递归遍历了。这是"带编译时信息的虚拟 DOM"和"纯运行时虚拟 DOM"的本质差异。

**❌ 误区5："Block 就是把静态节点包起来，避免被 diff"**

反了。Block 的核心是**收集动态节点**：它把整棵子树里所有带 patchFlag 的后代节点（**跨层级**）收集进 `dynamicChildren` 扁平数组，更新时遍历这个数组就够了。静态节点确实被略过，但方式是"压根不在收集列表里"，而不是"被包起来"。

**❌ 误区6："`with (_ctx)` 和 `_ctx.msg` 只是写法不同"**

代价差得很远。`with` 在**严格模式（ES Module 默认严格）下直接是语法错误**，而且会阻断引擎的变量查找优化；`_ctx.msg` 是纯属性访问，渲染函数还能被单独复用。所以 SFC/esm-bundler 默认 `prefixIdentifiers: true`，`with` 只是运行时编译的兼容兜底。

## 一句话总结

**Vue 模板编译就是「parse 建树 → transform 翻译并打标记 → generate 出代码」，真正值钱的不是这三步，而是 transform 顺便完成的静态缓存、patchFlag、Block Tree——它们把运行时的判断和遍历提前到了编译期。**
