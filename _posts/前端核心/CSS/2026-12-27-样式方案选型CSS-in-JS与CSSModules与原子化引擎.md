---
layout: post
title: "样式方案选型：CSS-in-JS / CSS Modules / 原子化引擎"
date: 2026-12-27 00:00:00 +0800
categories: ["前端核心", "CSS"]
tags: ["CSS-in-JS", "CSS Modules", UnoCSS, 零运行时, 样式方案选型, RSC]
description: >
  所有样式方案只差一个变量：样式在哪一刻被算出来。搞清楚"运行时 vs 构建期"、产物形态和 RSC 边界，选型题就变成了四个有顺序的问题。
---

## 一句话概括

市面上的样式方案眼花缭乱，但**它们只差一个变量：样式在哪一刻被算出来**。

- **构建期算**：`.module.css`、`.css.ts`、原子类工具 → 产物是静态 `.css` 文件，JS 里只剩一个类名字符串
- **运行时算**：styled-components、Emotion → JS 在每次渲染时算样式，再往 DOM 里插 `<style>`

这一个变量决定了后面几乎所有事：**客户端包体积、SSR 注水复杂度、能不能进 React Server Component、能不能靠 props 直接算样式**。本篇只做横向对比与选型决策，单个方案的机制细节见同系列的 Tailwind 篇与 BEM 篇。

四个主流方案摆在同一个坐标系里：

| 方案 | 样式算出来的一刻 | 产物 | 客户端运行时 | 能进 RSC |
| --- | --- | --- | --- | --- |
| CSS Modules | 构建期 | 哈希类名 + 静态 `.css` | 无 | ✅ |
| 运行时 CSS-in-JS（styled-components / Emotion） | **每次渲染** | JS 里的类名 + 注入的 `<style>` | 有 | ❌ 必须 `'use client'` |
| 零运行时 CSS-in-JS（vanilla-extract / Linaria / Panda CSS / Pigment CSS） | 构建期 | 静态 `.css` + 一个类名 | 无 | ✅ |
| 原子化引擎（Tailwind / UnoCSS） | 构建期（扫源码按需生成） | 静态 `.css` | 无 | ✅ |

看清楚了吗——**四个方案里只有一个是"运行时"**。所以真正的选型题不是"哪个库好"，而是"我能不能接受运行时"，以及"我的团队把样式写在哪儿"。

## 核心知识点

### 1. 运行时 CSS-in-JS 到底贵在哪（按严重程度排序）

它不是"慢一点"，四笔成本是叠加的：

**① 渲染期计算 + 注入。** 一个 `styled.div` 组件，每次遇到**没见过的新样式组合**都要：算样式对象 → 序列化成 CSS 文本 → 生成类名 → 往文档里插 `<style>`。最要命的是最后一步会**让浏览器重新算样式（style recalculation）**——这是要碰主线程的。同一个组件同样的 props 第二次会命中缓存，所以成本主要落在"首屏 + 大量变体"上。

**② SSR / 流式渲染要额外收集与序列化。** 服务端渲染时不能直接插 DOM，得先收集再喷进 HTML：styled-components 走 `ServerStyleSheet`，Emotion 走 `@emotion/server`，Next.js App Router 里还要一个客户端注册表组件配合 `useServerInsertedHTML` 把样式刷进流里。没配好就是 **FOUC（先渲染出无样式内容，JS 跑起来才补上）**。原子类和 CSS Modules 完全没这一步——它的 `.css` 就是构建产物，正常 `<link>` 进来即可。

**③ 客户端包体积。** 库本身要进 bundle，且它是**为样式服务的运行时**，不产生业务价值。

**④ 并发渲染下开销被放大。** React 的并发/可中断渲染意味着一次提交可能被重放，运行时计算多了一次被重复触发的机会（缓存能救掉绝大部分，但复杂度上去了）。

顺带一个加分细节：**React 18 专门为这类库加了 `useInsertionEffect`**——它的官方用途就是"在 layout effect 之前把样式插进 DOM"，避免"注入样式 → 触发一次布局 → 又注入"这种来回抖动。**同样是运行时方案，有没有用好这个钩子，性能差别很大**，这也是 2025 年前后社区出现一批"给旧库补 `useInsertionEffect` 的 fork"的原因。

**它换来的东西只有一个，但很强：样式可以依赖任意 JS 值。** 主题对象、props、甚至从接口拿到的颜色，都能直接参与样式计算：

```js
// 运行时方案独有能力：样式直接吃 JS 值
const Button = styled.button`
  background: ${(props) => props.theme.brandColor};
  color: ${(props) => (props.$primary ? '#fff' : '#333')};
`;
```

**这就是它的历史功劳，也是它今天的主要包袱**——因为"吃任意 JS 值"这件事 90% 的场景其实用 CSS 变量就够了。

**业界动向值得记一句**（面试说出来档次不一样）：Material UI 在 v6 引入 **Pigment CSS**——官方定位就是"零运行时引擎，用来替代 Emotion / styled-components"，理由是避免客户端重新计算、解锁 RSC 兼容、显著减小包体积。同时 styled-components 的核心维护者已在 **2025 年 3 月** 宣布项目进入**维护模式**（API 不再变、只做 bugfix），公开表态是 *"For new projects, I would not recommend adopting styled-components or most other css-in-js solutions."* 给出的理由里第一条就是 **Context API 在 RSC 里不可用且没有迁移路径**。**连最大的受益方和最早的开拓者都在往构建期搬。**

### 2. RSC 边界：为什么这条是"终局判据"

运行时 CSS-in-JS **天然进不了 Server Component**，而且是架构问题不是配置问题：

- 它靠 **React Context** 传主题（`ThemeProvider` / `CacheProvider`），而 Server Component 里没有 Context
- 它靠 **DOM 注入**样式，而 Server Component 里没有 DOM、没有 `document`

所以每个用了 `styled.xxx` 的文件都得加 `'use client'`。麻烦的是**边界会往上传染**：一个 Button 变成客户端组件，渲染它的父组件也得是客户端组件……最后你会发现"为了数据获取引入的 RSC"，在样式层被整个抵消掉了。

而静态 CSS 和原子类的产物是**构建期生成的独立 `.css` 文件**——它不参与组件树、不跨越 server/client 边界，从根上就没有这个问题。

**一个容易说错的点**：`'use client'` 不等于"客户端渲染"。它是**客户端边界**的标记，被标记的组件在 SSR 时照样会在服务端渲染出 HTML。所以"运行时 CSS-in-JS 在 RSC 里不能用"的准确理由是"它需要 Context 和 DOM 注入"，而不是"它只能在浏览器跑"。

### 3. CSS Modules：机制、产物，以及"动态样式"的正确赔钱买法

CSS Modules 的做法是**把类名本身哈希掉**，再通过构建工具把"原名 → 哈希名"的映射表交给 JS：

```css
/* Button.module.css —— 你写的 */
.button { padding: 8px 16px; border-radius: 6px; }
.primary { background: var(--color-primary); }
```

```js
// 你 import 到的是"映射表"，不是样式
import styles from './Button.module.css';
// styles.button -> 'y8oHnq_button'（本机用 lightningcss 实测的产物名）
```

构建产物（本机实测）是这样的——一份独立的静态 CSS，加一张"原名 → 哈希名"的映射表：

```css
/* 独立的静态 CSS 文件，类名被加哈希前缀 */
.y8oHnq_button { padding: 8px 16px; border-radius: 6px; }
.y8oHnq_primary { background: var(--color-primary); }
```

```json
{ "button": { "name": "y8oHnq_button" }, "primary": { "name": "y8oHnq_primary" } }
```

**"零运行时"是真的零**：JS 侧只是在读一个字符串，没有任何样式计算。

它的短板也是真的：**样式不能直接吃 JS 值**。所以"根据 props 变样式"一共只有三种写法，第一种是坑：

{% raw %}
```jsx
// ❌ 拼类名：类名被哈希后拼出来的名字根本不存在，映射表里也没有这个 key
<button className={styles[`btn-${size}`]} />

// ✅ 写死映射表：所有组合都是构建期已知的
const SIZE_CLASS = { sm: styles.sm, lg: styles.lg };
<button className={`${styles.btn} ${SIZE_CLASS[size]}`} />

// ✅ 量最大的动态值（颜色/尺寸/进度）走 CSS 变量
<button className={styles.btn} style={{ '--progress': `${p}%` }} />
```
{% endraw %}

```css
.btn::after { width: var(--progress, 0%); }   /* 动态值交给 CSS 变量，样式表不动 */
```

**记住这个分工**：**枚举值用类名（构建期确定），连续值用 CSS 变量（运行时不碰样式表）**。这条同样适用于原子类——它俩在这一层的思路是一模一样的。

顺带：CSS Modules 对 CSS 能力**没有限制**，`@keyframes`、`@container`、嵌套、`@layer` 都能正常用——这也是它在"设计系统 + 复杂组件"场景里一直很稳、甚至在"从运行时方案迁出来"的浪潮里被当作首选落点的原因。

### 4. 零运行时 CSS-in-JS：把 DX 留下、把运行时删掉

这一支的动机就一句话：**我想要"在 TS 里写样式 + 令牌类型安全 + 变体函数"的开发体验，但我不想要客户端运行时。**

```ts
// vanilla-extract 风格：写法是 TS，产物是 .css
import { style, createThemeContract } from '@vanilla-extract/css';

export const vars = createThemeContract({ color: { brand: null } });

export const button = style({
  background: vars.color.brand,
  borderRadius: 6,
});
// 构建后：一个静态 css 文件 + 一个类名字符串，运行时零成本
```

同一个阵营里还有 Linaria（保留 styled 的模板字符串写法）、Panda CSS、以及 MUI 的 Pigment CSS——**它们的共同点是：语法像 CSS-in-JS，产物是普通 CSS。**

| 你得到的 | 你付出的 |
| --- | --- |
| TS 类型安全的令牌（写错 token 直接编译报错） | 必须接构建插件（Babel / SWC / Vite），构建变慢 |
| 变体 API（recipe / cva 风格的组合） | 动态值只能走 CSS 变量，不能真的"用 JS 算样式" |
| 零运行时 + RSC 兼容 + 产物可被常规 CSS 工具链优化 | 生态比 Tailwind 小、调试时多一层构建产物 |

**怎么一眼认出它**：看它有没有一个构建插件，以及**构建产物里有没有 `.css` 文件**。有 `.css` → 构建期方案；只往 `<head>` 里插 `<style>` → 运行时方案。

### 5. UnoCSS：它是"引擎"，不是"框架"

UnoCSS 和 Tailwind 解决同一类问题（按需生成原子类），但设计哲学完全不同，**面试答"UnoCSS 就是 Tailwind 的替代品"会丢分**。

**第一，zero-core。** UnoCSS 的核心**不含任何工具类**——连 `m-1`、`flex` 都是预设提供的。核心只做一件事：把源码里的 token 拿去做规则匹配，匹配上就生成 CSS。所以你换 `presetWind4` 就是 Tailwind 风格，写自己的规则就是自己的设计系统。

```ts
// 没有的类就自己加，规则可以是一个函数
export default defineConfig({
  rules: [
    [/^m-([.\d]+)$/, ([, num]) => ({ margin: `${num}px` })],
  ],
});
```

**第二，匹配是"静态表 + 正则"，不解析 AST。** 这是它宣称比 Tailwind JIT 快数倍的原因——不做 CSS 树的转换，纯字符串匹配。

**第三，按需是真按需**（本机实测）：空的源码进 `generate()`，出来的 CSS 就是空字符串；只用 `flex items-center`，产物就只有这两条：

```css
.flex { display: flex; }
.items-center { align-items: center; }
```

顺手说清楚它的两个特色能力（都实测过）：

```html
<!-- attributify：把工具类写成属性，模板更短 -->
<div flex items-center p-4><span text-sm>hi</span></div>
```

```css
/* 产物会同时给出类选择器和属性选择器 */
.flex, [flex=""] { display: flex; }
.items-center, [items-center=""] { align-items: center; }
.p-4 { padding: calc(var(--spacing) * 4); }   /* 用主题变量，和 Tailwind v4 一个思路 */
```

```ts
// shortcuts：把一串类名打包成一个语义类
shortcuts: {
  btn: 'inline-flex items-center px-4 py-2 rounded',
  'btn-primary': 'btn bg-blue-500 text-white',
}
```

```css
/* 实测产物：类名用的是 shortcut 名，而且所有属性合进"一条"规则（不会留一个 .btn 规则再引用它） */
.btn-primary {
  display: inline-flex; align-items: center; border-radius: 0.25rem;
  background-color: rgb(59 130 246 / var(--un-bg-opacity));   /* bg-blue-500 展开 */
  padding-left: 1rem; padding-right: 1rem;                    /* px-4 */
  padding-top: 0.5rem; padding-bottom: 0.5rem;                /* py-2 */
  color: rgb(255 255 255 / var(--un-text-opacity));
}
```

**第四，它和 Tailwind 一样有"动态类名"这个死穴**（本机实测）：源码里写 `bg-${color}-500`，`generate()` 匹配到的类是 **0 条**——因为它把源码当**纯文本**扫，切出来的碎片 `bg-`、`-500` 谁都匹配不上。动态值要么走 CSS 变量，要么在映射表里穷举。

**它们的真正差异，一句话概括**：Tailwind 是**框架**（自带一整套设计系统 + 一套类名规范），UnoCSS 是**引擎**（默认什么都没有，全靠 preset 和你的规则）。团队想要"开箱即用的设计语言"选 Tailwind；想要"自己的设计系统 + 只生成用到的类"选 UnoCSS。

### 6. 选型：四个有顺序的问题

按顺序问，问完基本只剩一个答案：

| 顺序 | 问题 | 结论 |
| --- | --- | --- |
| 1 | 有 SSR / RSC / 流式渲染，或在意客户端包体积吗？ | 是 → **运行时 CSS-in-JS 直接出局** |
| 2 | 样式的变化来自"有限枚举"还是"运行时连续值"？ | 枚举 → 类名（CSS Modules / 原子类）；连续 → CSS 变量 |
| 3 | 设计系统需要 TS 层的令牌类型安全吗？ | 需要 → 零运行时 CSS-in-JS；不需要 → CSS Modules / 原子类 |
| 4 | 团队接受"样式写在模板上"吗？ | 接受 → 原子类；想保持样式独立成文件 → CSS Modules |

第 3、4 条其实是**协作问题而不是技术问题**：原子类把样式决策搬到了 JSX 里，好处是"改样式时不用跳文件"，坏处是模板变长、`cn()` 满天飞；CSS Modules 反过来，样式集中可读，但组件和样式两个文件来回切。**这没有对错，只有团队习惯**。

只有第 1 条是硬约束，剩下的都是偏好。

### 7. 混合才是常态（这也是最真实的答案）

真实项目很少"只用一种"，常见分层是这样：

| 层次 | 用什么 | 为什么 |
| --- | --- | --- |
| 布局 / 间距 / 响应式骨架 | 原子类 | 变化频繁、组合多，写 CSS 文件得不偿失 |
| 复杂组件内部（动画、伪元素、多层状态） | CSS Modules 或普通 CSS | 原子类表达不了 `::before` 里的复杂逻辑 |
| 主题 / 设计令牌 | **CSS 变量** | 三层方案互不冲突的公共接口，运行时切换零成本 |
| 组件库对外 API | 类名 + CSS 变量 | 让使用者能覆盖，又不用暴露内部结构 |

**这四层能共存的关键是 CSS 变量**：原子类生成的类名、CSS Modules 的类名、运行时切换的主题，最后都指向同一套 `--xxx` 变量。所以"选型"这件事的正确答案往往不是选一个，而是**把令牌层固定成 CSS 变量，其余各层按上表分工**。

## 其实你每天都在用

- **Ant Design 5 的 `ConfigProvider` 换主题**：它底层是运行时 CSS-in-JS，但提供了 `cssVar` 配置把令牌落到 CSS 变量上，后续版本还在推 `zeroRuntime` 模式——方向就是"把运行时的活挪到构建期"。
- **Next.js App Router 项目里那个 `useServerInsertedHTML` 注册表**：它存在的唯一理由就是给运行时 CSS-in-JS 补 SSR 样式收集，用 CSS Modules 的项目从来不需要它。
- **Vite 里 `import styles from './x.module.css'` 然后 `className={styles.card}`**：这就是 CSS Modules 的全部日常——JS 里只流通字符串。
- **shadcn/ui 那一句 `cn(...)`**：它把"条件类名"和"原子类"拼起来用，是当下最流行的组合。
- **Element Plus / Ant Design 的一堆 `--el-color-primary` / `--ant-color-primary`**：组件库把可变部分全部收敛到 CSS 变量，就是为了不把运行时塞进业务代码。
- **某次把类名拼成 `` `bg-${color}-500` `` 后上线发现样式没了**：Tailwind / UnoCSS 都扫不到这种动态构造，这就是"动态类名"这个坑的真实长相。
- **Vue 项目的 `<style scoped>`**：它是另一种"构建期隔离"（属性选择器 + 运行时打 `data-v-xxx`），和 CSS Modules 的类名哈希是两条路。
- **设计稿更新后只改一份 token 文件**：这是"令牌落到 CSS 变量"最直接的收益——不用改任何组件代码。

## 常见误解（FAQ）

**❌ 误区一："CSS-in-JS 已经死了"**

死的是**运行时**那一支，不是整个范式。要分三段看：运行时方案在退场（MUI 用 Pigment CSS 替代 Emotion、styled-components 进入维护模式）；**构建期提取的方案反而在长**（vanilla-extract / Panda CSS / Linaria / Pigment CSS）；而"在 JS/TS 里描述样式"这个**写法**本身活得很好——只是计算从运行时挪到了构建期。准确的说法是"运行时的样式计算在退场"。

**❌ 误区二："CSS Modules 也是 CSS-in-JS"**

不是。CSS-in-JS 的定义是"样式用 JavaScript **计算**出来"，CSS Modules 里 JS 一个样式都没算——它只是读了一张构建工具给的类名映射表。这个区别直接决定了：CSS Modules 零运行时、天然兼容 RSC，而运行时 CSS-in-JS 两条都做不到。

**❌ 误区三："给 styled-components / Emotion 配好插件就能在 Server Component 里用"**

不行，这是架构冲突不是配置问题。它们需要 **React Context** 传主题、需要 **DOM** 注入样式，而 Server Component 两者都没有。唯一解是加 `'use client'` 把组件变成客户端边界，但那会**往上传染**——被标记组件渲染的子组件全都落到客户端，RSC 的收益被抵消。想保留 RSC 就得换静态方案。

**❌ 误区四："UnoCSS 就是 Tailwind 的社区克隆"**

定位不同：Tailwind 是**框架**（自带完整设计系统 + 固定类名规范），UnoCSS 是**引擎**（zero-core，核心不含任何工具类，所有能力来自 preset 和你自己写的规则）。UnoCSS 的规则可以是函数、可以动态匹配，所以它能直接生成你公司的设计系统；代价是**默认什么都没有**，用之前得先挑 preset。

**❌ 误区五："原子类和 CSS Modules 是二选一"**

现实中混着用，而且它们**不冲突**。常见分工是：原子类负责布局/间距/响应式骨架，CSS Modules 或普通 CSS 负责复杂组件内部，两边共享同一套 CSS 变量做主题。真正需要小心的是"同一层里两套体系互相打架"（比如原子类的类名和手写 CSS 抢同一个属性），这时候用 `@layer` 显式排好层序就行——原子类工具本来就会把自己放进 `utilities` 层。

**❌ 误区六："零运行时 CSS-in-JS 和 CSS Modules 差不多"**

产物确实像（都是静态 CSS），差别在**开发期的约束能力**：零运行时方案能在编译期校验"你用的令牌/变体是否存在"，并且提供变体组合 API（写错 token 直接报错，写错变体也报错）；CSS Modules 里令牌名就是个字符串，写错了要等浏览器里看出问题。**代价是构建复杂度**——它必须挂着 Babel/SWC/Vite 插件，构建本来就慢的项目要算这笔账。

**❌ 误区七："样式方案无所谓，反正都会被打包工具处理掉"**

方案决定了三件打包救不了的事：**客户端要不要为样式跑 JS**、**SSR/流式渲染要不要额外的样式收集机制**、**能不能进 Server Component**。这三条都是架构级差异，"反正能构建"并不能抹平。

## 一句话总结

**样式方案的分水岭只有一条——样式在构建期算还是运行时算：运行时 CSS-in-JS 换来了"样式能直接吃 JS 值"，代价是客户端运行时、SSR 样式收集和进不了 RSC，所以 MUI 都在往零运行时（Pigment CSS）搬；CSS Modules 与零运行时 CSS-in-JS 都是"构建期哈希类名 + 静态 CSS"，区别只在令牌类型安全与构建插件；原子化引擎（Tailwind / UnoCSS）是"扫源码按需生成"，而 UnoCSS 是引擎不是框架（zero-core、规则可编程）。选型按四个有顺序的问题走，其中只有"有没有 SSR/RSC"是硬约束，剩下的都是团队偏好——最终答案通常是分层混合，而把令牌固定成 CSS 变量是各层共存的公共接口。**
