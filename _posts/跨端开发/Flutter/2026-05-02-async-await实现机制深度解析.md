---
layout: post
title: "Dart async/await 实现机制：从状态机到事件循环，拆开给你看"
date: 2026-05-02
categories: [跨端开发, Flutter]
tags: [Dart, async/await, Future, 事件循环, 微任务, 面试]
---

## 一句话概括

async/await 不是魔法——编译器把每个 async 函数编译成一个**状态机**，每个 await 是切分点，事件循环按"微任务先于事件队列"的规则调度状态机的执行，这就是 Dart 单线程不卡顿的底层原因。

## 核心知识点

### 1. 事件循环的执行顺序：微任务 → 事件队列

```dart
void main() {
  print('1');
  Future(() => print('4'));                     // 事件队列
  Future.microtask(() => print('2'));           // 微任务
  Future.value('3').then(print);                // 微任务
  print('5');
}
// 输出：1 → 5 → 2 → 3 → 4
```

规则：**每轮事件循环先清空所有微任务，再从事件队列取一个**。`Future.value().then()` 回调在微任务，`Future()` 构造函数体在事件队列。这是面试必考题。

### 2. async 函数编译为状态机

```dart
// 你写的
Future<int> compute(int a, int b) async {
  final x = await fetchA();     // 挂起点1
  final y = await fetchB(x);    // 挂起点2
  return y + b;
}

// 编译后等价于（简化）
Future<int> compute(int a, int b) {
  var state = 0; dynamic result;
  final c = Completer<int>();

  void step() {
    switch (state) {
      case 0:
        state = 1;
        fetchA().then((v) { result = v; step(); });
        break;
      case 1:
        state = 2;
        fetchB(result).then((v) { result = v; step(); });
        break;
      case 2:
        c.complete(result + b);
    }
  }
  step();
  return c.future;
}
```

每个 await = 一个 case，then 回调推进状态机。这就是为什么 await 不占线程——它只是**注册回调然后让出控制权**。

### 3. mount / dispose 与 async 的致命组合

```dart
// ❌ 常见错误：Widget 销毁后还调用 setState
@override
void initState() {
  super.initState();
  fetchUser().then((user) {
    // Widget 可能已经 dispose 了！
    setState(() => this.user = user); // 运行时错误！
  });
}

// ✅ 正确做法：
@override
void initState() {
  super.initState();
  _userFuture = fetchUser();
}
// 在 build 中用 FutureBuilder 或检查 mounted
```

核心问题：async 回调可能在 Widget 销毁后才执行。Flutter 中用 `mounted` 检查或 `FutureBuilder` 来保证安全。

### 4. await + try/catch = 同步式错误处理

```dart
Future<void> loadData() async {
  try {
    final user = await api.fetchUser();
    final posts = await api.fetchPosts(user.id);
    print('加载完成');
  } on TimeoutException {
    print('请求超时，使用缓存数据');
  } catch (e, st) {
    print('未知错误: $e');
    debugPrintStack(stackTrace: st);
  } finally {
    hideLoading(); // 无论成败都关 loading
  }
}
```

async 函数内 try/catch 能捕获 await 抛出的异常，写起来像同步代码。比 then 链的 catchError 更直观。

### 5. Future 的调度时机总结

```dart
// 四种调度方式，优先级从高到低：
// 1. 同步代码（main, sync*）
// 2. 微任务（scheduleMicrotask, Future.value().then, Future.microtask()）
// 3. 事件队列（Future(), Future.delayed(), IO 回调）
// 4. Timer.run, Future.delayed(Duration(...))
```

一条黄金法则：**Dart 永远不会中断正在执行的同步代码去跑微任务或事件队列**。微任务只在当前调用栈清空后才执行。

---

## 其实你每天都在用

- **`setState()` 不能放在 async 回调里**：Flutter 报错 "setState called after dispose" 就是因为 async 回调执行时 Widget 已销毁
- **`await Future.delayed(Duration(seconds: 1))`**：这不是 sleep，不会卡 UI
- **`FutureBuilder`**：Flutter 内置的对 Future 状态的响应式封装，自动处理 loading/data/error
- **`initState` 中的 `WidgetsBinding.instance.addPostFrameCallback`**：在帧渲染完成后执行，比微任务更晚
- **日志框架中 `Future.microtask` 延迟错误上报**：让当前帧正常渲染完再处理错误

---

## 常见误解（FAQ）

**❌ 误区：「await 后面的代码一定在微任务里执行」**

不一定。如果 await 的 Future 已经在事件队列中，那 await 后面的代码也在事件队列中执行。`await Future(() => 1)` 和 `await Future.value(1)` 的 then 回调在不同队列。

**❌ 误区：「Dart 是单线程所以不能做真并发」**

Dart 有 **Isolate**（隔离区），每个 Isolate 独立内存、独立事件循环。可以在后台 Isolate 做大量计算，UI Isolate 不受影响。`compute()` 函数就是 Isolate 的便捷封装。

**❌ 误区：「`for (var item in items) await process(item)` 和 `Future.forEach` 一模一样」**

基本一样都是串行。并行用 `Future.wait(items.map(process))`。

---

## 一句话总结

async/await = 编译器状态机 + 事件循环调度 = 单线程也能写出干净利落的异步代码。搞懂这个，80% 的 Flutter 异步 Bug 都能迎刃而解。
