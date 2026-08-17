---
layout: post
title: "computed 与 watch 原理"
date: 2026-02-22
categories: ["前端核心", "Vue"]
tags: ["Vue3", "computed", "watch", "watchEffect", "响应式"]
---

## 一句话概括

`computed` 和 `watch` 共享同一个底层 `ReactiveEffect`，区别只在两行配置：computed 加了 `scheduler`（依赖变只设 dirty）+ `lazy`（有人读才计算），watch 加了 `scheduler`（数据变执行回调）+ `immediate`（是否首次执行）。理解这两个标志位，就理解了两者全部行为差异。

## 核心知识点

### 1. computed——dirty 标志位的惰性求值

```ts
class ComputedRefImpl<T> {
  private _value!: T;
  private _dirty = true;  // 🔑 整个 computed 就靠这个 boolean

  get value() {
    trackRefValue(this);  // 收集 computed 自己的订阅者
    if (this._dirty) {
      this._dirty = false;
      this._value = this.effect.run()!;  // 有人读、且脏了，才计算
    }
    return this._value;
  }
}

// effect 的 scheduler：依赖变了不做计算，只开脏标志
// const effect = new ReactiveEffect(getter, () => {
//   if (!this._dirty) { this._dirty = true; triggerRefValue(this); }
// });
```

**核心逻辑：** 依赖变化 → 只设 `dirty = true` → 下次 `.value` 被读取时才真正计算。同一帧内依赖改 100 次，只算 1 次。

### 2. watch——精确指定源 + 新旧值对比

```ts
watch(source, (newVal, oldVal, onCleanup) => {
  // 副作用
}, { deep: true, immediate: true, flush: 'post' });
```

三种 flush 决定的是 **scheduler 的实现**：

| flush | 执行时机 | 场景 |
|-------|---------|------|
| `pre`（默认） | 组件更新**前** | 数据二次计算，和组件渲染合并 |
| `post` | DOM 更新**后** | 测量元素尺寸、操作 DOM |
| `sync` | 数据变立即同步执行 | ⚠️ 破坏批处理，除非确切知道后果否则别用 |

### 3. watchEffect——自动追踪 + 立即执行

```ts
const count = ref(0);
const name = ref('Vue');

watchEffect(() => {
  console.log(`${name.value}: ${count.value}`); // 自动收集 count 和 name
});
// 立即输出 "Vue: 0"  ← 等价 watch 同时设 { immediate: true }
```

**与 watch 的核心差异：** (1) 不需要显式声明监听源——回调里碰到谁就跟踪谁；(2) 回调里没有新旧值参数；(3) 初始化必然执行一次。

### 4. deep 选项——递归遍历触发全路径 track

```ts
function traverse(value, seen = new Set()) {
  if (!isObject(value) || seen.has(value)) return value;  // 循环引用终止
  seen.add(value);
  if (Array.isArray(value)) {
    value.forEach(item => traverse(item, seen));
  } else {
    for (const key in value) traverse(value[key], seen);
  }
  return value;
}
```

`deep: true` 的本质：对值做一次全量递归访问，路径上每个属性的 get 都被 track。**代价**：O(n) 遍历所有嵌套属性。万级节点的深层对象请用精确 getter 替代 `deep: true`。

### 5. 竞态处理——onCleanup 过期标记

```ts
watch(searchQuery, async (newQ, oldQ, onCleanup) => {
  let expired = false;
  onCleanup(() => { expired = true });  // 下次执行前先调这个

  const res = await fetch(`/api/search?q=${newQ}`);
  if (!expired) results.value = res.json();  // 没过期才更新
});
```

**原理：** watch 内部维护一个 `onCleanup` 回调变量，每次新回调执行前先调上一次的 cleanup → 标记过期 → 旧请求的响应被丢弃。这就是为什么快速连续输入时只有最后一次的结果会显示。

## 其实你每天都在用

- **购物车总价：** `computed(() => cart.reduce((sum, i) => sum + i.price * i.qty, 0))` —— 依赖没变直接从缓存读
- **搜索防抖：** `watch(keyword, debounce(fetchResults, 300))` —— 用户停手才发请求
- **路由参数自动拉数据：** `watch(() => route.params.id, fetchPage, { immediate: true })`
- **表单联动：** 选省份 → watch 省份 ref → 自动请求该省城市列表
- **echarts 图表响应：** `watchEffect(() => chart.setOption(buildOption(data.value)))` —— 数据变图表自动更新

## 常见误解（FAQ）

- **❌ 误区：「computed 每次访问都重新计算」** 只有 `_dirty === true` 时才计算。模板里用三次 `{ { doubleCount } }` 只算一次——第一次 dirty→计算→关 dirty，后两次直接读缓存。

- **❌ 误区：「computed 里可以放副作用」** 技术上不报错，但后果灾难：dirty 为 false 时副作用不执行（你以为每次都会发的请求不发），dirty 为 true 时如果没人读 `.value` 也不执行。副作用严格放 watch/watchEffect。

- **❌ 误区：「watch(reactiveObj, cb) 就是浅监听」** 不是——watch 检测到 source 是 reactive 对象会自动启用 deep。手动 `deep: false` 能显式关掉（虽然极少需要）。

- **❌ 误区：「watch 回调里不需要 nextTick 就能安全操作 DOM」** 看你选的 flush。`flush: 'pre'`（默认）的回调在 DOM 更新前执行——此时 DOM 还是旧的，操作它需要在回调里再套 `await nextTick()`。`flush: 'post'` 才直接拿到新 DOM。

## 一句话总结

computed 是"有人读我吗？脏了算一下"（懒），watch 是"你变了？我干活"（勤），watchEffect 是"我不挑，见谁变都干"——共享同一个 effect 引擎，差异只在标志位的排列组合。
