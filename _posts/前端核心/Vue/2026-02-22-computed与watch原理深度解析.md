---
layout: post
title: "computed 与 watch 原理"
date: 2026-02-22
categories: ["前端核心", "Vue"]
tags: ["Vue3", "computed", "watch", "响应式"]
---

## 一句话概括

`computed` 是**带缓存的派生值**，只在被读取且依赖变化时才重新计算；`watch` 是**响应式的副作用触发器**，监听数据变化执行回调。两者共享 `ReactiveEffect` 底层，差异仅在 scheduler 和 dirty 标志位。

## 核心知识点

### 1. computed——dirty 标志位 + 惰性求值

```ts
class ComputedRefImpl<T> {
  private _value!: T;
  private _dirty = true;  // 核心：脏标志

  get value() {
    trackRefValue(this);  // 收集 computed 自己的订阅者
    if (this._dirty) {
      this._dirty = false;
      this._value = this.effect.run()!;  // 执行 getter
    }
    return this._value;
  }
}

// effect 的 scheduler：依赖变化时只设 dirty
// new ReactiveEffect(getter, () => {
//   if (!this._dirty) { this._dirty = true; triggerRefValue(this); }
// })
```

**关键设计**：依赖变化时不重新计算，只设 dirty。下次读取 `.value` 时才真正执行 getter。一帧内依赖改 100 次，只计算 1 次。

### 2. watch——精确指定依赖 + 新旧值对比

```ts
watch(source, (newVal, oldVal, onCleanup) => {
  // 副作用逻辑
}, { deep: true, immediate: true, flush: 'post' });
```

三种 flush 模式：

| flush | 执行时机 | 使用场景 |
|-------|---------|---------|
| `pre`（默认） | 组件更新前 | 数据二次处理，合并到同一渲染周期 |
| `post` | 组件 DOM 更新后 | 测量元素尺寸、操作 DOM |
| `sync` | 数据变化立即执行 | ⚠️ 破坏批处理，非必要不用 |

### 3. watchEffect——自动追踪 + 立即执行

```ts
const count = ref(0);
const name = ref('Vue');

// 会自动追踪 count 和 name
watchEffect(() => {
  console.log(`${name.value}: ${count.value}`);
});
// 立即输出 "Vue: 0"
```

**与 watch 的区别**：不需要显式声明依赖源，初始化立即执行一次，回调里没有新旧值。

### 4. deep 选项的实现——递归遍历

```ts
function traverse(value: unknown, seen = new Set()): unknown {
  if (!isObject(value) || seen.has(value)) return value;
  seen.add(value);
  if (isArray(value)) {
    value.forEach(item => traverse(item, seen));
  } else {
    for (const key in value) traverse(value[key], seen);
  }
  return value;
}
```

`deep: true` 时，每次取值后递归访问所有嵌套属性，触发的 get 全部被 track，形成"全路径依赖"。**性能警告**：对万级节点的深层对象，`deep` 是 O(n) 操作，改用精确 getter 更高效。

### 5. 竞态处理——onCleanup

```ts
watch(query, async (newQ, oldQ, onCleanup) => {
  let expired = false;
  onCleanup(() => { expired = true });  // 新轮次执行前调用上一轮的清理

  const result = await fetch(`/api?q=${newQ}`);
  if (!expired) results.value = await result.json();
});
```

**原理**：每个 watch 回调维护一个 `cleanup` 变量，下次执行前先调前一次的 cleanup → 标记过期 → 旧请求的结果被丢弃。

## 其实你每天都在用

- **购物车总价**：`computed(() => cart.reduce(...))`，改任意商品数量总价自动更新，依赖不变直接从缓存读
- **搜索防抖**：`watch(searchQuery, debouncedFetch, { flush: 'post' })`，等 DOM 更新完再发请求
- **路由参数变化自动拉数据**：`watch(() => route.params.id, fetchPage, { immediate: true })`
- **表单联动**：选了省份 → 城市列表自动刷新 → watch 省份 ref 触发
- **echarts 图表自适应**：`watchEffect(() => { chart.setOption(...) })`，数据一改图表就更新

## 常见误解

- **❌ 误区：「computed 每被访问一次就计算一次」** 只有 `_dirty === true` 时才计算。模板中用三次 `{{ double }}` 只计算一次——第一次 dirty→计算→关 dirty，后两次直接读缓存。

- **❌ 误区：「computed 里可以有副作用」** 技术上可以，但后果严重：`_dirty === false` 时副作用不会执行（依赖没变），`_dirty === true` 时又可能在无人读取的情况下反复执行（如果有 watch 订阅了这个 computed）。副作用严格放在 watch/watchEffect 里。

- **❌ 误区：「`watch(reactiveObj, cb)` 等价于 `deep: true`」** 差不多——watch 检测到 source 是 reactive 对象会自动启用 deep。但有一个细小差异：手动设置 `deep: false` 可以显式关闭（虽然并不推荐）。

- **❌ 误区：「`watchEffect` 就是对 `watch` 的封装」** 底层是同一个 `doWatch` 函数，但语义不同。`watchEffect` 强调"自动追踪副作用"，`watch` 强调"精确监听某个数据源"。选型标准：需要旧值 → watch；只想干点副作用 → watchEffect。

## 一句话总结

computed 问自己"有人读我吗？"（有才计算）；watch 问自己"数据变了吗？"（变了就干）——一个懒、一个勤，底层都是同一个 effect。
