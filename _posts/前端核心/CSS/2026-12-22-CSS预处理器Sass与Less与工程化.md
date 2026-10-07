---
layout: post
title: "CSS 预处理器：Sass / Less 与工程化"
date: 2026-12-22 00:00:00 +0800
categories: ["前端核心", "CSS"]
tags: [Sass, Less, CSS预处理器, "@use", 变量作用域]
description: >
  面试问预处理器，八成人只会背「Sass 用 $、Less 用 @」。真正的考点是变量作用域和延迟求值这两个坑、@use 为什么取代 @import、@extend 为什么官方劝你别用。
---

## 一句话概括

CSS 预处理器就是一门**会编译成 CSS 的小语言**：浏览器不认识 `$color`、不认识嵌套，预处理器的编译器先把它们算明白、展开成纯 CSS，浏览器才看到。它解决的痛点只有一个 —— 原生 CSS 没有变量、没有复用、没有逻辑，样式一多就靠复制粘贴活着。

面试为什么问它？因为**大部分人的答案是错的**。网上流传最广的那套对比（"Sass 没有全局变量""Less 作用域最差"）是十几年前 Ruby Sass 时代的说法，今天用 dart-sass 跑一遍结论会反过来。面试官问这个题，其实是在看你有没有真的写过、有没有踩过坑。

这篇不讲 api 手册，按面试深挖的顺序讲五件事：**能力边界**、**变量作用域**、**编译期 vs 运行期**、**模块系统**、**`@extend` 陷阱**。

## 核心知识点

### 1. 先看能力：预处理器到底加了什么

一句话：**它是 CSS 的超集**，你写的一切合法 CSS 都是合法 SCSS，它只是额外给了你四个能力。

| 能力 | 作用 | 编出来的 CSS |
| --- | --- | --- |
| 变量 | 统一管理颜色/尺寸 | 值被替换进去 |
| 嵌套 | 按 DOM 结构写样式、用 `&` 拼选择器 | 展平成后代选择器 |
| Mixin | 可传参的样式块（复用） | 样式被复制到每个调用点 |
| 控制流 | `@if` / `@each` / `@for` 批量生成 | 展开成一条条规则 |

关键结论先记住：**变量和嵌套是给"人"看的，Mixin 和控制流是给"机器"看的**——前两个只是写法糖，后两个是真正的生成能力（批量造一套工具类、按配置生成主题）。

```scss
// 一行代码生成一整套间距工具类 —— 这是原生 CSS 做不到的（CSS 没有循环）
$spacers: (sm: 4px, md: 8px, lg: 16px);
@each $name, $v in $spacers {
  .p-#{$name} { padding: $v; }
}
```

```css
/* 编译产物（本例用 dart-sass 实跑，下文所有"实测"均由 dart-sass / less 4 本机跑出） */
.p-sm { padding: 4px; }
.p-md { padding: 8px; }
.p-lg { padding: 16px; }
```

### 2. 变量作用域：Sass 和 Less 的答案完全相反

这是第一个高频追问点，也是网上错误答案最多的地方。

**Sass 的规则（官方文档原话）：** *"Variables declared at the top level of a stylesheet are global... Those declared in blocks are usually local."* —— **块里的变量是局部的，会遮蔽外部的同名变量**，块外面拿不到。

```scss
$c: red;
.a { $c: blue; color: $c; }  // 这个 $c 是新的局部变量
.b { color: $c; }            // 仍是全局的 red
```

```css
/* 实测产物：互不影响 */
.a { color: blue; }
.b { color: red; }
```

想在块里改全局值，必须显式写 `!global`（且官方限制：**只能改已存在的全局变量，不能靠它新建**）：

```scss
$c: red;
.a { $c: blue !global; color: $c; }
.b { color: $c; }   // 现在变成 blue 了
```

**Less 的规则完全反着来：** Less 变量是**延迟求值（lazy evaluation）**的——*在作用域内按"最后一次定义"算*，而且**不看书写顺序**。

```less
@size: 40px;
.a { width: @size; }
@size: 60px;
.b { width: @size; }
```

```css
/* 实测产物（less 4 实跑）：两个都是 60px！ */
.a { width: 60px; }
.b { width: 60px; }
```

`.a` 写在 `@size: 60px` 之前，但拿到的仍是 60px。**这就是 Less 的招牌坑**：把一个变量文件 `@import` 在两处、后一处改了值，前面所有引用会一起变。所以官方自己都建议：**别到处重新定义同名变量**。

对比表（可以背）：

| 维度 | Sass / SCSS | Less |
| --- | --- | --- |
| 变量符号 | `$name` | `@name` |
| 作用域模型 | 词法作用域 + 遮蔽，块内是新的局部变量 | 延迟求值，作用域内按最后一次定义算 |
| 改全局 | 需要 `!global` | 直接重新定义即可 |
| 循环/条件 | 原生 `@each` `@for` `@if` | 没有 if，靠 `when` 守卫 + 递归模拟 |

补充一条 Sass 的例外：**`@if` / `@each` 这类流控制里赋值不算新变量**，会直接改外层已有的变量——这是官方特意设计的，方便条件赋值。所以下面这段能跑通：

```scss
$theme: dark;
$bg: white;
@if $theme == dark { $bg: black; }
.a { background: $bg; }   // 实测：black
```

### 3. 延迟求值 ≠ 运行时：别把 Sass 变量当成 CSS 变量

第二个高频追问："Sass 变量和 CSS 变量有什么区别？"——标准答案是**编译期替换 vs 运行期生效**，但光说这句面试官会继续追问，得讲透两个具体差别：

**差别一：Sass 变量是命令式的（imperative）。** 改了值，**之前写过的引用不会跟着变**。

```scss
$v: 1;
.a { width: $v; }   // 永远是 1
$v: 2;
.b { width: $v; }   // 2
```

CSS 变量是声明式的，改一处全体生效——这正是它适合做**主题切换**的原因。

**差别二：Sass 变量全被编译掉了。** 产物里根本没有变量，只有一个算好的字面值。所以你想运行时用 JS 改主题，Sass 变量一点忙都帮不上，必须用 CSS 变量。

面试一句话结论：**Sass 变量管"构建期算清楚"，CSS 变量管"运行期动态改"，两者互补而不是二选一。** 实战里常见组合是 `@each` 把 token 批量的写成 CSS 变量。

还有一个易错点值得点一下：**Sass 里连字符和下划线等价**——`$font-size` 和 `$font_size` 是同一个变量（早期历史遗留）。别在同一个项目里两种写法混用。

### 4. 模块系统：`@use` 取代 `@import`，这是必答项

面试官问"怎么组织大型项目样式"，就等这个答案。

`@import` 有两个致命问题（官方原话概括）：**它把成员塞进全局作用域**（谁都能用，名字冲突只能靠 `$mat-corner-radius` 这种超长命名规避），而且**同一个文件被导入几次就复制几遍 CSS**——这条实测能直接看到，同一个文件 `@import` 两次，产物里 `.dup` 就老老实实出现了两遍：

```scss
@import "dup";
@import "dup";   // 实跑产物：.dup { color: red; } 输出两次
```

`@use` 修掉了这两点：

```scss
// _variables.scss
$radius: 6px;
```

```scss
// style.scss
@use "variables" as v;      // 默认命名空间就是文件名
.btn { border-radius: v.$radius; }
```

三条必须记住的规则（官方原文）：

1. **`@use` 只把成员加载到当前文件**，不会污染全局——没 `@use` 的文件用不了；
2. **同一个模块无论加载几次，CSS 只输出一次**（`:root` 变量不会重复）;
3. **`@use` 必须写在文件开头**，不能嵌套在样式规则里。

配 `!default` 做"可配置的库"，这是组件库改主题的正式姿势：

```scss
// _library.scss
$black: #000 !default;          // 没被配置过才用这个值
$radius: 0.25rem !default;
code { border-radius: $radius; }
```

```scss
// style.scss
@use 'library' with ($black: #222, $radius: 0.1rem);
```

`!default` 的语义要精确：**只在变量"未定义"或"值为 null"时才赋值**（实测 `$c: null; $c: blue !default;` 结果是 blue）。另外 `!default` 的变量**只有写在文件顶层才能被配置**——把 `$inner: 1rem !default` 写进 `.wrap { }` 块里再用 `with` 去配，实测直接报错 `This variable was not declared with !default in the @used module.`，这个坑在封装组件库时很常见。

再确认一下"只输出一次"（实测）：`_base.scss` 里有 `:root` 变量和 `.shared` 规则，`entry.scss` 既 `@use "a"`、而 `_a.scss` 里也 `@use "base"`，最后产物里 `:root` 和 `.shared` **只出现一次**。

`@forward` 是配套工具：给"聚合入口文件"用的（`_index.scss` 把子模块转发出去），或者转发时顺便给内部成员加前缀。

关于废弃现状，面试可以主动说：**`@import` 已进入废弃流程**，dart-sass 会打印 `DEPRECATION WARNING [import]`，官方迁移工具是 `sass-migrator`。新项目没有理由再用 `@import`。

### 5. `@extend` 陷阱：官方明确建议少用

`@extend` 的意思是"A 继承 B 的样式"，配 `%placeholder` 用可以不产出多余选择器：

```scss
%btn-base { display: inline-flex; padding: 0 12px; }
.btn-primary { @extend %btn-base; background: blue; }
.btn-danger  { @extend %btn-base; background: red; }
```

```css
/* 实测产物：共用一条规则，没有多余的类 */
.btn-danger, .btn-primary {
  display: inline-flex;
  padding: 0 12px;
}
.btn-primary { background: blue; }
.btn-danger  { background: red; }
```

看着挺美，但 `@extend` 有四个硬性限制，一个比一个难受：

**坑一：层叠顺序不看你写在哪。** 官方原话：*"their styles have precedence in the cascade based on where the extended selector's style rules appear, not based on where the @extend appears."* —— 继承者拿到的样式，**重排到了被继承者的位置**。这意味着 `@extend` 混着普通规则写时，谁覆盖谁很不直观。

**坑二：不能跨 `@media`。** 实测直接报错 `You may not @extend selectors across media queries.`——因为媒体查询里的选择器没法安全地合并到外面去。**在 `@media` 内部只能扩展同一个 `@media` 内部的选择器**。

**坑三：只能 `@extend` 单个简单选择器。** `.a.b` 这种复合选择器实测报错 `compound selectors may no longer be extended.`；`.main .info` 这类带后代的也不允许。

**坑四：扩展范围受 `@use` 边界约束。** 官方规定：一个样式表 `@extend` 只影响它**上游模块**（自己 `@use`/`@forward` 的、以及那些模块再 `@use` 的）里的规则。这是为了让 `@extend` 的影响范围可预测，但也意味着你在子组件里 `@extend` 一个"碰巧看到过"的选择器，很可能匹配不到而报错。

所以官方给出的取舍口径是：**`@extend` 只适合表达"语义类之间的从属关系"**（`.error--serious` **就是一个** `.error`，所以 `@extend .error` 合理）；**纯粹想复用一段样式，就该用 `@mixin`**。

顺带解决一个常见纠结："mixin 会复制样式、体积更大，是不是不如 `@extend` 省流量？" 官方直接回了一句：**服务器压缩算法对重复文本处理得非常好，mixin 多出来的 CSS 基本不会增加用户下载量**——别为了省体积选 `@extend`。

关于 mixin 的进阶用法，`@content` 是面试加分项（做媒体查询断点最常见）：

```scss
@mixin respond($bp: 768px) {
  @media (min-width: $bp) { @content; }
}
.a { color: red; @include respond(1024px) { color: blue; } }
```

```css
/* 实测：@content 的样式被塞进了媒体查询里 */
.a { color: red; }
@media (min-width: 1024px) { .a { color: blue; } }
```

### 6. 两者的其他真实差异（答题时补刀用）

- **单位运算**：Sass 支持带单位运算并会报错保护（`$w / 2` 得 `50px`，但 dart-sass 现在会打印 `DEPRECATION WARNING [slash-div]`，官方推荐改用 `math.div($w, 2)` 或 `calc($w / 2)`——这个警告**很多人以为是自己的代码有问题**）；**Less 4 默认不再对 `/` 做除法**，实测 `width: @w / 2` 原样输出 `100px / 2`，必须写成 `(@w / 2)` 才算出 `50px`。这是从 Less 3 到 4 的破坏性变更，老项目升级时特别容易翻车。
- **插值语法**：Less 是 `@{name}`，能用在选择器/属性名/字符串里（`.@{name}-title`），还能 `@@name` 取"变量的变量"；Sass 是 `#{$name}`。
- **条件能力**：Less 没有 `@if`，只有 mixin 守卫 `when (@w > 100px)`，批量生成只能靠**递归调用自己**来模拟；Sass 有原生 `@each/@for/@if`。
- **mixin 形态**：Less 可以直接复用类选择器（`.mixin-a;`），但这样**类本身也会输出**；写成 `.mixin-b()` 带括号才不输出自身。Sass 用 `@mixin/@include`，不存在"顺带输出"的问题。

## 其实你每天都在用

- **Element Plus / Ant Design 换主题色**：它们暴露的就是一堆 `$color-primary !default` 变量，你在自己的 scss 里 `@use ... with` 覆盖，这就是 `!default + with` 的标准用法。
- **项目里的 `_variables.scss` / `_mixins.scss`**：这就是模块拆分；如果还是 `@import` 进来的，升级建议就是换 `@use`。
- **写响应式时的 `@mixin respond($bp)`**：断点 mixin 配 `@content`，几乎每个中后台项目都有。
- **`&:hover` / `&__title`**：嵌套加 `&`，前者拼伪类、后者拼 BEM 的 `element` 部分。
- **颜色变深变浅**：`darken()` / `lighten()` 是全站用得最多的内置函数（注意：Sass 已推荐改用基于色相空间的 `color.adjust`）。
- **批量生成 margin/padding 工具类**：`@each` 遍历 map，是"自己动手造一套小 Tailwind"的最短路径。
- **Less 项目里改了一个全局 `@primary-color`**：结果整个页面颜色都变了——这就是延迟求值在起作用，别以为是 bug。

## 常见误解（FAQ）

**❌ 误区一："Sass 作用域最差，没有全局变量这个概念"**

这是十几年前 Ruby Sass 时代的说法，网上现在还大量流传。今天 dart-sass 的规则是：**顶层变量就是全局的，块内变量是局部的并遮蔽全局**（官方文档 "Shadowing" 一节）。真正"作用域最差"的反而是 Less——它的延迟求值会让后面的定义**反向影响**前面的引用，这个坑比 Sass 的遮蔽严重得多。

**❌ 误区二："@extend 比 @mixin 性能好，因为它不复制样式"**

两个错。第一，`@extend` 只是把选择器合并了，**并没有减少规则数量**，只是省了几个选择器名字符；第二，官方明确说过服务器压缩对重复文本很友好，`@mixin` 多出来的体积基本不影响下载。真正该选 `@extend` 的理由是"语义上确实是 is-a 关系"，不是性能。而 `@extend` 的代价（顺序不可控、不能跨 `@media`、不能继承复合选择器）通常比那几个字节贵得多。

**❌ 误区三："预处理器变量能替代 CSS 变量做主题切换"**

不能。Sass/Less 变量**编译后就不存在了**，产物里只有算死的字面值。想运行时换主题（点按钮换深色模式、按系统偏好切换）必须用 CSS 变量——它的值由浏览器在运行期解析，JS 改一下 `style.setProperty` 全站立即生效。预处理器变量只能在**构建期**决定"这个包用哪套主题"，做多主题发版可以，做运行时切换不行。

**❌ 误区四："@use 和 @import 只是写法新旧，功能一样"**

差别是实质性的：`@import` **把成员暴露到全局**、且**同一文件导入几次就重复输出几次 CSS**；`@use` **成员只在当前文件可见**、**每个模块只输出一次**。跨文件重名冲突（两个库都定义 `$radius`）在 `@import` 时代只能靠长命名规避，`@use` 用命名空间从根上解决了。

**❌ 误区五："Sass 变量改了值，前面用它的地方也会跟着变"**

不会。Sass 变量是**命令式**的——`$v: 1` 之后写下的 `width: $v` 就固定成 1 了，后面再把 `$v` 改成 2 也不会回头改它。这一点和 CSS 变量（声明式，改一处全体生效）正好相反，也是两者最本质的区别。

**❌ 误区六："Less 的变量也是就近优先，和 Sass 一样"**

不一样。Less 是**先按作用域找，找到这个名字后按该作用域里最后一次定义算**——所以 `.a { color: @c; @c: green; }` 实测输出 `green`，`@c` 写在后面照样生效。Sass 里同样的写法，`.a` 里拿到的是外层全局的 `red`（块里那个是新局部变量，作用不到前面）。这个对比在面试里非常容易问，也很容易答反。

## 一句话总结

**预处理器是"编译期的 CSS"：Sass 用词法作用域 + `@use` 模块系统把项目管住，Less 用延迟求值埋坑但语法更贴近 CSS；`@extend` 只用于表达语义从属、复用一律用 `@mixin`；需要运行时动态就用 CSS 变量，别指望预处理器变量。**
