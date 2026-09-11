---
layout: post
title: "TanStack Query 请求状态管理原理"
date: 2026-11-28 00:00:00 +0800
categories: ["前端核心", "React"]
tags: [React, TanStackQuery, 服务端状态缓存, staleTime, gcTime, 乐观更新]
description: >
  面试向讲清 TanStack Query（React Query v5）：服务端状态为什么不能塞进 Redux/Zustand、queryKey 为什么就是缓存地址、
  staleTime 与 gcTime 到底差在哪、isPending / isFetching / isLoading 三个状态怎么区分、请求去重是怎么发生的、
  invalidateQueries 默认只重拉 active 查询、以及乐观更新必须走的三步（cancel → 覆盖 → 回滚）。
---

## 一句话概括

TanStack Query（v5 之后 React Query 的正式名字）**不是请求库，它是"服务端状态的缓存与同步层"**。

分工是这样的：`fetch` / `axios` 负责"怎么把请求发出去"，TanStack Query 负责"这份数据放哪、多久算过期、几个组件同时要怎么只发一次、改完数据哪些缓存要失效、失败了怎么退回来"。所以它不代替 axios，反而经常和 axios 一起用。

面试为什么爱问它？因为它把你平时用 `useEffect` 手写、但基本写不对的四件事标准化了：**缓存、去重、竞态、失效**。能答清楚这题，说明你对"前端状态该分类管理"这件事有认知，不是只会往 store 里塞。

一句话记住：**它管的是"别人家（服务端）数据在你这的一份缓存"，不是你自己的 UI 状态。**

## 核心知识点

### 1. 地基：先分清「客户端状态」和「服务端状态」

这是整道题的入口，答不出这层，后面全是散的。

| | 客户端状态 | 服务端状态 |
| --- | --- | --- |
| 谁拥有它 | 你自己（浏览器） | 服务端，你只有一份**拷贝** |
| 典型例子 | 弹窗开关、tab 选中、表单草稿、主题 | 用户列表、订单详情、字典配置 |
| 会不会"自己变" | 不会，只有你改它才变 | 会，别人改了数据库，你这份就脏了 |
| 需要的能力 | 读写、持久化 | 缓存、失效、重新拉取、竞态处理 |
| 该用什么 | `useState` / Zustand / Redux | TanStack Query / SWR |

用 `useEffect` 手写取数，你会踩四个坑，而且很难全填平：

{% raw %}
```tsx
// ❌ 经典手写取数：四个坑全中
function UserList() {
  const [data, setData] = useState<User[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    setLoading(true);
    fetchUsers()
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);
  // 坑1：没缓存 —— 组件卸载再挂载，又 loading 一遍
  // 坑2：没去重 —— 页面上有 3 个组件要这份数据，就发 3 次请求
  // 坑3：竞态  —— 依赖变了快速切换，先发的慢请求后回来会把新数据覆盖掉
  // 坑4：没失效 —— 新增用户后，你得自己记得去刷新列表，忘一次就是一次线上 bug
}
```
{% endraw %}

换成 TanStack Query，四个坑一次性交给框架：

{% raw %}
```tsx
// ✅ 描述"我要什么数据"，剩下的它管
function UserList() {
  const { data, isPending, isFetching, error } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
    staleTime: 60_000,      // 1 分钟内认为数据新鲜，不重复请求
  });

  if (isPending) return <Skeleton />;          // 只有"首次加载且无缓存"才转圈
  if (error) return <ErrorTip error={error} />;
  return (
    <>
      {isFetching && <Refreshing />}           {/* 后台刷新只显示小提示 */}
      {data.map(u => <UserRow key={u.id} user={u} />)}
    </>
  );
}
```
{% endraw %}

### 2. queryKey 就是这份缓存在内存里的「地址」

这是最容易答浅的一问。以下几个结论要能直接背：

1. **queryKey 顶层必须是数组**。可以是 `['todos']`，也可以是 `['todos', 'list', { page: 1, status: 'done' }]`。
2. **哈希是确定性的**：对象里的**键顺序无关**，`['todos', { status, page }]` 和 `['todos', { page, status }]` 是同一个 key。但**数组项的顺序有关**，`['todos', status, page]` 和 `['todos', page, status]` 是两个 key。
3. **queryFn 用到了哪个会变的变量，就必须把那个变量写进 key**，否则缓存会张冠李戴。
4. key 要**层级化**，方便"精确失效"和"按前缀批量失效"。

{% raw %}
```tsx
// ❌ 翻页翻到第 2 页，key 还是 ['todos']，缓存里拿到的是第 1 页
useQuery({ queryKey: ['todos'], queryFn: () => fetchTodos(page) });

// ✅ page 是 queryFn 的依赖，必须进 key；翻页自动重新拉取
useQuery({ queryKey: ['todos', 'list', { page }], queryFn: () => fetchTodos(page) });
```
{% endraw %}

{% raw %}
```tsx
// ✅ 层级化 key：既能精确失效单条，也能按前缀失效整棵
queryClient.invalidateQueries({ queryKey: ['todos'] });              // 全部 todos 相关
queryClient.invalidateQueries({ queryKey: ['todos', 'list'] });      // 只失效列表
queryClient.invalidateQueries({ queryKey: ['todos', 'detail', id] }); // 只失效一条详情
```
{% endraw %}

### 3. 必问：staleTime 和 gcTime 到底差在哪

**一句话：staleTime 管"能不能重新取"，gcTime 管"要不要丢掉"。** 两个完全正交的维度，别混。

| | `staleTime`（新鲜期） | `gcTime`（回收期） |
| --- | --- | --- |
| 默认值 | `0`（拿到即过期） | `5 * 60 * 1000`，即 5 分钟；SSR 下是 `Infinity` |
| 回答的问题 | 这份数据多久内**不用重新请求**？ | 没人用了之后，还在内存里**留多久**？ |
| 归零后发生什么 | 允许在触发时机上后台重新拉取 | 缓存被垃圾回收，下次挂载重新 loading |
| v4 里的名字 | 同名 | 叫 `cacheTime`（v5 改名） |

一个查询的完整生命周期，按顺序列一遍（面试照着说很顺）：

1. 首个组件挂载 → 缓存里没有 → `isPending = true`，发请求 → 数据写入缓存；
2. 接到响应那一刻起开始计时 `staleTime`，**此刻数据还"新鲜"**；
3. `staleTime` 走完 → 数据变 **stale（陈旧）**。注意：陈旧 ≠ 不能用，它照样显示，只是下次有触发时机就会后台刷新；
4. 同 key 的第二个组件挂载 → **立刻拿到缓存数据渲染**（不闪 loading），同时因为已经 stale，后台发一次刷新；
5. 所有用它的组件都卸载 → 这个查询变 **inactive**，`gcTime` 开始计时；
6. `gcTime` 内又有组件挂载 → 立刻用缓存，且**不管 stale 与否都会后台刷新一次**；
7. `gcTime` 走完还没人用 → 缓存被删掉，下次挂载回到第 1 步。

**所以：一份数据可以同时是 "stale 但还在缓存里"，也可以是 "fresh 但组件全卸载了"。** 这句话说出来，面试官就知道你真懂了。

{% raw %}
```tsx
// ✅ 静态字典：几乎不变，直接钉死
useQuery({ queryKey: ['countries'], queryFn: fetchCountries, staleTime: Infinity });

// ✅ 订单状态：几秒就变
useQuery({ queryKey: ['order', id], queryFn: () => fetchOrder(id), staleTime: 3_000 });
```
{% endraw %}

### 4. isPending / isFetching / isLoading：三个状态别张冠李戴

v5 里一个查询有**两套正交的状态**，这是很多人没意识到的：

- `status`：`'pending' | 'error' | 'success'` —— **有没有数据**；
- `fetchStatus`：`'fetching' | 'paused' | 'idle'` —— **此刻在不在发请求**。

派生出来的布尔值：

| 字段 | 含义 | 什么时候用 |
| --- | --- | --- |
| `isPending` | 还没有任何成功的数据（连缓存都没有） | 首屏骨架屏 |
| `isFetching` | `queryFn` 正在执行，**包括首次加载和后台刷新** | 右上角"更新中…"小转圈 |
| `isLoading` | **`isFetching && isPending`**，即"首次加载正在进行" | 只想在第一次转圈时用 |
| `isPlaceholderData` | 当前显示的是 placeholder（比如翻页时暂留的上一页） | 列表变灰但不清空 |

{% raw %}
```tsx
// ❌ 用 isFetching 控制骨架屏：每次后台刷新整页闪回 loading，体验灾难
if (isFetching) return <Skeleton />;

// ✅ 骨架屏用 isPending，后台刷新只显示右上角小提示
if (isPending) return <Skeleton />;
return <>{isFetching && <Spin />}<Table data={data} /></>;
```
{% endraw %}

### 5. 请求去重和自动刷新：它是怎么发生的

**去重**：同一时刻 3 个组件都要 `['users']`，缓存里只有一个 `Query` 实例，因此**只发一次网络请求**，3 个组件共享同一份结果（它们的 `isFetching` 会同步变化）。这也是为什么你要靠 key 而不是靠组件来组织数据。

**自动刷新的触发时机**（默认是"激进但合理"的）：

| 时机 | 默认值 | 说明 |
| --- | --- | --- |
| 新实例挂载 | 开启 | 只有数据 stale 才真的发请求 |
| 窗口重新聚焦 | 开启 | 切回标签页自动同步，同样要求 stale |
| 网络重连 | 开启 | 断网恢复后立刻补数据 |
| `refetchInterval` | 关闭 | 传毫秒数可轮询，**不受 staleTime 影响** |
| 手动 `invalidateQueries` / `refetch` | — | 业务主动触发 |

**失败重试**：客户端默认重试 **3** 次（SSR 环境默认 0 次），延迟是指数退避 `min(1000 * 2^n, 30000)`，也就是 1s → 2s → 4s，封顶 30s。

{% raw %}
```tsx
// ✅ 明确的 4xx 不重试，网络类错误才重试
useQuery({
  queryKey: ['user', id],
  queryFn: () => fetchUser(id),
  retry: (failureCount, error) => {
    if (error instanceof ApiError && error.status >= 400 && error.status < 500) return false;
    return failureCount < 2;
  },
});
```
{% endraw %}

### 6. 改数据：invalidateQueries 还是 setQueryData

两条路，取舍一句话：**要正确性用 invalidate，要即时反馈用 setQueryData（但事后最好还是 invalidate 兜底）。**

| | `invalidateQueries` | `setQueryData` |
| --- | --- | --- |
| 本质 | 标记为 stale + 重新发请求 | 直接改缓存里的数据 |
| 网络请求 | 有（多一次往返） | 无 |
| 数据正确性 | 一定和服务端一致 | 靠你自己和服务端返回对齐 |
| 适合 | 默认首选 | 服务端已返回新对象，省一次请求；或乐观更新 |

**一个高频丢分点**：`invalidateQueries` 会把它匹配到的**所有**查询标记为 stale，但**默认只重新拉取处于 active 状态的查询**（内部 `refetchType` 默认 `'active'`）。那些已经卸载的查询只是被标记，等下次挂载时才会重新取。想强制全拉，得显式写 `refetchType: 'all'`。

{% raw %}
```tsx
const queryClient = useQueryClient();

const addMutation = useMutation({
  mutationFn: (name: string) => createUser(name),
  onSuccess: (created) => {
    // 服务端已经把新对象返回了：先塞进详情缓存，省一次请求
    queryClient.setQueryData(['users', 'detail', created.id], created);
    // 列表内容变了，交给框架去重拉；默认只重刷当前正在显示的列表
    queryClient.invalidateQueries({ queryKey: ['users', 'list'] });
  },
});
```
{% endraw %}

### 7. 乐观更新三步走（加分题）

"点赞 / 收藏"这类场景，你想点了立刻变色，失败了再退回来。官方推荐的写法是三步，**缺一步就会出 bug**：

1. `onMutate` 里先 `cancelQueries` —— **把正在飞的请求掐掉**。不掐的话，那个慢请求回来会用旧数据把你的乐观值覆盖掉，这是最常见的翻车点；
2. `setQueryData` 写入乐观值，同时把旧值塞进 `context`；
3. `onError` 用 context 里的旧值回滚，`onSettled` 无论成败都 `invalidate` 兜底。

{% raw %}
```tsx
const queryClient = useQueryClient();

const likeMutation = useMutation({
  mutationFn: (id: string) => likeTodo(id),

  // 1️⃣ 掐掉在途请求 + 存旧值
  onMutate: async (id) => {
    await queryClient.cancelQueries({ queryKey: ['todos', 'detail', id] });
    const previous = queryClient.getQueryData<Todo>(['todos', 'detail', id]);
    // 2️⃣ 写乐观值
    queryClient.setQueryData<Todo>(['todos', 'detail', id], (old) =>
      old ? { ...old, liked: true, likes: old.likes + 1 } : old,
    );
    return { previous };   // 返回值会成为 onError / onSettled 的第三个参数
  },

  // 3️⃣ 失败回滚（第三个参数就是 onMutate 的返回值，老文档里习惯叫它 context）
  onError: (_err, id, snapshot) => {
    if (snapshot?.previous) {
      queryClient.setQueryData(['todos', 'detail', id], snapshot.previous);
    }
  },

  // 3️⃣ 无论成败，最后都以服务端为准再拉一次
  onSettled: (_d, _e, id) => {
    queryClient.invalidateQueries({ queryKey: ['todos', 'detail', id] });
  },
});
```
{% endraw %}

### 8. v5 相对 v4 的破坏性变更（老项目迁移常问）

| v4 | v5 | 说明 |
| --- | --- | --- |
| `cacheTime` | `gcTime` | 换个更能表达"垃圾回收"的名字 |
| `status: 'loading'` | `status: 'pending'` | `isLoading` 语义变成 `isFetching && isPending` |
| `isInitialLoading` | `isLoading` | 旧字段已废弃 |
| `keepPreviousData: true` | `placeholderData: keepPreviousData` | 变成函数形式，从包里导入 `keepPreviousData` |
| `useQuery(key, fn, opts)` | `useQuery({ queryKey, queryFn, ...opts })` | 只保留单对象参数 |
| `useQuery` 的 `onSuccess/onError/onSettled` | **已移除** | 只有 mutation 和全局 `QueryCache` 还保留回调 |
| 其它 | `useInfiniteQuery` 需必传 `initialPageParam` + `getNextPageParam` | 分页 API 重做 |

{% raw %}
```tsx
// ✅ v5 翻页保留上一页数据（骨架不清空、不闪 loading）
import { keepPreviousData, useQuery } from '@tanstack/react-query';

const { data, isPlaceholderData } = useQuery({
  queryKey: ['todos', 'list', { page }],
  queryFn: () => fetchTodos(page),
  placeholderData: keepPreviousData,   // 注意：这是函数引用，不是布尔值
});
```
{% endraw %}

关于 v5 移除 `useQuery` 回调这件事，面试官常追问"那我拿到数据要做什么怎么办"——答案是**用 `select` 做派生、用 `useEffect` 响应 `data` 变化、或者把副作用放到 mutation 的 `onSuccess` 里**，本来就不该挂在查询上。

### 9. 它和 Redux / Zustand 是什么关系

**不是替代，是分工。** 这是最能体现"状态分类"认知的一问：

- **服务端数据（列表、详情、字典）** → TanStack Query，它天生会缓存、失效、去重；
- **客户端 UI 状态（弹窗开关、主题、tab）** → Zustand / Context；
- **复杂客户端领域状态（编辑器、表单向导、购物车流程）** → Redux Toolkit / Zustand。

**反面教材**：把接口返回的数据 `dispatch` 进 Redux，然后手写一堆 `loading/error/refresh` 的 action，最后还得自己处理缓存过期——这就是在用 Redux 重新实现一个更差的 TanStack Query。

## 其实你每天都在用

1. **后台管理系统的列表页**：翻页、搜索、筛选参数全进 `queryKey`，切回来自动刷新，不用手写刷新按钮。
2. **详情页从列表点进去**：列表已经缓存过详情，详情页首屏直接出数据（零 loading），同时后台静默刷新。
3. **切窗口回来数据自动更新**：用户去别的标签页待了半小时，切回来列表自动同步，你一行代码没写。
4. **点赞 / 收藏 / 关注**：乐观更新，点了立刻变色，失败自动退回去。
5. **下拉刷新和"更新中"提示**：`isFetching` 直接驱动右上角那个小转圈。
6. **无限滚动**：`useInfiniteQuery` + `getNextPageParam`，比自己维护 `page` 和拼接数组省心得多。
7. **多个组件共用同一份用户信息**：Header 和设置页都要 `['me']`，只发一次请求。
8. **断网重连后自动补数据**：地铁里信号断了，出来之后页面自己就对了。

## 常见误解（FAQ）

**❌ 误区1："TanStack Query 是 axios 的替代品，用它就不用 axios 了。"**

不是一层东西。`queryFn` 里你照样写 `fetch` 或 `axios`，它只关心"你返回 data 或 throw error"。**它不提供拦截器、不提供 baseURL、不提供超时配置**——那些还是 axios 的活。

**❌ 误区2："staleTime 就是缓存时间，设长了数据就一直在。"**

错。缓存留多久是 `gcTime` 决定的（默认 5 分钟）。`staleTime` 只决定"这段时间内不重新发请求"。你把 `staleTime` 设成 1 小时，`gcTime` 保持默认 5 分钟，结果就是：组件卸载 5 分钟后缓存照样没了。反过来，`gcTime` 设再长也不影响重新请求的频率。

**❌ 误区3："isLoading 就是'正在加载'，我拿它控制 loading 就行。"**

v5 里 `isLoading = isFetching && isPending`，**只在首次加载（缓存里没有任何数据）时为 true**。后台刷新时它是 `false`。所以"首屏骨架屏"用 `isPending`，"更新中"用 `isFetching`，两个都要有，别只留一个。

**❌ 误区4："invalidateQueries 会把所有匹配的查询立刻重新拉一遍。"**

只拉**active**的。源码里 `refetchType` 默认就是 `'active'`，已经卸载的查询只被标记成 stale，等下次挂载才重新取。想无条件全拉要显式写 `refetchType: 'all'`。

**❌ 误区5："queryKey 里放对象不安全，属性顺序变了会导致缓存 miss。"**

不会。对象会被**确定性哈希**，键顺序无关，`{ status, page }` 和 `{ page, status }` 是同一个 key。但**数组项的顺序是有关系的**，`['todos', status, page]` 和 `['todos', page, status]` 是两个不同的缓存。别把变量裸着往数组里堆，统一放一个对象里最稳。

**❌ 误区6："乐观更新只要 setQueryData 覆盖一下就行了。"**

三步缺一不可，最容易漏的是第一步 `cancelQueries`。不取消在途请求的话，那个请求回来时服务端还是旧值，会把你的乐观值直接覆盖回去，表现就是"点赞闪一下又变回去了"。另外 `onError` 里不回滚，失败后 UI 会停在假数据上。

**❌ 误区7："我在 useQuery 里写 onSuccess 去做副作用。"**

v5 已经把这些回调从查询里移除了，写出来直接不生效（而且类型会报错，因为选项里没这个字段）。需要派生数据用 `select`，需要响应数据变化用 `useEffect` 依赖 `data`，副作用该放到 mutation 的 `onSuccess` 里。

## 一句话总结

**TanStack Query 是服务端状态的"缓存 + 同步"层：queryKey 定位缓存，staleTime 决定什么时候重新取，gcTime 决定没人用之后留多久，改完数据用 invalidate 让它自己重新同步；它不是 Redux 的竞品，而是把"本就不该放进 Redux 的那部分状态"接管走了。**
