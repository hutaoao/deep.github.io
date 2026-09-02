---
layout: post
title: "框架中的类型设计：props / events / 插槽推导"
date: 2026-11-14 00:00:00 +0800
categories: ["前端核心", "TypeScript"]
tags: [TypeScript, defineProps, defineEmits, defineSlots, ComponentPropsWithoutRef, 泛型组件]
description: >
  面试向讲清现代前端框架的类型设计：Vue 的 defineProps / defineEmits / defineSlots / defineModel / generic 如何由编译器静态分析推导，
  React 的泛型组件、多态组件 as、ComponentPropsWithoutRef 与 React 19 ref as prop 的类型变化，以及两边共有的类型失效边界。
---

## 一句话概括

组件本质上就是一个函数，**props 是入参、events 是回调出参、插槽是 render prop**。所谓"框架中的类型设计"，就是想办法把这三样东西的类型**自动推导出来**，让调用方在写 `<MyTable :columns="x" />` 的时候有补全、写错了立刻报错，而不是运行时才发现字段拼错。

两条技术路线不一样，面试各说各的：

- **Vue 走"编译器"路线**：`defineProps` / `defineEmits` / `defineSlots` 这些叫**编译器宏**，它们在构建时被 `@vitejs/plugin-vue` 静态分析掉，编译成运行时的 `props` / `emits` 选项。**类型是从你写的 TS 类型"翻译"成运行时声明的**。
- **React 走"纯类型"路线**：没有编译步骤参与，全是 TS 类型运算——泛型、`ComponentPropsWithoutRef`、`Omit` 这些拼出来的。**类型只存在于编译期，运行时什么都没有**。

记住这个对比，面试官问"Vue 和 React 的类型方案有什么本质区别"时，这一句就能撑起开头。

## 核心知识点

### 1. 组件即函数：三件套的类型契约

先建立一个统一心智模型，后面两节都在填这个表：

| 契约 | Vue | React |
|---|---|---|
| 入参（props） | `defineProps<Props>()` | `Props` 泛型 / 函数参数类型 |
| 出参（事件） | `defineEmits<Emits>()` | props 里的 `onXxx` 回调函数 |
| 内容（插槽） | `defineSlots<Slots>()` | `children` / `renderItem` 之类的 prop |
| 暴露给父级 | `defineExpose()` | `ref` + `useImperativeHandle` |

React 没有"事件"和"插槽"这两个概念——**事件就是回调 prop，插槽就是 `children` 或 render prop**。这是两边类型设计差异的根源：Vue 有专门的宏，React 全靠类型组合。

### 2. Vue：defineProps 的两种声明，别混用

`defineProps` 支持两种写法，**同一个组件里只能选一种，同时写会直接编译报错**。

```ts
// 运行时声明：传一个选项对象，跟 Options API 的 props 一模一样
const props = defineProps({
  foo: { type: String, required: true },
  bar: Number,
});
// props.foo: string，props.bar: number | undefined

// 类型声明：传一个泛型参数，更简洁也更常用
const props = defineProps<{
  foo: string;
  bar?: number;
}>();
```

看起来第二种只是"写法更短"，实际上背后有个关键机制，面试常问：

**类型声明会被编译器静态分析，反向生成运行时声明。** 因为运行时还得做 props 校验和响应式处理，光有 TS 类型是不够的。具体来说：

- **开发模式**：编译器尽力从类型推导运行时校验。比如 `foo: string` 推导出 `foo: String`。
- **生产模式**：为了减小包体积，直接编译成数组形式 `['foo', 'bar']`，不做校验。

所以类型声明不是"零成本"的，它依赖编译器能看懂你的类型。这就引出了最经典的坑（下一节）。

老项目里还会见到 `PropType`——那是运行时声明下给复杂类型标注用的，比如 `type: Object as PropType<Book>`。**用了类型声明就不需要 `PropType` 了**，别混着写。

### 3. Vue 静态分析的边界：什么是编译器看不懂的

这是 Vue 类型题里**最高频的追问**，一定要答得出来。

Vue 3.2 及以前，`defineProps` 的泛型只能是**字面量类型**或**同文件内的本地 interface**。3.3 起放开了，支持导入的类型和一部分复杂类型。但核心限制没变：

> **类型转换是基于 AST 的静态分析，不是完整的类型计算。**

翻译成人话：编译器是"看代码文本"猜出来的，不是真的跑了一遍 TS 类型系统。所以：

```ts
import type { Props } from './types';      // ✅ 3.3+ 支持导入类型
import type { User } from '@/api';

// ❌ 整个 props 对象用条件类型 —— 编译器算不出来
const props = defineProps<Admin extends true ? AdminProps : UserProps>();

// ✅ 单个 prop 的类型用条件类型 —— 没问题
interface Props {
  value: IsAdmin extends true ? string : number;
}
const props = defineProps<Props>();
```

记住这条判断标准：**单个 prop 的类型可以随便复杂，整个 props 对象的顶层类型必须是编译器能"走"进去的**（字面量、interface、类型别名、`&` 交叉、导入的类型都行）。

另外导入的类型有个隐含前提：**TypeScript 必须是 Vue 的 peer dependency**（正常项目都满足）。生产模式下导入类型推导不出具体校验，会退化成 `foo: null`（等于不校验）。

### 4. Vue 3.5 响应式 Props 解构：默认值写法变了

类型声明有个历史痛点：**没法写默认值**。以前的解决方案是 `withDefaults` 宏：

```ts
interface Props { msg?: string; labels?: string[] }

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello',
  labels: () => ['one', 'two'], // ⚠️ 数组/对象必须包成工厂函数
});
```

注意那个 `() => [...]`，**引用类型必须包工厂函数**，否则所有组件实例共享同一个数组引用——这跟 Vue 2 的 `data` 必须返回函数是同一个坑。

**Vue 3.5 起，Props 解构是响应式的了**，可以直接写原生 JS 默认值：

```ts
const { msg = 'hello', labels = ['one', 'two'] } = defineProps<Props>();
// 3.5+ 这里不用包工厂函数，每次求值都是新数组
```

背后的原理一句话：编译器会在同一个 `<script setup>` 块里，把你访问的解构变量自动加回 `props.` 前缀。

```ts
const { foo } = defineProps<{ foo: string }>();
watchEffect(() => console.log(foo));
// 编译后 ≈ watchEffect(() => console.log(props.foo))
```

**在 3.4 及以下这么写会丢失响应式**（`foo` 就是个普通常量），这是版本差异题的必考点。

但解构后有个新坑：**解构出来的变量是"值"不是"响应式数据源"**，直接丢给 `watch` 不行：

```ts
const { foo } = defineProps<{ foo: string }>();

// ❌ 相当于 watch(props.foo, ...)，传的是值不是数据源，编译器会警告
watch(foo, (v) => console.log(v));

// ✅ 包一层 getter
watch(() => foo, (v) => console.log(v));

// ✅ 传给组合式函数时也用 getter，函数内部用 toValue() 归一化
useSomething(() => foo);
```

### 5. Vue：defineEmits、defineModel、defineSlots

**defineEmits** 有两种类型写法，3.3 之后的元组写法更清爽：

```ts
// 老写法：调用签名（call signature）
const emit = defineEmits<{
  (e: 'change', id: number): void;
  (e: 'update', value: string): void;
}>();

// 3.3+ 新写法：事件名 -> 参数元组，还能给元组元素加标签
const emit = defineEmits<{
  change: [id: number];
  update: [value: string];
}>();

emit('change', 1);      // ✅
emit('change', 'abc');  // ❌ 类型报错
emit('unknown');        // ❌ 没声明的事件名直接报错
```

两种编译结果一样，新项目用第二种。注意**没声明的事件名会直接编译报错**，这是类型声明比运行时声明强的地方。

**defineModel（3.4+ 稳定）** 把 `v-model` 的 prop + event 双声明压成一行：

```ts
const modelValue = defineModel<string>();
//    ^? Ref<string | undefined>

const required = defineModel<string>({ required: true });
//    ^? Ref<string>，required 会去掉 undefined

const [value, modifiers] = defineModel<string, 'trim' | 'uppercase'>();
//    modifiers ^? Record<'trim' | 'uppercase', true | undefined>
```

有个细节值得记：**`defineModel` 背后会悄悄加一个 `modelModifiers` prop**（具名 model 是 `xxxModifiers`），不管你用不用修饰符。调试时发现组件上多了个没声明的 prop，就是它。

**defineSlots（3.3+）** 只接受类型参数、不接受运行时参数，这是唯一能给插槽上类型的方式：

```ts
const slots = defineSlots<{
  // key 是插槽名，value 是插槽函数；第一个参数是作用域插槽的 props
  default(props: { item: string; index: number }): any;
  header(props: { title: string }): any;
}>();
```

插槽函数的返回值类型目前会被忽略（写 `any` 就行），未来可能用于内容检查。它返回的 `slots` 对象等价于 `useSlots()` 拿到的那个。

### 6. Vue：泛型组件与 defineExpose

`<script setup>` 上可以加 `generic` 属性声明泛型，用法跟 TS 的 `<...>` 完全一致：

```vue
<script setup lang="ts" generic="T extends { id: string }">
defineProps<{ items: T[]; selected?: T }>();
</script>
```

父组件一般能自动推导，推导不出来时用 `@vue-generic` 指令显式指定：

```vue
<template>
  <!-- @vue-generic {import('@/api').Actor} -->
  <ApiSelect v-model="ids" endpoint="/api/actors" />
</template>
```

这里有个反直觉的坑：**泛型组件不能用 `InstanceType` 取实例类型**（`InstanceType` 对泛型组件会失效）。要用官方推荐的 `vue-component-type-helpers` 库：

```ts
import { ref } from 'vue';
import type { ComponentExposed } from 'vue-component-type-helpers';
import GenericComp from './generic-component.vue';

// ❌ 泛型组件用 InstanceType 拿不到正确类型
// const r = ref<InstanceType<typeof GenericComp>>();

// ✅ 泛型组件用 vue-component-type-helpers 的 ComponentExposed
const r = ref<ComponentExposed<typeof GenericComp>>();
```

**defineExpose** 的存在是因为 `<script setup>` 组件**默认是封闭的**——父组件通过模板 ref 拿到的实例上，什么都访问不到。想暴露就要显式声明：

```ts
const count = ref(0);
const reset = () => { count.value = 0 };

// ✅ 只暴露方法，别把状态直接扔出去
defineExpose({ reset });
```

**最佳实践是暴露方法而不是暴露 ref 状态**。父组件拿到 `count` 这个 ref 就能直接改，等于把内部状态开放成公共 API，后面重构会寸步难行。

### 7. React：泛型组件与 props 反推

React 这边全是纯类型运算，没有编译器参与。

**泛型组件**要用 `function` 声明式写法，不能用箭头函数——因为 `.tsx` 里 `<T>` 会被当成 JSX 标签解析：

```tsx
// ✅ 函数声明式，TS 能正确识别泛型
function List<T>({ items, renderItem }: {
  items: readonly T[];
  renderItem: (item: T) => React.ReactNode;
}) {
  return <ul>{items.map((i, k) => <li key={k}>{renderItem(i)}</li>)}</ul>;
}

// ❌ 箭头函数 + 泛型在 .tsx 里会被解析成 JSX，要么报错要么行为诡异
const List2 = <T,>(props: P) => ...; // 加了逗号能过，但不推荐
```

调用方不用写泛型，TS 从 `items` 的类型自动推导出 `T`，`renderItem` 的参数类型就跟着确定了——这是泛型组件最爽的地方。

**从组件反推 props 类型**，用 `ComponentProps` 系列：

```ts
type ButtonProps = React.ComponentProps<typeof Button>;          // 含 ref（React 19 类型）
type Props2 = React.ComponentPropsWithRef<'button'>;             // 转发 ref 时用
type Props3 = React.ComponentPropsWithoutRef<'button'>;          // 不支持 ref 时用
```

三者的区别就在 ref。**`ComponentProps` 的官方注释就明说了：转发 ref 用 `WithRef`，不支持 ref 用 `WithoutRef`。** 别再无脑用 `ComponentProps` 了。

### 8. React：多态组件（as prop）

多态组件是设计系统的招牌模式：一个 `Button`，传 `as="a"` 就渲染成链接。难点是**props 要跟着 `as` 变**——`as="a"` 时才允许 `href`。

```tsx
import type { ElementType, ComponentPropsWithoutRef } from 'react';

type BoxOwnProps = { variant?: 'primary' | 'secondary' };

type BoxProps<E extends ElementType = 'div'> =
  BoxOwnProps &
  { as?: E } &
  Omit<ComponentPropsWithoutRef<E>, keyof BoxOwnProps | 'as'>; // 去掉会冲突的同名 props

function Box<E extends ElementType = 'div'>({ as, variant, ...rest }: BoxProps<E>) {
  const Tag = (as ?? 'div') as ElementType;
  return <Tag {...rest} />;
}

// ✅ href 只在 as="a" 时可用
<Box as="a" href="/home">Home</Box>
// ❌ 没写 as="a" 就传 href → 类型报错
<Box href="/home">Home</Box>
```

三个要点：

1. **`Omit` 去掉自己已声明的 props**，否则 `children` / `className` 之类会跟原生 props 冲突，报出一堆看不懂的错。
2. **给泛型设默认值 `= 'div'`**，否则不传 `as` 时 TS 推导不出默认标签，会漏掉类型检查。
3. 内部的 `as ElementType` 强转是 React JSX 类型系统的已知限制，属于"必要之恶"。

这条也常作为"你怎么做设计系统"的追问，能讲到 `Omit` 和泛型默认值就够用了。

### 9. React 19 的类型变化（2026 年面试必问版本差异）

React 19 动了大手术，几个跟类型直接相关的改动：

```tsx
// React 18：必须 forwardRef
const Input = forwardRef<HTMLInputElement, Props>((props, ref) => <input ref={ref} {...props} />);

// React 19：ref 就是普通 prop，forwardRef 已废弃（仍能跑，但有 dev 警告）
function Input({ ref, ...props }: Props & { ref?: React.Ref<HTMLInputElement> }) {
  return <input ref={ref} {...props} />;
}
```

配套的类型变化：

- **`React.ComponentProps` 现在自动包含 `ref`**（React 18 里是排除的）。这是 `WithRef` / `WithoutRef` 区分意义的来源。
- **`MutableRefObject` 已废弃**，`RefObject<T>` 的语义变了，声明"T 或 null"的写法要跟着改。
- **`ReactElement` 的 props 类型从 `any` 改成了 `unknown`**。以前 `cloneElement` 里瞎读 props 是静默通过的，现在会报错。
- **ref 回调可以返回清理函数**（跟 `useEffect` 一样）。副作用是：**如果你的 ref 回调返回了非函数值，React 19 会警告**。
- **全局 `JSX` 命名空间没了**，得写 `React.JSX.Element`。升级后满屏 `Cannot find namespace 'JSX'` 就是这个原因——库代码里尤其常见。

一个实际的坑：`{...props}` 展开现在会把 `ref` 也带上，可能意外转发。想精确控制就先解构出来：`const { ref, ...rest } = props`。

## 其实你每天都在用

- Element Plus / Ant Design Vue 的组件：鼠标悬停能看到完整的 props 表格和事件签名，靠的就是 `defineProps` / `defineEmits` 生成的类型。
- `<ElTable>` 的作用域插槽里 `{ row }` 有类型补全——`defineSlots` 声明的插槽 props。
- Modal 组件 `defineExpose({ open, close })`，父组件 `modalRef.value?.open()` 能自动补全出方法名。
- MUI / Chakra 的 `<Button component={RouterLink} to="/home">`——多态组件的真实例子。
- `React.ComponentProps<typeof Child>` 复用子组件 props，不用再手动 `export interface Props`。
- 封装 `useRequest<T>()` 时，泛型让 `data` 直接是 `T`，调用处写 `useRequest<User>()`。
- 表单组件的 `v-model` 用 `defineModel` 一行搞定，不用再 `computed({ get, set })`。
- 老代码里的 `type: Object as PropType<Book>`——运行时声明时代的产物，见到知道是什么就行。

## 常见误解（FAQ）

**❌ 误区1："`defineProps` 是运行时函数，可以从 vue 里 import。"**
不是。它是**编译器宏**，编译后完全消失，产物里没有 `defineProps` 这个函数调用。`defineEmits` / `defineSlots` / `defineModel` / `defineExpose` / `withDefaults` 同理。所以它们**只能在 `<script setup>` 顶层使用**，不能放进函数里、不能条件调用。

**❌ 误区2："用了类型声明就不需要运行时校验了，反正 TS 检查过了。"**
TS 只在编译期生效，运行时该校验还得校验——否则后端传错字段、或 JS 调用方传错，组件就静默跑歪。这就是为什么编译器要把类型**反向翻译**成 `props` 选项。生产模式下为了包体积会退化成数组形式（不做校验），这是官方的有意取舍。

**❌ 误区3："`defineProps` 的泛型里可以随便写条件类型和工具类型。"**
整个 props 对象的顶层不行，**必须是编译器 AST 能走通的结构**（字面量、interface、类型别名、导入类型、`&` 交叉）。单个 prop 的类型可以任意复杂。这条是 Vue 类型题里最常被追问的边界。

**❌ 误区4："Vue 3 里 `const { foo } = defineProps()` 会丢失响应式。"**
**3.5 起不会了**，编译器会自动加回 `props.` 前缀。但 3.4 及以下会丢。另外即使是 3.5+，解构出来的变量传给 `watch` 或组合式函数时仍要包 getter（`() => foo`），因为它是值不是响应式数据源。

**❌ 误区5："`withDefaults` 的默认值直接写对象/数组就行。"**
不行，**引用类型必须包工厂函数**（`labels: () => []`），否则所有实例共享同一个引用。改成 3.5 的解构默认值写法则不需要。

**❌ 误区6："React 里泛型组件用箭头函数写也行。"**
`.tsx` 里 `<T>` 会被解析成 JSX 标签。就算加 `<T,>` 绕过，可读性也差。**泛型组件一律用 `function` 声明式**。

**❌ 误区7："拿组件 props 类型就用 `React.ComponentProps`。"**
看情况：**转发 ref 用 `ComponentPropsWithRef`，不支持 ref 用 `ComponentPropsWithoutRef`**。React 19 类型里裸 `ComponentProps` 会包含 ref，容易跟自己声明的 props 冲突。

**❌ 误区8："React 19 里还得用 `forwardRef`。"**
不用了，**ref 就是普通 prop**，`forwardRef` 已标记废弃（现有代码还能跑，dev 下有警告）。顺带注意 `MutableRefObject` 废弃、`ReactElement` props 变成 `unknown`、`{...props}` 会把 ref 一起展开这三个连带变化。

**❌ 误区9："插槽类型能约束插槽内容返回什么。"**
不能。`defineSlots` 的插槽函数**返回值类型目前被忽略**（写 `any` 即可），它只约束插槽名和作用域插槽的 props。官方说未来可能用于内容检查，现在还没有。

**❌ 误区10："泛型组件用 `InstanceType<typeof Comp>` 就能拿到实例类型。"**
对泛型组件**不行**。官方推荐 `vue-component-type-helpers` 的 `ComponentExposed`。非泛型组件用 `InstanceType` 没问题。

## 一句话总结

**Vue 靠编译器宏把 TS 类型"翻译"成运行时声明（`defineProps` / `defineEmits` / `defineSlots` / `defineModel`，3.5 起解构即响应式、但顶层类型不能是条件类型）；React 靠纯类型运算（泛型 + `ComponentPropsWithRef` / `WithoutRef` + `Omit` 拼出多态组件，React 19 起 ref 已是普通 prop）——两条路的共同目标只有一个：让组件这份"函数签名"在调用侧有补全、有校验。**
