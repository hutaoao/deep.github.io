---
title: RN新架构JSI与Fabric深度解析
date: 2026-04-12
categories: [跨端开发, React Native]
tags: [前端, React Native, 跨端开发, JSI, Fabric, TurboModules, Codegen]
description: 从 Bridge 到 JSI + Fabric：拆解 RN 新架构三大组件（JSI/Fabric/TurboModules）的设计原理与面试要点。
layout: post
---

## 一句话概括

React Native 新架构做了三件事：JSI 让 JS 能直接持有并调用 C++ 对象（不再 JSON 序列化），Fabric 把布局计算从 UI 线程拆到独立 Shadow 线程，TurboModules 把原生模块从"启动全量加载"改成"按需懒加载"——三项合在一起，把 RN 从"可用的跨端方案"推到了"接近原生的跨端方案"。

## 核心知识点

### 1. JSI：从"传纸条"到"直接打电话"

```cpp
// JSI 的核心：HostObject 让 C++ 类对 JS 引擎可见
class CalculatorHost : public facebook::jsi::HostObject {
public:
  // JS 访问 calculator.add 时触发
  facebook::jsi::Value get(facebook::jsi::Runtime& rt,
                           const facebook::jsi::PropNameID& name) override {
    if (name.utf8(rt) == "add") {
      return facebook::jsi::Function::createFromHostFunction(
        rt, name, 2,
        [](facebook::jsi::Runtime& rt,
           const facebook::jsi::Value&,
           const facebook::jsi::Value* args,
           size_t count) -> facebook::jsi::Value {
          return facebook::jsi::Value(args[0].asNumber() + args[1].asNumber());
        });
    }
    return facebook::jsi::Value::undefined();
  }
};
```

```js
// JS 侧：直接同步调用，不经过任何序列化
const result = calculator.add(3, 5) // 8，箭头都没转一下
```

Bridge 的"传纸条"：`JS 调 → JSON.stringify → 入队 → 等 flush → 跨线程 → JSON.parse → 执行`，每次都有 4 次数据拷贝 + 类型信息丢失。JSI 的"直接打电话"：JS 引擎通过 C++ 指针直接跳转到目标函数，一次拷贝都没有。

### 2. Fabric：三阶段流水线

```
JS Thread               Shadow Thread           Main Thread
│                       │                       │
│  React Diff           │                       │
│  生成 Element Tree ───→ 构建 Shadow Tree       │
│                       │  Yoga 布局计算        │
│                       │  树对比（Diff Tree）   │
│                       │                       │
│                       │  原子提交 ────────────→ 批量更新 Native View
│                       │  (一次性打包所有变更)  │  create/update/delete
│                       │                       │  屏幕绘制
```

Fabric 三个核心改进：

-**独立 Shadow 线程**：旧架构里 Yoga 布局在 UI 线程跑，Fabric 拆到独立线程，布局计算和 UI 绘制不再互相阻塞。

-**原子提交（Atomic Commit）**：一轮渲染周期内的所有变更打包，一次性提交到 UI 线程，而不是每个组件改一次就通知一次。减少了 UI 线程被唤醒的次数和重排次数。

-**C++ 层实现**：Shadow Tree 纯 C++ 实现，JSI 直接访问，没有序列化开销。

### 3. TurboModules：按需加载，启动时只加载用到的

```cpp
// 旧架构：启动时 createNativeModules() 返回所有模块
// 不管用不用，全部初始化 → 启动慢、内存占用大

// TurboModules：懒加载
// JS 首次访问 NativeModules.BluetoothScanner 时才初始化
// 通过 JSI 注册为 HostObject，JS 直接持有 C++ 引用
```

Codegen 也值得一提：旧架构的 `@ReactMethod` 注解在运行时靠反射调用（慢 + 类型不安全），新架构用 Codegen 在编译时生成类型安全的 C++ 桥接代码，杜绝了 JS 传 string 但原生期望 int 这类运行时错误。

### 4. 新旧一张表看完

| 维度 | 旧架构 (Bridge) | 新架构 (JSI+Fabric) |
|------|----------------|---------------------|
| 通信 | 异步 JSON 序列化 | 同步 C++ 宿主对象 |
| 布局 | UI 线程跑 Yoga | 独立 Shadow 线程 |
| 模块加载 | 启动全量注册 | 按需懒加载 |
| 类型安全 | 运行时反射 | 编译时 Codegen |
| JS 引擎 | JSC（固定） | 可插拔（Hermes/JSC/V8） |
| 渲染提交 | 逐个通知 UI | 原子批量提交 |

### 5. 迁移到新架构最关键的三个变化

```js
// ① 同步调用的心智模型变化
// 旧：所有 NativeModule 调用都是异步的，习惯了 await
const result = await NativeModules.Storage.get('key')

// 新：部分 JSI 绑定是同步的，不会返回 Promise
// 需要分清楚哪些同步、哪些异步，避免误阻塞 JS 线程
const result = NativeModules.NewStorage.get('key') // 调用即返回

// ② NativeModule 需要重写为 TurboModule
// 不再用 @ReactMethod 注解，改用 Codegen 生成接口

// ③ 事件的 push → pull 转变
// 新架构推荐"JS 主动查询"而非"原生主动推送"
// 避免事件风暴和 JS 侧未初始化导致的丢事件问题
```

## 「其实你每天都在用」

1. **`useNativeDriver: true` 的动画就是新架构理念的预演** — 动画计算完全不经过 JS 线程，直接在原生线程跑，这正是 Fabric 想对所有渲染做的事
2. **React 的 concurrent mode 和 Fabric 的原子提交思路一致** — 都在做"攒一批变更，一次性提交"，减少中间状态的无效渲染
3. **Web Worker 的 `SharedArrayBuffer` 就是 Web 版的 JSI** — 从 postMessage 拷贝数据进化到共享内存直接访问，同样是为消除序列化开销
4. **TurboModules 的懒加载和 `React.lazy()` 思路一致** — 都是"不用就不加载"，减少首屏负担
5. **Codegen 和 tRPC/GraphQL Codegen 的编译时类型安全目标一致** — 运行时靠约定 → 编译时靠检查，bug 提前暴露

## 常见误解（FAQ）

**❌ 误区：「新架构比旧架构快，是因为用了 C++」**
语言切换（Java/ObjC → C++）带来的性能收益微乎其微。真正的加速来自：消除 JSON 序列化（数据拷贝 → 指针引用）、独立 Shadow 线程（布局不阻塞 UI）、原子提交（批量更新减少重排）。C++ 只是实现手段，不是性能原因。

**❌ 误区：「升级到新架构，现有代码不用改」**
Native Module 必须重写为 TurboModule（从 `@ReactMethod` 注解改为 Codegen 接口），自定义 ViewManager 要适配 Fabric。虽然 RN 0.74+ 提供了兼容层，但兼容层的性能不如原生新架构。真正的迁移需要一定的工作量。

**❌ 误区：「JSI 同步调用意味着可以直接在主线程做耗时操作」**
JSI 同步只是"调用方式同步"，原生方法内部的耗时操作（文件 IO、网络请求）仍然必须在后台线程执行。如果在 JSI 的 HostFunction 里做了耗时同步操作，会直接阻塞 JS 线程——这比旧架构更危险，因为旧架构至少异步不会阻塞 JS。

**❌ 误区：「迁移到新架构后，所有 Bridge 问题都没了」**
Fabric 并没有完全消灭异步——JS 线程通知 Shadow 线程"该布局了"依然是异步消息（JSI 只解决了 JS↔C++ 的同步调用，线程间调度仍有异步）。只是新架构把异步从"每次通信"缩小到了"每帧一次"，频率大幅降低。

## 一句话总结

RN 新架构不是给旧架构打了个补丁，而是把通信层从"翻译-传纸条-翻译"升级成了"共享母语直接对话"——JSI 让 JS 和 C++ 说同一种语言，Fabric 让三条线程各司其职不再互相等，TurboModules 让模块不再"一人吃饱全家不饿"地全量加载。
