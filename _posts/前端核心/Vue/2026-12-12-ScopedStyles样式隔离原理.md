---
layout: post
title: "Scoped Styles 样式隔离原理：它不是 Shadow DOM，是两个动作拼出来的"
date: 2026-12-12 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [Scoped CSS, 样式隔离, "data-v", ":deep()", CSS Modules]
description: >
  面试常问"scoped 怎么实现样式隔离"，多数人只会背"加个 data-v 属性"。
  其实它拆成两半：编译期用 PostCSS 给每个选择器后缀加 [data-v-xxx]，运行时靠组件的
  __scopeId 在挂载元素时把属性写上去。文章带真实编译产物、选择器变形全量实测表
  （含 :deep / :slotted / :global / :root / @keyframes 的坑）和属性落点规则。
---

## 一句话概括

**Vue 的 `scoped` 是"编译期改 CSS + 运行时改 DOM"两个动作拼出来的假隔离，本质是给同一组件的样式和元素贴同一个随机标记，然后靠属性选择器把它们对上。**

面试为什么爱问这个？因为它一道题能同时考三件事：你知不知道 SFC 编译流程、懂不懂 CSS 选择器权重、以及有没有在真实项目里被样式坑过。而且它跟 Shadow DOM 的"真隔离"很容易被混为一谈——这是最常见的答错点。

先给结论，两个动作分别是什么：

1. **编译期**：PostCSS 插件把所有选择器改写成带 `[data-v-xxx]` 后缀的形式（`.title` → `.title[data-v-abc123]`）。
2. **运行时**：组件挂载元素时，把 `data-v-abc123` 这个属性写到元素的 DOM 属性上。

样式表和元素各自带上同一个标记，属性选择器一匹配，样式自然就只作用在本组件的元素上了。没有 Shadow DOM，没有改 shadow root，也没有任何运行时样式计算——就是**一个属性选择器**。

## 核心知识点

### 1. 关键一：`data-v-xxx` 是哪来的，怎么算的

这个 id 不是编译器算的，是**构建插件**（`@vitejs/plugin-vue` / `vue-loader`）算的。看 plugin-vue 源码就一句话：

```js
// node_modules/@vitejs/plugin-vue/dist/index.mjs
const hash = crypto.hash ?? ((alg, data, enc) => crypto.createHash(alg).update(data).digest(enc));
function getHash(text) {
  return hash('sha256', text, 'hex').substring(0, 8);   // 取 sha256 前 8 位十六进制
}

// 生成 descriptor.id
descriptor.id = getHash(normalizedPath + (isProduction ? source : ''));
```

注意最后一行：**开发环境只用文件路径算哈希，生产环境是"路径 + 源码"**。带来的两个后果：

- 同一个组件文件，**开发环境和线上看到的 `data-v-xxx` 不一样**。你在 DevTools 里看到的 id 拿到线上对不上，这是正常的。
- 生产环境只要你改了源码，hash 就会变，所以构建缓存里的 CSS 可能和 JS 对不上——这类问题排查时要先想到。

### 2. 关键二：`data-v-xxx` 不是编译器 inline 进去的，是运行时打的

这里是绝大多数人答错的地方。很多人以为编译产物是 `_createElementVNode("div", { "data-v-abc123": "" })`。**在 Vue 3.5 里不是了。** 我们用真实的 `@vue/compiler-sfc` 编译一遍：

```js
import { parse, compileTemplate } from '@vue/compiler-sfc'
const { descriptor } = parse(`<template><div class="wrapper" :id="uid"><p>hi</p></div></template>`,
  { filename: 'Test.vue' })

compileTemplate({
  source: descriptor.template.content,
  filename: 'Test.vue',
  id: 'abc123',
  scoped: true,          // 关键：开启 scoped
}).code
```

产物里 props 干干净净，**一个 `data-v` 都没有**：

```js
import { createElementVNode as _createElementVNode, openBlock as _openBlock,
         createElementBlock as _createElementBlock } from "vue"

const _hoisted_1 = ["id"]

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", {
    class: "wrapper",
    id: _ctx.uid
  }, [...(_cache[0] || (_cache[0] = [
    _createElementVNode("p", null, "hi", -1 /* CACHED */)
  ]))], 8 /* PROPS */, _hoisted_1))
}
```

那属性从哪来？从运行时。完整的链条是这样的：

```text
构建插件：_sfc_main.__scopeId = "data-v-abc123"      // 只在有 scoped style 时才挂
   ↓
runtime-core：setCurrentRenderingInstance 里
              currentScopeId = instance.type.__scopeId
   ↓
createVNode：vnode.scopeId = currentScopeId
   ↓
mountElement：setScopeId(el, vnode, vnode.scopeId, vnode.slotScopeIds, parentComponent)
              → hostSetScopeId(el, 'data-v-abc123') → el.setAttribute(...)
```

所以记忆点是一句话：**编译产物管"长什么样"，运行时管"贴什么标记"，两边靠同一个 id 对上。** 这也解释了一个现象——如果你用 `h()` 手写 render 函数（不走 SFC 编译），照样能有样式隔离，只要你给组件加上 `__scopeId`。

`__scopeId` 是构建插件加在组件对象上的，不是 `compileScript` 加的（实测 `compileScript` 产物里搜不到 `__scopeId`），而且**只有这个组件有 `<style scoped>` 时才会挂**。组件没写 scoped style，就没有这个静态属性，它的元素也不会被父组件打标记。

### 3. 样式侧的选择器到底怎么变形的（全量实测）

`compileStyle` 就是拿 PostCSS 跑一个插件，逐个 selector 用正则改写。这里把真实跑出来的结果列全，面试和排查都用得上（表里 `data-v-x` 是 `data-v-abc123` 的简写）：

| 你写的 | 编译后 | 说明 |
| --- | --- | --- |
| `.a { }` | `.a[data-v-abc123] { }` | 常规，权重从 (0,1,0) 变 (0,2,0) |
| `p { }` | `p[data-v-abc123] { }` | 元素选择器也加，权重变 (0,1,1) |
| `#id { }` | `#id[data-v-abc123] { }` | id 也加 |
| `* { }` | `[data-v-abc123] { }` | **`*` 被丢掉了**，只剩属性选择器 |
| `.a, .b { }` | `.a[data-v-x], .b[data-v-x] { }` | 逗号拆分后各加各的 |
| `.a:hover { }` | `.a[data-v-abc123]:hover { }` | 伪类在属性后面 |
| `[data-x] .a { }` | `[data-x] .a[data-v-x] { }` | **只有最右边那一段被加**，左边不受限 |
| `html.dark .a { }` | `html.dark .a[data-v-x] { }` | `html` 本身拿不到标记，别指望限制它 |
| `.a > .b, .c .d { }` | `.a > .b[data-v-x], .c .d[data-v-x] { }` | 每条链只给最后一个复合选择器加 |
| `:is(.a, .b) .c { }` | `:is(.a, .b) .c[data-v-x] { }` | `:is` 内部不动 |
| `:where(.a) { }` | `:where(.a[data-v-x]) { }` | 加在 `:where` 括号里面；因为 `:where()` 权重恒为 0，加了属性选择器权重也不会涨 |
| `:root { }` | `[data-v-abc123]:root { }` | ⚠️ **死规则**，见下面第 5 节 |
| `@keyframes spin { }` | `@keyframes spin-abc123 { }` | 动画名会被改名，见下面第 5 节 |
| `@media` / `@supports` 包裹 | 里面的选择器照常加 | 条件规则不影响 |
| `@font-face` | 原样 | 不加 |

一句话规律：**只给每条选择器链的最后一个"复合选择器"加属性后缀，前面的祖先部分一概不动。**

### 4. 属性侧：`data-v-xxx` 落在哪些元素上

这半边才是踩坑重灾区。规则只有一条核心逻辑：**"这个元素是在哪个组件的 render 里创建的"**，就在哪个组件的标记下。但有几个特殊分支，全部实测结果如下：

| 元素 | 实际拿到的属性 | 原因 |
| --- | --- | --- |
| 自己模板里的普通元素 | `data-v-self` | 常规 |
| 子组件根节点（**单根**） | `data-v-child` + `data-v-parent` | `setScopeId` 里有个 `parentComponent` 分支：如果这个 vnode 就是父组件的 `subTree`，把父的标记也补上 |
| 子组件根节点（**多根 / Fragment**） | 只有 `data-v-child` | 父的 `subTree` 是 Fragment，不满足 `vnode === subTree`，所以**父的标记补不上**（实测 `.m1`/`.m2` 都只有子自己的） |
| 子组件内部元素 | `data-v-child` | 不属于父的 render |
| `<slot>` 传进来的内容 | `data-v-parent` + `data-v-child-s` | 模板在父组件里编译，所以先有父标记；`renderSlot` 再补一个 `-s` 后缀的 |
| `v-html` 注入的 DOM | **无** | 那串 HTML 不是模板编译出来的，运行时也不会遍历它 |
| `<Teleport>` 里的内容 | `data-v-self` | 属性在挂载时打，跟最终插到哪无关，所以照样有 |

两个最容易答错的点：

**① "子组件根节点会被父组件的 scoped CSS 影响"是有前提的——单根。** 官方文档那句"child component's root node will be affected by both"只对单根组件成立。多根组件实测两个根元素都拿不到父的 `data-v-parent`，父组件想通过 scoped 样式控制多根子组件的外观是**做不到**的（得用 `:deep()` 或者让子组件接受 class）。

**② 插槽内容为什么会有一个 `-s` 后缀的属性。** 因为插槽内容的模板属于**父组件**，如果只按父的标记走，子组件里的 `:slotted()` 就永远匹配不上。所以 `renderSlot` 会给渲染出来的 vnode 挂上 `slotScopeIds = [scopeId + '-s']`，运行时再补一个 `data-v-child-s` 属性。样式侧 `:slotted(div)` 也正好编译成 `div[data-v-child-s]`——两边对上了。**没有 `withCtx` 包住的插槽函数拿不到这个 `-s`**，这是我们做实验时第一轮踩到的（用 `this.$slots.default()` 直接调，插槽内容就只有父标记没有 `-s`）。

### 5. 三个逃生舱：`:deep()` / `:slotted()` / `:global()`，以及它们的坑

因为标记是"只加最后一段"，遇到需要穿透的场景就得靠这三个伪类。变形规则实测：

| 你写的 | 编译后 | 语义 |
| --- | --- | --- |
| `.a :deep(.b) { }` | `.a[data-v-x] .b { }` | `.b` 不再被限制，能命中子组件内部 |
| `.a:deep(.b) { }` | `.a[data-v-x] .b { }` | **中间的空格写不写都一样**，别以为有区别 |
| `:deep(.b) { }` | `[data-v-x] .b { }` | 开头就用会先留一个属性选择器 |
| `.a :deep(.b) .c { }` | `.a[data-v-x] .b .c { }` | `:deep()` 之后的部分全都"放行" |
| `:slotted(div) { }` | `div[data-v-x-s] { }` | 打标在插槽内容上 |
| `::v-deep(.a) .b { }` | `[data-v-x] .a .b { }` | 老写法，函数形式还能用 |
| `.a ::v-deep .b { }` | `.a[data-v-x] .b { }` | ⚠️ 会打印废弃警告，别再用组合器形式 |
| `:global(.red) { }` | `.red { }` | 伪类被摘掉，规则变全局 |

`::v-deep` 当组合器用的时候，编译器会给出精确的警告文案，可以直接照着记：

```text
[@vue/compiler-sfc] ::v-deep usage as a combinator has been deprecated.
Use :deep(<inner-selector>) instead of ::v-deep <inner-selector>.
```

然后是三个真实的坑：

**坑一：`:global()` 会把选择器里除它以外的部分全部丢掉——不管放在哪个位置。** 这点非常反直觉，实测四种摆法结果一样，都是只剩 `.b`：

```css
/* 下面几种写法编译结果全都只有 .b，`.a` 和 `.c` 都不见了 */
.a :global(.b) .c { color: red }
:global(.b) .c { color: red }
.a :global(.b) { color: red }
.a:global(.b) { color: red }
```

所以 `:global()` **必须单独用**，不能拿它做"局部限定 + 全局选择器"的组合。真实项目里的表现是"我明明写了限定条件，样式却全站生效了"——而且编译不报错，很难查。

**坑二：`:root { }` 在 scoped 里是死规则。** 编译成 `[data-v-abc123]:root`，但 `<html>` 元素永远不可能有 `data-v-abc123`，所以这条规则一条也匹配不到。想在组件里定义全局 CSS 变量，必须写 `:global(:root)`：

```css
/* ❌ 死规则，什么都不生效 */
:root { --brand: #07c160; }
/* ✅ 正确写法 */
:global(:root) { --brand: #07c160; }
```

**坑三：`@keyframes` 会被改名，而且引用会被一起改。** 实测结果（引用同步改写这点做得挺到位）：

```css
/* 你写的 */
@keyframes spin { from { transform: rotate(0deg) } }
.a { animation: spin 1s linear }
/* 编译后 */
@keyframes spin-abc123 { from { transform: rotate(0deg) } }
.a[data-v-abc123] { animation: spin-abc123 1s linear }
```

改名解决了两个组件都定义 `@keyframes spin` 时后者覆盖前者的问题，而且 `animation` 和 `animation-name` 两种写法都会同步改写。但**如果你在 JS 里动态拼动画名**（`el.style.animation = 'spin 1s'`），编译器管不到，就得自己处理了。

**坑四：`:slotted()` 和 `:deep()` 不能组合用。** 实测两者放在一起时，后一个会被**原样留在 CSS 里**，浏览器不认识 `:deep()` 这个伪类 → 整条规则直接失效：

```css
/* 你写的 */  :slotted(.a) :deep(.b) { }
/* 编译后 */  .a[data-v-abc123-s] :deep(.b) { }   /* :deep() 没被处理，规则废了 */
```

需要"给插槽内容里的深层元素设样式"时，正确做法是把类名下发给插槽，或者干脆把那段样式放到插槽内容所属的父组件里写。

### 6. `v-bind()` in CSS 和 CSS Modules（顺带记住机制）

`<style>` 里能写 `v-bind()`，它的实现是把值变成 CSS 变量。实测：

```css
/* 源码 */
.a { color: v-bind(txtColor); }
.b { --x: v-bind('theme.color'); }
```

```css
/* CSS 产物：变量名带上了 scope 哈希前缀 */
.a[data-v-abc123] { color: var(--abc123-txtColor); }
.b[data-v-abc123] { --x: var(--abc123-theme\.color); }
```

```js
// JS 产物：useCssVars 的 key 和 CSS 变量名一模一样
import { useCssVars as _useCssVars } from 'vue'
_useCssVars(_ctx => ({
  "abc123-txtColor": (txtColor.value),
  "abc123-theme\.color": (_ctx.theme.color)   // 点号会保留（带反斜杠转义）
}))
```

两个要点：**变量名前面拼了 scope 哈希**，所以两个组件各写 `v-bind(color)` 不会互相污染；**变量是内联在组件根元素上的**，而 Teleport 出去的元素靠 `data-v-owner="<组件uid>"` 这个属性被 `useCssVars` 找到并一起更新（实测传送出去的元素上也带了同样的内联变量）。顺带提醒：如果 `useCssVars` 从不生效，先确认你引的 `vue` 入口不是 SSR 那个构建——node 条件下解析到的 `vue` 是服务端构建，`useCssVars` 在里面是个空函数。

CSS Modules 是另一条路：`<style module>` 不靠属性选择器，而是**把类名本身哈希掉**，再把"原名 → 哈希名"的映射通过 `$style` 注入组件实例。实测产物：

```text
源码：<style module>
      .red { color: red }
      .blue-x { color: blue }
```

```text
CSS 产物：._red_1cpek_1 { color: red }
          ._blue-x_1cpek_1 { color: blue }
映射对象：$style = { red: "_red_1cpek_1", "blue-x": "_blue-x_1cpek_1" }
```

名字本身就唯一了，所以不需要 `data-v` 那套标记。两者也不是互斥的——`<style module scoped>` 实测会同时哈希类名**并**加上属性后缀（`._red_1cpek_1[data-v-abc123]`），只是实际项目里一般二选一。

### 7. 性能与权重：为什么官方特意说"别丢掉 class"

官方文档在 "Scoped Style Tips" 里有一句容易被忽略的话：scoped 之后 `p { color: red }` 会比原来慢很多倍，改成 `.example { }` 就基本没开销。原因是浏览器的选择器匹配是**从右往左**的：看到 `p` 可以走"标签名 → 元素集合"的快速索引，而 `p[data-v-abc123]` 这个组合没法用索引，只能逐个候选元素去比对属性。

另一半影响在**权重**上，这才是项目里真会被坑到的：

```text
.a[data-v-x]      → (0, 2, 0)
p[data-v-x]       → (0, 1, 1)   ← 元素选择器 scoped 后权重反而超过了单个类
.foo              → (0, 1, 0)
:global(.bar)     → (0, 1, 0)
```

所以一个组件里写 `p { color: red }`，能盖掉全局样式表里的 `.text { color: blue }`。排查"我的全局样式在这个页面不生效"时，第一反应就应该是去数一下对方的权重。

`:deep()` 还有抬高权重的副作用：`.a :deep(.b)` 编译成 `.a[data-v-x] .b`，权重 (0,2,0)，比直接用 `.b` 高。子组件内部想覆盖父组件用 `:deep()` 写进来的样式，经常要再加一层或 `!important`，根因就在这。

### 8. 面试怎么答（口述版，可以直接背）

> **scoped 不是真正的隔离，它是编译期和运行时配合出来的"标记匹配"。**
>
> 编译期由 PostCSS 把选择器改写成 `[data-v-xxx]` 后缀形式，运行时由组件上的 `__scopeId` 静态属性在挂载元素时把同名属性写到 DOM 上。两边共用构建插件按文件路径算出来的哈希 id，属性选择器一匹配，样式就只作用于本组件的元素了。
>
> 需要穿透的时候用 `:deep()`——它会把属性后缀从"最后一段"挪到 `:deep()` 前面那段，后面就不再加限制。插槽内容用 `:slotted()`，编译出来的是 `[data-v-xxx-s]`，那个 `-s` 是 `renderSlot` 在运行时补的。
>
> 要注意的边界：子组件根节点会同时带父子的标记（仅单根），`v-html` 的内容拿不到标记，`:root` 在 scoped 里是死规则得写 `:global(:root)`，以及 scoped 会让元素选择器权重变高、匹配变慢，所以官方建议还是用 class。

## 其实你每天都在用

- DevTools 里元素上那些 `data-v-7ba5bd90` 属性，就是这套机制留下的痕迹。
- 组件库用 `:deep()` 覆盖样式，本质就是把属性后缀挪到前面，让被覆盖的选择器"脱管"。
- `<style scoped>` 和 `<style>` 同时写在一个文件里做"局部 + 全局"混用，是最常见的一行代码。
- 两个组件都写了 `@keyframes spin` 却不冲突，是因为编译时被改名成了 `spin-<hash>`。
- `v-bind(color)` 在主题色切换时能响应，靠的是 `useCssVars` 往根元素写 CSS 变量。
- 递归组件里 `.a .b` 这种后代选择器会误伤所有递归子组件内部的 `.b`，官方专门提示过。
- 换肤系统里 `:global(:root) { --primary: ... }` 这类写法，就是被 `[data-v-x]:root` 这个死规则逼出来的。

## 常见误解（FAQ）

**❌ 误区1："scoped 的实现跟 Shadow DOM 差不多。"**
差得远。Shadow DOM 是浏览器原生的样式作用域，外面进不来、里面出不去，靠的是 shadow tree 边界。scoped 只是加了个属性选择器——外面的全局样式照样能命中你的元素（只要权重够），你的样式也能通过 `:deep()` 出去。真正的差别是：**Shadow DOM 是隔离，scoped 只是限定了选择器的匹配范围。**

**❌ 误区2："Vue 会给每个组件内的每个元素都打上 `data-v`。"**
给"自己模板里的元素"打是对的，但漏了两个例外：`v-html` 创建的 DOM 拿不到；子组件内部元素拿到的是子组件自己的 id，不是父的。另外多根子组件的根元素**拿不到父组件的 id**，这点很多人以为跟单根一样。

**❌ 误区3："`data-v-xxx` 是模板编译时 inline 到 vnode props 里的。"**
在 Vue 3.5 里实测不是。`compileTemplate({ scoped: true })` 的产物里没有任何 `data-v`，属性完全由运行时根据 `__scopeId` 在挂载时写入。所以"跟着编译产物读一遍就懂了"这种说法会把机制讲反。

**❌ 误区4："用 `:deep()` 就能随便覆盖子组件样式，权重不会变。"**
`:deep()` 会让属性选择器落在前面那段上，权重反而变高（`.a[data-v-x] .b` = (0,2,0)）。所以经常出现"我用了 `:deep()` 却没盖住外面的 `!important`"或者"两边都在加 `:deep()` 越写越长"的情况。

**❌ 误区5："`scoped` 之后就不用管类名了，反正自动隔离。"**
官方明确说了不能省。`p { }` scoped 后变成 `p[data-v-x]`，权重升到 (0,1,1)、匹配走不了标签索引，既可能在样式优先级上出意外，也真的更慢。老老实实用 class。

**❌ 误区6："`:global()` 只是把某个选择器变全局，其余部分保持不变。"**
实测不是。`:global()` 会把同一条选择器里除它以外的部分**全部丢掉**，而且不管放在开头、中间还是紧贴着写，结果都一样——`.a :global(.b) .c`、`:global(.b) .c`、`.a:global(.b)` 编译出来都只剩 `.b`。这条在真实项目里表现为"我写了限定条件但样式全站生效了"，编译不报错，非常难查。

**❌ 误区7："开发环境看到的 `data-v-xxx` 就是线上那个。"**
不是。生产环境的 hash 是用"文件路径 + 源码"算的，开发环境只用路径。所以改一行源码生产环境的 id 就变了，跨环境对着 id 排查会白费功夫。

## 一句话总结

**scoped = 编译期给选择器加 `[data-v-xxx]` 后缀 + 运行时给元素打 `data-v-xxx` 属性，靠同一个哈希把样式和元素对上；它是"限定匹配范围"的属性选择器技巧，不是 Shadow DOM 那种真隔离。**
