---
layout: post
title: "Vue + TypeScript 工程化实践：类型检查到底在哪跑"
date: 2026-12-06 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [Vue TypeScript, vue-tsc, Volar, strictTemplates, 组件库类型]
description: >
  面试向梳理 Vue + TS 的工程链路：为什么 vite build 通过不代表类型安全、vue-tsc 与 Volar 分工、
  strictTemplates 默认关着会漏掉哪四类错误（附实测输出）、vueCompilerOptions 该写在哪份 tsconfig、
  组件库出 d.ts 的正确姿势，以及类型声明宏在运行时到底还剩多少校验能力。
---

## 一句话概括

Vue + TS 的工程化，面试真正想听的不是"我会写 `defineProps<Props>()`"，而是**类型检查在哪跑、谁拦住错误**。

结论先给：**类型检查不在构建里，得单独跑 `vue-tsc`，并且要在 CI 里当门禁。**

原因在于 Vite 用的是 esbuild，它是**单文件转译**——遇到 `lang="ts"` 就把类型注解原地擦掉，不读整个依赖图，也不做类型推导。这带来两个直接后果：

- 类型写错了，`vite build` 照样能过（构建绿 ≠ 类型对）；
- `.vue` 文件的**模板部分**更是完全不检查——模板里引用了不存在的组件、给子组件传了没声明的 prop，构建都不会吭声。

所以"我项目是 TS 写的"和"我项目类型是安全的"是两件事。把这句讲清楚，再补上 `vue-tsc` / Volar / `strictTemplates` 三者的分工，工程化这题基本就稳了。

## 核心知识点

### 1. 先破一个错觉：`vite build` 通过不代表类型正确

官方文档写得很直白：**基于 Vite 的配置里，开发服务器和打包器只做转译，不做任何类型检查**（transpilation-only and do not perform any type-checking）。这么设计是为了让 dev server 保持极快，代价就是类型错误得靠别的工具兜。

我实测了一个最小项目（`vue@3.5.42` + `vue-tsc@3.3.11`），里面故意埋了 4 个模板层的问题：

```text
// App.vue 模板里
<MyButton title="x" :unknown-prop="1" @undeclared-event="..." />   // prop 和事件都没声明
<NeverRegistered />                                                // 组件根本没注册
<div v-undefined-dir>dir</div>                                     // 指令根本不存在
```

| 命令 | 输出 |
|---|---|
| `vite build` / 原生 `tsc --noEmit` | **0 error**（全放过） |
| `vue-tsc --noEmit` | 4 个错误全抓出来 |

原生 tsc 那一格的细节值得记一下：`tsc` 根本不认识 `.vue`，你得先写 `declare module '*.vue'` 它才肯罢休。而这样一来**所有组件都退化成 `DefineComponent<{}, {}, any>`，props 类型信息全丢**——它不是在检查你的组件，只是在"装作认识"。另外如果 `include` 里一个 `.ts` 文件都没有（`env.d.ts` 也算），纯 tsc 会直接报 `TS18003: No inputs were found`。

> 面试可以这么说：**esbuild 只擦类型不做检查，这是设计取舍不是 bug；Vue 官方给的补偿方案就是独立的 vue-tsc。**

### 2. `vue-tsc` 到底是什么，以及它默认有多"宽松"

`vue-tsc` 是 `tsc` 的**包装器**（官方原话：a wrapper around tsc），行为基本一致，差别是它额外支持 `.vue` 单文件组件。所以：

- 它能读 `tsconfig.json`，`strict`、`paths`、`verbatimModuleSyntax` 这些选项全部照旧生效；
- 它多一份自己的配置入口：`vueCompilerOptions`；
- 类型检查场景下要用 `--noEmit`（只查不产出）。

**但重点来了：它默认不检查表里的四类模板问题。** 官方给的 `strictTemplates` 默认值是 `false`，下面这几项默认全关：

| 选项 | 默认 | 管什么 |
|---|---|---|
| `strictTemplates` | `false` | 一键打开下面所有 `checkUnknown*` |
| `checkUnknownComponents` | `false` | 模板里用了未注册的组件 |
| `checkUnknownProps` | `false` | 给组件传了它没声明的 prop |
| `checkUnknownEvents` | `false` | 监听了组件没 `defineEmits` 的事件 |
| `checkUnknownDirectives` | `false` | 用了未注册的自定义指令 |

实测对比，同一个项目、同一份代码：

```text
// 没有 vueCompilerOptions → 0 error（一个都不报）
// 打开 strictTemplates → 4 个错误全出来：

error TS2339: Property 'NeverRegistered' does not exist on type '{}'.
error TS2353: 'unknownProp' does not exist in type
  '{ readonly title: string; } & VNodeProps & AllowedComponentProps & ComponentCustomProps'.
error TS2353: 'onUndeclaredEvent' does not exist in ... 
error TS2339: Property 'vUndefinedDir' does not exist on type '{ title: string } & GlobalDirectives'.
```

注意第二个报错里的类型串 `& VNodeProps & AllowedComponentProps & ComponentCustomProps`——这是**组件 props 的真实类型形状**，Vue 内部属性被交叉进来，所以你不能拿它当"用户声明的 props"用（第 5 节会讲正确的抽取方式）。

配置写哪儿也有讲究：用 `create-vue` 生成的项目有多份 tsconfig，`vueCompilerOptions` 要写进**包含 Vue 源码的那一份**，也就是 `tsconfig.app.json`，别写进根 `tsconfig.json` 或 `tsconfig.node.json`：

```jsonc
// tsconfig.app.json
{
  "compilerOptions": { "strict": true, "verbatimModuleSyntax": true },
  "vueCompilerOptions": {
    "target": 3.5,
    "strictTemplates": true
  }
}
```

`strictTemplates` 全开如果误伤太多（比如 `data-*` 属性会被当成未知属性报错），可以退化成单独开某一项：

```jsonc
{
  "vueCompilerOptions": {
    "checkUnknownComponents": true,   // 先抓"组件名写错/忘了 import"这类硬伤
    "checkUnknownProps": false        // 误伤多的先不动
  }
}
```

### 3. 类型检查其实分三层，别只盯着一层

面试答"Vue 里哪些东西会被类型检查"时，按这三层说，层次立刻清楚：

| 层次 | 检查对象 | 谁负责 |
|---|---|---|
| ① `<script>` 层 | 普通 TS 代码、组合式函数、类型推导 | `tsc` / `vue-tsc` 都会查 |
| ② 模板表达式 | `:prop="a.b"`、`{ { x + 1 } }`、`v-if` 里的表达式 | 只有 `vue-tsc` 查 |
| ③ 模板契约 | 组件是否注册、props / events / 指令是否声明 | 只有 `vue-tsc` **且开了 `strictTemplates`** 才查 |

第一层大家的直觉都对，第二层是"用 vue-tsc 就自动有"，**第三层才是最容易漏的——因为它默认关着**。很多团队"配了 vue-tsc"但其实只覆盖到第二层，白拿了一个宽松的假安全感。

### 4. 编辑器侧：Vue - Official（原 Volar），以及 Vetur 必须卸

- **Vue - Official** 是官方 VS Code 扩展（前身叫 Volar），提供 SFC 内的 TS 支持：补全、跳转、错误提示。Vue 2 时代的 **Vetur 已经淘汰，在 Vue 3 项目里必须禁用**——它不理解 `<script setup lang="ts">`，会对着完全合法的代码报假错。
- 编辑器里看到的是**语言服务**给的即时反馈，`vue-tsc` 是**命令行版的同一套能力**。两者不是替代关系：IDE 负责"写的时候爽"，vue-tsc 负责"合并的时候拦"。
- 顺带一个 2026 年会遇到的变化：**Volar 3.0（vue-language-server 3.x）改了语言服务协议**，Neovim 那套配置如果还挂着 `ts_ls` 会失效，官方建议换 `vtsls`；VS Code 和 JetBrains 只要把扩展/插件升到最新即可。

一句面试话术：**IDE 的报红不是门禁，它挡不住别人把错代码合进主干；`vue-tsc` 才是。**

### 5. 组件库场景：出 d.ts、抽 props 类型

做组件库（或者 monorepo 里的基础包）时，你要对外发的是**类型声明**，不是 `.vue` 源码。做法是让 vue-tsc 走一遍 emit：

```bash
# package.json scripts
"type-check": "vue-tsc --noEmit -p tsconfig.app.json",
"build:types": "vue-tsc -p tsconfig.build.json --declaration --emitDeclarationOnly"
```

```jsonc
// tsconfig.build.json：继承主配置，只改产出相关
{
  "extends": "./tsconfig.app.json",
  "compilerOptions": {
    "noEmit": false,
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "dist/types"
  }
}
```

实测产出的 `.vue.d.ts` 长这样，可以看出组件的 props 类型是怎么被"翻译"成声明文件的：

```ts
// dist/types/MyButton.vue.d.ts
type __VLS_Props = {
    title: string;
};
declare const __VLS_export: import("vue").DefineComponent<
  __VLS_Props, {}, {}, {}, {}, import("vue").ComponentOptionsMixin,
  import("vue").ComponentOptionsMixin, {}, string, import("vue").PublicProps,
  Readonly<__VLS_Props> & Readonly<{}>, {}, {}, {}, {}, string,
  import("vue").ComponentProvideOptions, false, {}, any
>;
export default __VLS_export;
```

**注意一个前提：类型检查报错会直接阻断 d.ts 产出**（实测 4 个模板错误时 `dist/types` 是空的）。所以 `type-check` 和 `build:types` 天然是一条线，顺序别搞反。

另一个高频场景：**写包装组件时怎么复用另一个组件的 props 类型？**

```ts
// ❌ 不行：$props 里混着 Vue 内部属性（onUpdate:*、class、style、VNodeProps...）
type Props = InstanceType<typeof BaseButton>['$props'];

// ✅ 用 vue-component-type-helpers 提供的工具类型
import type { ComponentProps, ComponentEmit, ComponentSlots } from 'vue-component-type-helpers';
import BaseButton from './BaseButton.vue';

type BaseProps = ComponentProps<typeof BaseButton>;
interface Props extends BaseProps {
  size: 'sm' | 'md' | 'lg';
}
defineProps<Props>();
```

`vue-component-type-helpers` 是 Vue 语言工具链官方出的包，专门解决"从 `.vue` 组件反推类型"。**Vue 自带的 `ExtractPropTypes` 是给运行时 props 对象用的（`props: { foo: String }`），对 `.vue` 组件不管用**——这个区别面试里很能加分。

### 6. 类型声明宏的运行时边界：编译期安全 ≠ 运行时校验

这一节是我实测下来最值得记住的：**`defineProps<Props>()` 的运行时校验强度，完全取决于编译器能不能从类型里静态推导出构造函数。**

```ts
interface Props {
  title: string;
  union: string | number;
  obj: { a: number };
  fn: (x: number) => void;
  arr: string[];
}
const props = defineProps<Props>();
```

编译产物（实测，保留运行时声明部分）：

```text
// 编译产物摘录（不是完整语句）
props: {
  title: { type: String, required: true },   // ✅ 精准
  union: { type: [String, Number], required: true },  // ✅ 还行
  obj:   { type: Object, required: true },   // ⚠️ 内部结构丢了
  fn:    { type: Function, required: true },
  arr:   { type: Array, required: true }
}
```

再换一种写法——**从外部导入的、编译器看不到结构的类型**：

```ts
import type { Foo } from './types';
interface Props { foo: Foo; bar: string }
const props = defineProps<Props>();
```

```text
// 编译产物摘录
props: {
  foo: { type: null, required: true },   // ❗ 等价于"不校验"
  bar: { type: String, required: true }
}
```

`type: null` 意味着**运行时对这个 prop 不做任何类型校验**，编译期的类型检查在运行时是"裸奔"的。这也回答了一个经典追问：*为什么我的 props 传错了运行时没报错？* —— 因为它本来就不一定校验。

工程上的取舍建议：

- **内部组件**：`defineProps<Props>()` 够用，编译期拦住就够了；
- **对外发布的组件 / 边界数据**：关键 prop 补运行时声明（`validator` 也顺手写上），或者**用一套 Zod / schema 在入口层统一校验**（运行时数据校验是另一条线，别指望类型系统兜）；
- 类型声明和运行时声明**不能在同一个组件里混用**（同时写直接编译报错），只能二选一。

### 7. CI 门禁怎么接

推荐三条脚本分开，职责单一、报错定位清楚：

```jsonc
{
  "scripts": {
    "dev": "vite",
    "type-check": "vue-tsc --noEmit -p tsconfig.app.json",   // 类型门禁
    "build": "vite build",                                   // 只负责产物
    "build:types": "vue-tsc -p tsconfig.build.json --declaration --emitDeclarationOnly"
  }
}
```

CI 里的顺序：**`npm ci` → `type-check` → `build`**。理由：类型检查比构建便宜，错得早、停得快，没必要等打完包才发现类型炸了。

两条实践细节：

- 用 `create-vue` 那种 project references 结构时，可以直接 `vue-tsc --build`（按引用图增量检查），局部重跑更快；
- 别把类型检查塞进构建管线（比如 webpack 时代的 `ts-loader`）。官方专门解释过：类型检查需要**整个模块图**的信息，塞在单个文件的 transform 阶段既查不准（只能看转译后的代码、行号对不上源码）又拖慢构建。**类型检查和转译必须是两件事。**

## 其实你每天都在用

- `npm create vue@latest` 生成的 `tsconfig.json` / `tsconfig.app.json` / `tsconfig.node.json` 三件套，就是"不同运行环境用不同全局类型"的落地。
- `env.d.ts` 里的 `/// <reference types="vite/client" />`，让你能 `import.meta.env.BASE_URL` 而不报错。
- `<script setup lang="ts">` 里那个 `lang="ts"` 不是给 Vite 看的，是给 **Vue 编译器**看的——编译器据此决定要不要做类型静态分析。
- IDE 里给子组件传 prop 立刻标红，那是 Vue - Official 在后台跑的语言服务，跟 CI 跑的是同一套逻辑。
- 组件库里把 `Props` 抽到 `src/types/components.ts` 复用，一个接口被 5 个组件共享，改字段时编译器帮你把所有调用点找出来。
- 改事件名（比如 `change` 改成 `update`）时，`vue-tsc` 会列出所有监听点；不开 `strictTemplates` 的话它一个都不报，你只能靠搜字符串。
- `defineModel<string>()` 在类型层面要写泛型，运行时才有 `modelValue` / `update:modelValue` 这对约定。
- 提交前的 husky + lint-staged 跑一遍 `vue-tsc`，这就是把门禁左移到了本地。

## 常见误解（FAQ）

**❌ 误区1："项目用了 Vite + TS，类型错了构建就会失败"**

不会。Vite 是 transpile-only，esbuild 单文件擦类型，不检查。实测：同一个有 4 个模板错误的项目，`vite build` / 原生 `tsc --noEmit` 零报错，只有 `vue-tsc --noEmit` 报出来。**构建通过说明"语法能转译成 JS"，不说明"类型对"。**

**❌ 误区2："IDE 装了 Vue - Official 就不需要 vue-tsc 了"**

两者是同一套能力的两个出口，但**职责不同**：IDE 是给写代码的人看的即时反馈，挡不住别人把有类型错误的代码合进主干；`vue-tsc` 是可放进流水线的、能返回非零退出码的门禁。面试别说"有 Volar 就够了"。

**❌ 误区3："`strict: true` 开了，模板也会严格检查"**

两个开关管两件事。`strict` 是 **TypeScript 编译器**的严格模式，管代码层；`strictTemplates` 是 **Vue 编译器选项**，管模板层（且默认 `false`）。只开 `strict` 的话，模板里用了没注册的组件、传了没声明的 prop——全都不报错。这是实测过的。

**❌ 误区4："vue-tsc 就是 tsc 加个后缀，配置完全一样"**

配置基本一样（它读同一份 tsconfig），但多了两处差异：① 多了 `vueCompilerOptions` 这个专属配置块，必须写进**包含 Vue 源码的那份 tsconfig**（通常是 `tsconfig.app.json`）；② 类型检查时要 `--noEmit`，否则它会像 tsc 一样试图产出文件。

**❌ 误区5："`defineProps<Props>()` 生成的运行时校验和手写 `props: {}` 一样强"**

不一样，差得还挺关键。实测：`title: string` → `{ type: String }`；但**导入的复杂类型**（编译器看不到结构）会变成 `{ type: null }`，运行时完全不校验；`obj: { a: number }` 只生成 `type: Object`，内部结构丢失。**编译期安全 ≠ 运行时校验**，对外组件别指望它兜底。

**❌ 误区6："Vetur 和 Volar 装哪个都行，Vetur 更老更稳"**

**Vue 3 项目必须用 Vue - Official（Volar），并且禁用 Vetur。** Vetur 是 Vue 2 时代的扩展，不理解 `<script setup>`，会给合法代码报假错。装错扩展的典型症状就是"IDE 到处飘红但 `vue-tsc` 是干净的"。

**❌ 误区7："包装组件的 props 类型用 `InstanceType<typeof X>['$props']` 抽"**

这个类型里混着 Vue 内部属性——`onUpdate:*` 事件、`class`、`style`、`VNodeProps`、`AllowedComponentProps`（第 2 节实测报错信息里那一串就是它），抽出来的接口又脏又容易冲突。正确做法是用 `vue-component-type-helpers` 的 `ComponentProps<typeof X>`，官方 Vue 语言工具链出品。顺带记一下：**Vue 内置的 `ExtractPropTypes` 是给运行时 props 对象用的，对 `.vue` 组件无效。**

**❌ 误区8："类型检查塞进构建管线更省事，一次跑完"**

反而更糟。官方解释过：类型检查需要整个模块图的信息，塞进单文件 transform 阶段会导致 ① 只能检查转译后的代码，报错行号跟源码对不上；② 和转译抢同一个线程/进程，显著拖慢构建。**转译（esbuild/vite）和类型检查（vue-tsc）必须拆成两条独立链路。**

## 一句话总结

**Vite 负责"跑得快"，`vue-tsc` 负责"错不了"；构建通过不是类型安全的证明，CI 里那道 `vue-tsc --noEmit` 才是——而且别忘了 `strictTemplates` 默认是关着的。**
