---
layout: post
title: "React 19 新特性：useOptimistic / useActionState / Server Actions"
date: 2026-11-27 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, React19, useActionState, useOptimistic, useFormStatus, ServerActions, 乐观更新]
description: >
  面试向讲清 React 19 的 Actions 体系：useActionState 返回的三元组各自是什么、
  dispatchAction 为什么必须包在 Transition 或 form action 里、useOptimistic 乐观值何时回滚、
  useFormStatus 为什么必须写在 form 的子组件里、以及 Server Actions 的序列化边界与安全红线。
---

## 一句话概括

React 19 这一组新 API 解决的是同一件老事：**提交一个表单 / 点一个按钮 → 发请求 → 期间要 loading → 回来要更新 UI → 失败要报错回滚**。以前这套流程你得手写 `useState` 三连（`loading` / `data` / `error`），现在 React 把它收编成了 Actions 体系。

三个主角分工很清晰，一句话记住：

- **`useActionState`**：管"这次提交的结果和状态"——替代 `useState` 三连，从 `react` 导入。
- **`useOptimistic`**：管"趁请求还没回来，先给用户看点东西"——乐观更新，失败自动回滚，从 `react` 导入。
- **`useFormStatus`**：管"深层子组件不用 props 也能知道表单在提交"——从 **`react-dom`** 导入（注意不是 `react`）。

再加上 **Server Actions**（`'use server'`），让客户端可以直接调用服务端函数，连 API 路由都不用写。

一句话：**React 19 把"异步变更"从手写状态机，变成了框架内置能力。**

## 核心知识点

### 1. useActionState：一个 Hook 顶三个 useState

签名（React 19 正式版）：

{% raw %}
```tsx
const [state, dispatchAction, isPending] = useActionState(reducerAction, initialState, permalink?);
```
{% endraw %}

| 返回值 | 是什么 |
| --- | --- |
| `state` | 首次渲染 = `initialState`；之后 = 上一次 `reducerAction` 的返回值 |
| `dispatchAction` | 触发函数，**身份稳定**（可安全放进 effect 依赖，也可省略） |
| `isPending` | 是否有该 Hook 派发的 action 还在进行中 |

`reducerAction` 的签名是 `(prevState, payload) => newState`，**可以同步也可以 async**——这就是它比 `useReducer` 强的地方：`useReducer` 的 reducer 必须纯且同步。

{% raw %}
```tsx
// ✅ 典型写法：表单提交 + 服务端校验错误回显
'use client';
import { useActionState } from 'react';

type State = { error: string | null };

async function submitAction(prev: State, formData: FormData): Promise<State> {
  const name = formData.get('name') as string;
  if (!name?.trim()) return { error: '名字不能为空' };   // 返回值直接成为新的 state
  await saveName(name);
  return { error: null };
}

export function NameForm() {
  const [state, formAction, isPending] = useActionState<State, FormData>(submitAction, { error: null });
  return (
    <form action={formAction}>
      <input name="name" />
      {state.error && <p className="err">{state.error}</p>}
      <button disabled={isPending}>{isPending ? '提交中…' : '提交'}</button>
    </form>
  );
}
```
{% endraw %}

注意这里**没有 `e.preventDefault()`**，也**没有 `new FormData(e.target)`**——React 拦下提交，直接把 `FormData` 作为第二个参数喂给你的 action。

### 2. dispatchAction 必须在 Action / Transition 里调

这是最容易踩的坑。不在 Transition 或 action prop 上下文里调用 `dispatchAction`，开发模式直接报错。

{% raw %}
```tsx
function Bad() {
  const [state, dispatch, isPending] = useActionState(action, 0);
  return <button onClick={() => dispatch()}>加购</button>;
  // ❌ 报错：dispatch 必须在 startTransition 里，或作为 action prop 传入
}

function Good() {
  const [state, dispatch, isPending] = useActionState(action, 0);
  return (
    <button onClick={() => startTransition(() => dispatch())}>加购</button>
    // ✅ 包一层 startTransition
  );
}

function Best() {
  const [state, formAction, isPending] = useActionState(action, 0);
  return <form action={formAction}><button>加购</button></form>;
  // ✅ 作为 action prop 传入时，React 自动包 Transition，最省事
}
```
{% endraw %}

其他几条面试加分点：

- **多次调用按顺序排队执行**，后一次拿到的是前一次的返回值，不会互相覆盖。
- **StrictMode 下 `reducerAction` 不会被调用两次**——因为它被设计为允许副作用（和 `useReducer` 的纯函数要求不同）。
- **`dispatchAction` 抛错**，React 会取消所有排队的 action，并把错误交给最近的 **Error Boundary**。
- **多个 action 同时进行时 React 会批处理**——这是当前版本的已知限制，未来可能改。
- 第三个参数 `permalink` 是给 RSC 渐进增强用的：JS 包还没加载完就提交表单时，浏览器会导航到这个 URL。

### 3. useOptimistic：先改 UI，失败自动回滚

{% raw %}
```tsx
const [optimisticState, setOptimistic] = useOptimistic(baseValue, reducer?);
```
{% endraw %}

- `baseValue`：真实值。没有 pending action 时，`optimisticState === baseValue`。
- `reducer?`：可选的纯函数 `(current, actionValue) => next`。不传的话，`optimisticState` 直接等于你 `setOptimistic(x)` 传的 `x`。
- `setOptimistic`：只在 Action / Transition 内部调用才生效。

**生命周期（必背）**：调用 `setOptimistic` → 立即渲染乐观值 → 请求回来更新真实值，React 在**同一次提交里合并**，不会再多一次"清除"渲染 → 如果 action 抛错，Transition 结束，React 渲染当前的 `baseValue`，**因为真实值没变，UI 自动回滚**。

{% raw %}
```tsx
'use client';
import { useOptimistic, startTransition } from 'react';

function LikeButton({ likes, postId }: { likes: number; postId: string }) {
  //  reducer 形式：相对增量，避免用绝对值覆盖掉期间别人点的赞
  const [optimisticLikes, addOptimistic] = useOptimistic(
    likes,
    (current: number, delta: number) => current + delta
  );

  function handleClick() {
    startTransition(async () => {
      addOptimistic(1);            // 立刻 +1
      try {
        await toggleLike(postId);  // 服务端确认
      } catch (e) {
        toast('点赞失败');          // 乐观值自动回滚，不用手写回滚逻辑
      }
    });
  }

  return <button onClick={handleClick}>❤️ {optimisticLikes}</button>;
}
```
{% endraw %}

两个必须记住的坑：

{% raw %}
```tsx
// ❌ 在 Action 外调用：乐观值闪一下就消失，并报警
//    "An optimistic state update occurred outside a Transition or Action"
setOptimistic(1);
<button onClick={() => setOptimistic(1)}>点</button>

// ❌ 在渲染阶段调用：直接报错
//    "Cannot update optimistic state while rendering"
function C() { setOptimistic(1); return null; }
```
{% endraw %}

另外，**基础值在 pending 期间变了，React 会拿新基础值重跑 reducer**——所以要用"相对增量"（`+1`）而不是"绝对值"（`= 5`），否则会把别人的更新覆盖掉。

### 4. useFormStatus：别写在使用它的 form 里

`useFormStatus` 来自 **`react-dom`**，返回 `{ pending, data, method, action }`：

- `pending`：父 `<form>` 是否正在提交
- `data`：正在提交的 `FormData`（没有提交时为 `null`）
- `method`：`'get'` 或 `'post'`
- `action`：父 form 的 action 函数引用；如果 action 是 URL 字符串则为 `null`

**最大的坑（官方文档专门标了 Pitfall）**：它只读取**父级** `<form>` 的状态，**不会**追踪同一个组件里渲染的 `<form>`。

{% raw %}
```tsx
import { useFormStatus } from 'react-dom';   // 注意：react-dom，不是 react

// ❌ pending 永远是 false —— 这个 Hook 不追踪本组件里渲染的 form
function Form() {
  const { pending } = useFormStatus();
  return <form action={submit}><button disabled={pending}>提交</button></form>;
}

// ✅ 把按钮抽成子组件，放在 <form> 内部
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>{pending ? '提交中…' : '提交'}</button>;
}
function Form() {
  return <form action={submit}><SubmitButton /></form>;
}
```
{% endraw %}

这个设计的好处是**免 props 透传**：设计系统里的 `<SubmitButton />` 塞进任何 form 都能自己知道在不在提交。

**`useFormStatus` vs `useActionState` 的 `isPending` 怎么选？** 简单：就在提交按钮上用 `useFormStatus`（不用透传）；需要在 form 外层或其他地方用这个状态，就用 `useActionState` 的 `isPending`。

### 5. Server Actions：'use server' 到底做了什么

给一个 async 函数加上 `'use server'`，它就变成了 Server Action——**只在服务端执行**，客户端调用它时，React 把参数序列化后发一个网络请求过去，服务端跑完再把返回值序列化送回来。

{% raw %}
```tsx
// app/actions.ts —— 文件顶部加指令，则所有导出都是 Server Action
'use server';

export async function createComment(formData: FormData) {
  const body = formData.get('body') as string;
  if (!body?.trim()) return { error: '内容不能为空' };
  await db.comment.create({ data: { body } });
  revalidatePath('/posts/1');     // 让受影响的服务端组件缓存失效
  return { error: null };
}

// 也可以内联写在 Server Component 里（指令必须是函数体第一句）
export default function Post() {
  async function addComment(formData: FormData) {
    'use server';
    // ...
  }
  return <form action={addComment}>...</form>;
}
```
{% endraw %}

几条硬规则：

- 指令必须在**函数体或模块的最开头**（注释可以放在它上面），**必须用单/双引号**，反引号不行。
- 只能用于 **async 函数**（底层是网络调用，天然异步）。
- **参数和返回值必须可序列化**。支持：原始类型、`FormData`、`Date`、`Map`/`Set`、`TypedArray`、`ArrayBuffer`、普通对象、数组、Promise、以及 Server Action 本身。**不支持**：函数（非 Server Action）、类实例、React 元素/JSX、Symbol（除 `Symbol.for` 注册的）。
- 客户端要 import Server Action，指令必须写在**模块层级**。
- Server Action 应该在 **Transition 中调用**；传给 `action` / `formAction` 时 React 自动包，手动调用就得自己 `startTransition`。

### 6. Server Actions 的安全红线（面试官很爱追问）

**`'use server'` ≠ 安全。** 它只是把函数暴露成了一个**公开的 HTTP 端点**，任何人都能直接构造请求打过来。所以：

{% raw %}
```tsx
'use server';
export async function deletePost(postId: string) {
  // ❌ 只看指令就以为安全了 —— 任何人传任意 postId 都能删
  await db.post.delete({ where: { id: postId } });

  // ✅ 每一个 Server Action 内部都要自己做鉴权 + 校验
  const session = await auth();
  if (!session?.user) throw new Error('Unauthorized');
  const post = await db.post.findUnique({ where: { id: postId } });
  if (post?.authorId !== session.user.id) throw new Error('Forbidden');
  await db.post.delete({ where: { id: postId } });
}
```
{% endraw %}

记住两句：**参数完全由客户端控制，一律当不可信输入；每个 Server Action 都是独立鉴权单元。**

还有一个定位问题：**Server Actions 是为"写"设计的，不适合拿来做数据获取**。框架通常一次只处理一个 action，且返回值不缓存。取数还是走 RSC 渲染或查询库（TanStack Query 等）。

### 7. React 19 表单的两个实用变化

- **提交成功后自动重置非受控表单**：给 `<form action={fn}>` 传函数，函数成功返回后，React 会自动清空里面的非受控字段。想保留原值就改用 `onSubmit` + `preventDefault` 的老写法，或者用 `useActionState` 把值回传成新的 `defaultValue`。
- **手动重置**：用 React 19 新增的 `requestFormReset`（`react-dom`）。

## 其实你每天都在用

- **点赞 / 收藏按钮**：点了立刻变红，请求失败了再变回去——`useOptimistic` 的标准场景。
- **提交评论**：输入框下面立刻冒出自己的评论（带个半透明表示"发送中"），服务端确认后变实——`useOptimistic` + `useActionState` 组合。
- **登录/注册表单**：按钮 loading、服务端返回"密码错误"直接显示在表单上——`useActionState` 的 `isPending` + `state.error`。
- **设计系统的提交按钮**：`<SubmitButton />` 丢进任何表单都知道当前在不在提交——`useFormStatus`。
- **后台管理的新建/编辑弹窗**：`useActionState` 接 Server Action，成功后 `revalidatePath` 刷新列表，一气呵成。
- **Todo 勾选、购物车加减**：都是"改一个字段"的小变更，特别适合乐观更新。
- **草稿保存**：注意 React 19 会自动重置非受控表单，所以"保存草稿后还想保留输入"要额外处理。

## 常见误解（FAQ）

- **❌ 误区1："useActionState 就是 useFormState 换个名"**
  不完全是。名字确实从 `useFormState`（react-dom）改成了 `useActionState`（react），而且**位置也搬到了 `react`**；更重要的是它不再局限于表单，配 `startTransition` 可以用于任何异步变更，返回值还多了第三个 `isPending`。

- **❌ 误区2："useOptimistic 失败了要手动写回滚逻辑"**
  不用。action 抛错（或 Transition 结束而真实值没变）时，React 渲染的就是当前的 `baseValue`，**乐观值自动消失**。你只需要 `catch` 里给用户一个提示。这正是它比手写 `setState` 再 `setState` 回去省心的地方。

- **❌ 误区3："useFormStatus 写在 form 组件里就能用"**
  恰恰不能。只读取**父级** `<form>`；写在渲染该 `<form>` 的同一个组件里，`pending` 永远是 `false`。必须把 Hook 放到 `<form>` 内部的**子组件**里。

- **❌ 误区4："Server Action 加了 'use server' 就只在服务端跑，所以是安全的"**
  危险的想法。它只是让函数可以从客户端被调到，**本质是一个公开 HTTP 端点**，参数完全由客户端控制。鉴权和参数校验必须自己在函数体内做。

- **❌ 误区5："Server Actions 可以当接口用来取数"**
  不建议。官方定位是**变更（mutation）**，框架一般一次只处理一个 action、返回值不缓存。取数请用 RSC 渲染或专门的请求库。

- **❌ 误区6："useOptimistic 可以传任意对象/函数当 action 值"**
  只要是 JS 值都行（它不走网络），但 **`reducer` 必须是纯函数**，且不要在里面做副作用。另外别在渲染阶段调用 `setOptimistic`，会直接抛 "Cannot update optimistic state while rendering"。

- **❌ 误区7："用了 React 19 的 form action 就不需要受控组件了"**
  两回事。`action` 管的是"提交流程"，输入框的 `value`/`onChange` 管的是"实时取值"。要做实时校验、格式化、按钮联动禁用，还是得受控（详见受控/非受控那篇）。

## 一句话总结

React 19 的 Actions 体系就三件事：**`useActionState` 管结果和 loading（记得 dispatch 要包 Transition 或当 action prop）、`useOptimistic` 管先给用户看什么（失败自动回滚，别手写）、`useFormStatus` 管深层子组件免透传（必须放在 form 的子组件里）**；而 Server Actions 让"调后端"变成"调函数"，但**它是公开端点，鉴权一条都不能少**。
