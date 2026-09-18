---
layout: post
title: "provide / inject 原理：一条原型链搞定的依赖注入"
date: 2026-12-08 00:00:00 +0800
categories: ["前端核心", "Vue"]
tags: [provide, inject, 依赖注入, InjectionKey, 原型链]
description: >
  面试向拆解 Vue 依赖注入：为什么它的查找不是"遍历组件树"而是原型链查找、
  "组件自己 provide 的值自己 inject 不到"这条反直觉规则从哪来、inject 为什么不自动解包 ref、
  四类 dev 警告原文分别对应哪种错误写法，以及 InjectionKey / hasInjectionContext / runWithContext
  在组件库与组合式函数里的正确姿势。
---

## 一句话概括

provide / inject 就是 Vue 版的"跨层级传值"：祖先把值挂在某个对象上，后代顺着一条查找链往上取，中间的组件完全不用参与。

面试为什么爱问它？因为它看着像道 API 题，实际是道**数据结构题**。Vue 3 的做法是：每个组件实例挂一个 `provides` 对象，子组件的 `provides` 以父组件的 `provides` 为原型，于是"往上找"就变成了**原型链查找**——`in` 运算符天然会沿原型链走，不用写循环。而 Vue 2 是真的拿 `while` 沿着 `$parent` 一层层遍历组件树。

把这句话记住，后面所有"为什么"都能推出来：**provide 是往原型链上写属性，inject 是沿着原型链读属性。**

先给个反直觉的结论垫底：**组件自己 provide 的值，自己 inject 不到。** 原因在第 1 节。

## 核心知识点

### 1. 数据结构：一个 provides 对象 + 一条原型链

先看实例初始化那一行（runtime-core 的 `createComponentInstance`）：

```js
// 子组件默认直接复用父组件的 provides 对象，不是复制、也不是新建
provides: parent ? parent.provides : Object.create(appContext.provides)
```

再看 `provide()` 本尊：

```js
function provide(key, value) {
  if (currentInstance) {
    let provides = currentInstance.provides
    const parentProvides = currentInstance.parent && currentInstance.parent.provides
    // 本组件第一次 provide 时，才用父级 provides 作原型分裂出一层
    if (parentProvides === provides) {
      provides = currentInstance.provides = Object.create(parentProvides)
    }
    provides[key] = value  // 挂在自己这一层
  }
}
```

最后是 `inject()` 的查找起点：

```js
function inject(key, defaultValue, treatDefaultAsFactory = false) {
  const instance = getCurrentInstance()
  if (instance || currentApp) {
    const provides = currentApp
      ? currentApp._context.provides
      : instance.parent == null || instance.ce  // 根组件 / 自定义元素
        ? instance.vnode.appContext && instance.vnode.appContext.provides
        : instance.parent.provides              // ← 注意是 parent，不是自己
    if (provides && key in provides) return provides[key]   // in：会沿原型链查
    else if (arguments.length > 1) {
      return treatDefaultAsFactory && isFunction(defaultValue)
        ? defaultValue.call(instance && instance.proxy)
        : defaultValue
    }
  }
}
```

**面试就答这三点：**

1. **没 provide 就不复制对象**。父子实例的 `provides` 是同一个引用（实测 `子实例.provides === 父实例.provides` 为 `true`）。所以一万个组件也只有一条链，零额外内存开销。
2. **第一次 provide 才 `Object.create(parentProvides)`**。新对象自己只装本组件写的 key，其余全走原型链继承——这就是"往上找"的全部秘密。
3. **inject 的起点是 `instance.parent.provides`**，所以自己的 provide 对自己不可见；根组件 `parent == null`，走 `vnode.appContext.provides`，同理也拿不到自己的。

第 3 条实测验证过：父组件不 provide 任何东西时，同一个组件里先 `provide('self', 'mine')` 再 `inject('self', 'NOT_FOUND')`，拿到的是 `'NOT_FOUND'`；但它的子组件 `inject('self')` 拿到 `'mine'`。

这个设计其实很合理：要用自己的值直接读变量就行，绕 inject 没意义。而查找从父级开始，也就避免了"自己 provide 的值把自己祖先的同名值覆盖掉、然后自己 inject 到自己"的歧义。

### 2. 就近覆盖与兄弟隔离

因为是一条链，所以规则非常简单：

- **就近覆盖**：任意一层 provide 了同名的 key，它**整棵子树**都读这一层的值，祖先的值被"挡住"。
- **兄弟隔离**：两个兄弟组件各自 provide 同名的 key，互不影响——它们的链在父节点处就分叉了。

实测：`ROOT` provide `'ROOT'`，它下面的中间层 provide `'MID'`，中间层的叶子和另一个分支读到的都是 `'MID'`。

面试常追问的"怎么在中间层覆盖主题"，答案就是这个，不用写任何额外代码。

### 3. 响应性：它只传引用，不制造响应性

这是最高频的答错点。记住一句话：**provide / inject 本身跟响应式没有任何关系，它只是把值原样递过去。**

```js
// ❌ 普通对象：递的是同一个引用，但它没有响应式，后代不会被通知重渲染
const state = { count: 0 }
provide('state', state)
state.count++            // 后代不重渲染（实测渲染次数仍是 1）

// ✅ ref / reactive：响应性来自它们自己，跟 provide 无关
const count = ref(0)
provide('count', count)  // 后代 inject 到的就是这个 ref 本身
count.value++            // 后代重渲染（实测渲染次数 1 → 2）
```

还有一个必答细节：**inject 不会自动解包 ref**。

```js
const count = inject('count')
console.log(count.__v_isRef) // true —— 拿到的是 RefImpl，不是数字
// 脚本里必须 count.value，模板里才能直接 { { count } }
```

官方文档专门写了这一点，而且这是**故意的**——正因为不解包，注入方才能通过这个 ref 对象跟供给方保持响应式连接。

配套的两个官方建议：

```js
// 1. 变更逻辑留在供给方，不要把 ref 直接丢出去让人随便改
provide('location', { location, updateLocation })   // 值 + 改值的方法

// 2. 想彻底禁止改动，用 readonly 包一层
provide('count', readonly(count))
```

注意 `readonly` 的行为容易答错：**它不是抛错，是"改不动 + dev 警告"**。

实测 `ro.value = 99` 不抛异常，也**静默失败**（读回来还是 1），只在 dev 下打一条警告：

```text
[Vue warn] Set operation on key "value" failed: target is readonly.
```

### 4. 四个 dev 警告原文（对应四类错误写法）

面试里"我用 provide 不生效"这类问题，基本都能靠警告定位。原文和病因一一对应：

| dev 警告原文 | 触发场景 | 正确写法 |
| --- | --- | --- |
| `provide() can only be used inside setup().` | 在 `onMounted` 或事件回调里调 `provide()` | 挪到 `setup` 顶层（同步执行） |
| `injection "x" not found.` | 没有任何祖先 provide 这个 key，且自己没给默认值（返回 `undefined`） | 给默认值，或检查 key 拼写与组件层级 |
| `inject() can only be used inside setup() or functional components.` | 完全脱离组件上下文调用，比如 `setTimeout` 里、普通工具函数里 | 用 `hasInjectionContext()` 兜底 |
| `App already provides property with key "x". It will be overwritten with the new value.` | 同一个 key 调了两次 `app.provide` | 换 key 或合并值 |

**「只能同步调用」这条要理解准**，别记成"生命周期里拿不到"。实测：**祖先 provide 的值，在子组件的 `onMounted` 里 `inject` 是能拿到的**（Vue 调用生命周期钩子时会把当前实例重新挂上）。真正失效的是"完全脱离组件上下文"的场景——那时 `getCurrentInstance()` 和 `currentApp` 都是空，直接走最后一条警告。

顺带把"找不到"的行为钉死：**只警告、不抛错，返回 `undefined`**。所以线上环境（无 dev 警告）里写错 key 是彻底静默的——这也是大型项目推荐用 Symbol 当 key 的原因之一。

### 5. 默认值的两种写法：值 vs 工厂函数

```js
// 普通默认值
const theme = inject('theme', 'light')

// 工厂函数：第三个参数 true 表示"把它当工厂调用"
const store = inject('store', () => createExpensiveStore(), true)
```

工厂存在的意义：默认值终究是个**实参**，`inject('k', new ExpensiveThing())` 里那个 `new` 不管你用不用都会执行。包成函数 + 传 `true` 才是"按需创建"。

实测两个细节：工厂**只在真的需要时调用一次**；调用时 `this` 被绑定到**组件代理**（`defaultValue.call(instance.proxy)`，所以工厂里能访问 `this.$options.name` 这类东西）。

### 6. 类型化：InjectionKey + Symbol

字符串 key 的问题是**没有唯一性也没有类型**——两个库都用 `'config'`，谁覆盖谁全看层级。Symbol 天然唯一，再配上 `InjectionKey<T>` 泛型就能把类型串起来：

```ts
// keys.ts
import type { InjectionKey, Ref } from 'vue'
export const LocationKey: InjectionKey<Ref<string>> = Symbol('location')
```

```ts
// 供给方：value 类型不对会报错
provide(LocationKey, ref('North Pole'))

// 注入方：自动推导为 Ref<string>
const location = inject(LocationKey)   // 类型：Ref<string> | undefined
```

注意最后那个 `| undefined`：**即使 `InjectionKey` 带了泛型，编译器也不知道运行时到底有没有人 provide**。实测给了默认值之后类型就不含 `undefined` 了（可以直接赋给 `Ref<string>`），没给默认值就只能 `!` 断言——团队规范里最好把 `!` 限定在"确定在 `app.provide` 里提供过"的场景。

另外 `provide` 这一侧也是类型安全的：`provide(LocationKey, ref(123))` 会直接报错，写错类型编译期就拦住了。

### 7. 组合式函数的正确姿势：hasInjectionContext / runWithContext

自己写的 composable 直接调 `inject` 有个老问题：它在 `setup` 之外被调用时（比如在组件外初始化 store、单元测试里）会整个报警告。Vue 3.3 起给了两个配套 API：

```js
// 3.3+：先探上下文，没有就优雅降级
export function useTheme() {
  if (!hasInjectionContext()) return defaultTheme
  return inject(ThemeKey, defaultTheme)
}
```

```js
// 给插件、工具函数一个能调 inject 的入口
const app = createApp(App)
app.provide('api', apiClient)
app.runWithContext(() => {
  // 这里 inject('api') 能正常拿到
})
```

`runWithContext` 的实现很朴素：把 `currentApp` 临时指过去，执行完再换回来。实测在它里面 `inject('api')` 能读到 `app.provide` 的值，`hasInjectionContext()` 返回 `true`。

`<script setup>` 里不需要这些——编译器能识别出组件上下文，直接用就行。

## 其实你每天都在用

- `<el-form>` 和 `<el-form-item>`：form 往下 provide 校验规则和 model，form-item 一 inject 就拿到，中间你写的那些 div 和布局组件完全不用管。
- `app.provide('i18n', ...)`：语言包在应用层提供一次，全站任何组件 `inject` 一下就能用，这是 i18n 插件的标准做法。
- Vue Router 的 `useRoute()`、Pinia 的 `useStore()`：底层都是依赖注入——先在全站范围内注入一下核心实例，组件里取出来直接用。
- 主题切换：根组件 provide 一个 `theme` ref，深层某个图表组件 inject 后自己换配色，不用从顶部一层层传 props。
- UI 库的命令式 API：`provide(ModalKey, { open })` 让深层的小按钮能叫起顶层弹窗，避免弹窗组件挂得到处都是。
- 单元测试里替换依赖：`mount(Comp, { global: { provide: { ThemeKey: mockTheme } } })`，不改组件代码就能把真实依赖换成假的。
- 组件库的 `useFormDisabled()` 这类内部 composable：父级 `<el-form :disabled="true">` 一 provide，下面所有输入框自动变灰，靠的就是它。
- 自己写 `useTableContext()`：表格父组件 provide 一份分页状态，各个子列组件 inject 后自己取数据，省掉一堆 props 透传。

## 常见误解（FAQ）

**❌ 误区1："provide / inject 是响应式的，所以 provide 一个普通对象后代就会更新"**

不成立。它俩只是**递引用**，响应性完全来自递过去的那个值本身。实测 provide 一个普通对象后改它的属性，后代组件渲染次数仍然是 1（没更新）；provide 一个 `ref` 或 `reactive` 才会跟着更新。所以规范是：**要响应就 provide 响应式对象，要安全就再套 `readonly`。**

**❌ 误区2："inject 拿到的 ref 会自动解包"**

不自动。实测 `inject('count').__v_isRef` 是 `true`，脚本里必须 `.value`。这是**有意设计**：正因为不解包，注入方才能通过这个 ref 跟供给方保持联系。会自动解包的场景是"模板顶层属性"和"`reactive` 对象的属性"，跟 inject 无关。

**❌ 误区3："inject 找不到 key 会抛错，所以线上不会踩坑"**

恰好相反：**只打 dev 警告、返回 `undefined`，绝不抛错**。所以线上写错 key 或层级不对，表现是"值莫名是 undefined"，而不是报错。三条防线：① 用 `InjectionKey` 配 Symbol，让 key 在编译期就不可能写错；② 拿不准有没有人 provide 时给个默认值；③ 在自己写的 composable 里先用 `hasInjectionContext()` 兜一层，避免脱离上下文时刷警告。

**❌ 误区4："组件可以 inject 自己 provide 的值"**

不行，这是本文最反直觉的一条。`inject` 的查找起点是 `instance.parent.provides`，**自己的 provide 对自己永远不可见**，根组件也一样。实测同一组件里 `provide('self','mine')` 之后 `inject('self','FALLBACK')` 拿到 `'FALLBACK'`。自己的值直接读变量就好。

**❌ 误区5："provide 必须在 setup 里同步调用，所以 onMounted 里 inject 也拿不到"**

把两个 API 搞混了。`provide` 确实只能同步（在 `onMounted` 里调会警告 `provide() can only be used inside setup().`），但 **`inject` 在 `onMounted` 里是能拿到祖先值的**——实测拿到了。真正会让 `inject` 失效的是完全脱离组件上下文（比如 `setTimeout` 回调），那时的警告是 `inject() can only be used inside setup() or functional components.`

**❌ 误区6："provide / inject 能当轻量版状态管理，就不用 Pinia 了"**

小范围（主题、表单上下文、组件库内部）确实是它的主场，但**不要用它替代全局状态管理**。三个硬伤：① 它是单向的，注入方只知道"有个值"，不知道"谁会改它"，改动责任链完全不可追踪；② 没有 devtools 时间旅行，出问题只能一层层看组件树；③ 一旦某个组件决定自己再 provide 一层同名 key，它整棵子树就"脱轨"了，排查成本很高。

**❌ 误区7："`readonly` 包一层，注入方改动会直接报错，能立刻发现问题"**

不会报错，只警告，而且是**静默失败**——实测写入后值没变、代码继续往下走，dev 下才有一条 `Set operation on key "value" failed: target is readonly.`。所以别把 `readonly` 当校验手段，它只是防手抖；要强约束还是靠 TypeScript 类型 + 代码规范。

**❌ 误区8："provide 的 key 用字符串就行，反正组件内部自己和自己约定"**

自己写业务组件问题不大，但**组件库、插件、以及任何会被别人 `app.use` 的东西必须用 Symbol**。原因是字符串 key 没有命名空间，两个库都用 `'config'`，谁先谁后、谁在谁里面会直接决定读到谁的值，而且没有任何编译期或运行期提示。官方文档明确建议这种场景用 Symbol，并统一放在 `keys.ts` 里导出。

## 一句话总结

provide 是往一条用 `Object.create` 串起来的原型链上挂属性，inject 是从**父级**开始用 `in` 沿链读——不响应是因为递的是普通值、不用 `.value` 是因为 ref 不解包、找不到只警告不报错、自己 provide 的自己永远 inject 不到。
