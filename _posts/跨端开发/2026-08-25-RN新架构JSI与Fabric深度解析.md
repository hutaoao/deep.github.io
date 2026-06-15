---
title: RN新架构JSI与Fabric深度解析
date: 2026-08-25
categories: [跨端开发, React Native]
tags: [前端, React Native, 跨端开发, JSI, Fabric]
description: 深入解析React Native新架构核心——JavaScript Interface与Fabric渲染器的设计原理、实现机制与工程实践。
---

## 一句话概括

React Native新架构通过JSI（JavaScript Interface）取代了旧有的Bridge桥接层，配合Fabric渲染器实现了JS与原生层的同步调用和异步渲染解耦，从根本上突破了传统跨端框架的性能瓶颈。

## 背景与意义

React Native自2015年开源以来，凭借"Learn Once, Write Anywhere"的理念迅速获得了大量开发者的青睐。然而，旧架构中基于JSON序列化的Bridge通信机制长期为人诟病：每次JS与原生层的交互都需要经过序列化-传输-反序列化的过程，这种异步消息队列模式在高频交互场景（如手势拖拽、列表快速滚动）中不可避免地引入了几十毫秒的延迟。

2021年，React Native团队正式推出了新架构（New Architecture），其核心就是JSI（JavaScript Interface）和Fabric渲染器。这一变革不仅仅是性能微调，而是对整个通信层和渲染层进行了重写。理解JSI与Fabric的工作机制，已经成为每一位React Native中高级开发者必须掌握的知识。

## 概念与定义

**JSI（JavaScript Interface）** 是一个轻量级的C++层，它暴露给JavaScript引擎一组底层接口，允许JS代码直接获取对C++对象的引用并调用其方法，而无需经过JSON序列化。简单来说，JSI把"JS调用原生"从异步消息变成了同步函数调用。

**Fabric渲染器** 是React Native的新一代UI渲染引擎。它在Shadow Tree中构建完整的布局树，然后通过JSI与原生端通信，最终由原生平台（Android/iOS）进行实际绘制。Fabric将渲染流水线拆分为"影子树创建"和"原生提交"两个阶段，使JS线程的渲染工作不再阻塞UI线程。

**Turbo Modules** 是基于JSI的原生模块系统。传统Native Modules通过Bridge注册，调用时走异步消息队列；Turbo Modules在初始化时通过JSI在JS侧创建对应的C++宿主对象（Host Object），调用时直接同步执行。

## 最小示例

以下是一个极简的Turbo Module定义，展示JSI如何实现直接的C++/JS绑定：

```cpp
// C++侧：定义一个JSI宿主对象
#include <jsi/jsi.h>

using namespace facebook::jsi;

class CalculatorHostObject : public HostObject {
public:
  std::vector<PropNameID> getPropertyNames(Runtime& rt) override {
    return {PropNameID::forUtf8(rt, "add")};
  }

  Value get(Runtime& rt, const PropNameID& name) override {
    auto propName = name.utf8(rt);
    if (propName == "add") {
      return Function::createFromHostFunction(
        rt, name, 2,
        [](Runtime& rt, const Value& thisVal, const Value* args, size_t count) -> Value {
          double a = args[0].asNumber();
          double b = args[1].asNumber();
          return Value(a + b);
        });
    }
    return Value::undefined();
  }
};
```

然后在JS侧，JSI让对这个C++对象的调用变得像本地函数一样自然：

```javascript
// JS侧：直接同步调用
import { NativeModules } from 'react-native';

// 通过JSI获取到C++宿主对象的引用
const calculator = NativeModules.Calculator;
const result = calculator.add(3, 5); // 同步！无序列化
console.log(result); // 8
```

这段代码之所以"同步"，是因为JSI让JS引擎直接调用了C++内存中的函数指针，不再经过Bridge的消息队列。这是新架构最核心的突破。

## 核心知识点拆解

### 1. JSI的设计思想：从"消息传递"到"对象绑定"

旧架构中，Bridge是一个抽象层，JS无法感知原生对象的存在——它只能发送一个JSON格式的"请求"，等待原生侧处理完后通过回调返回结果。这就像两个不同国家的人通过翻译官传话，每句话都要翻译一遍。

JSI的思路是：让JS引擎能直接"看到"C++对象。JSI定义了一个宿主对象（HostObject）的C++接口，C++侧可以实现这个接口，然后JS引擎通过JSI获取到这个对象的引用，直接调用上面的方法。从JS引擎的视角看，这个C++宿主对象和JS的普通对象看起来没有区别——属性访问和方法调用都是同步的。

JSI的接口设计非常精炼，核心只有几个类：
- `Runtime`：代表一个JS引擎实例的抽象
- `Value`：JS值的抽象（number/string/object/undefined等）
- `Object`：JS对象的抽象
- `HostObject`：C++宿主对象的基类，需要实现`get`、`set`、`getPropertyNames`方法
- `Function`：JS函数的抽象，可以通过`createFromHostFunction`创建C++实现的函数

### 2. Fabric渲染器的流水线架构

Fabric将渲染过程拆分为五个阶段：

**Stage 1 - 创建React元素树（JS线程）：** React在JS线程中运行diff算法，生成新的React Element Tree。

**Stage 2 - 构建Shadow Tree（Shadow线程）：** 与旧架构不同，Fabric拥有独立的Shadow线程来处理布局计算。C++实现的Shadow Tree节点会进行Yoga布局计算，生成每个组件的布局信息（x、y、width、height）。

**Stage 3 - 树对比与原子提交：** Fabric会对新旧两棵Shadow Tree做差量对比（类似于React的Reconciliation），然后将变更以原子操作的形式打包提交。

**Stage 4 - 原生视图挂载（UI线程）：** 原生侧收到Shadow Tree的变更后，在UI线程中创建或更新实际的Native View。由于Fabric保证了渲染操作的原子性，UI线程可以一次性处理一批变更，减少重排和重绘次数。

**Stage 5 - 屏幕绘制：** 最终由Android的View系统或iOS的UIKit完成屏幕绘制。

这种架构的关键在于：Shadow线程承担了布局计算，JS线程专注于状态管理和虚拟DOM diff，UI线程只负责最终的视图提交。三线程并行工作，互不阻塞。

### 3. JSI与Fabric的协同

Fabric实际的渲染指令是通过JSI传递的。具体来说，Fabric在C++层实现了一个`Scheduler`对象，它拥有对Shadow Tree的控制权。当JS线程需要触发渲染时，通过JSI调用`Scheduler`对象上的方法（如`scheduleTransaction`），这是一个同步调用。

```cpp
// Fabric中JSI调用的简化示意
// 在JS侧通过JSI获取到Scheduler引用后：
Runtime& runtime = ...;
Object scheduler = ...; // 从JSI获取的Scheduler宿主对象
Function scheduleFn = scheduler.getPropertyAsFunction(runtime, "scheduleTransaction");

// 调用scheduleTransaction，同步触发
Value args[1] = {Value(runtime, newTransaction)};
scheduleFn.call(runtime, args, 1);
```

## 实战案例：用Turbo Modules实现图片缓存模块

假设我们需要实现一个支持磁盘缓存和内存缓存的自定义图片加载模块。在旧架构中需要走Bridge，但现在可以用JSI直接实现。

**第一步：定义C++宿主对象**

```cpp
#include <jsi/jsi.h>
#include <unordered_map>
#include <string>

using namespace facebook::jsi;

class ImageCacheHostObject : public HostObject {
private:
  std::unordered_map<std::string, std::string> _memoryCache;
  
public:
  std::vector<PropNameID> getPropertyNames(Runtime& rt) override {
    return {
      PropNameID::forUtf8(rt, "prefetch"),
      PropNameID::forUtf8(rt, "getFromCache"),
      PropNameID::forUtf8(rt, "clearCache"),
    };
  }

  Value get(Runtime& rt, const PropNameID& name) override {
    auto prop = name.utf8(rt);
    
    if (prop == "prefetch") {
      return Function::createFromHostFunction(
        rt, name, 2,
        [this](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {
          if (count < 2) throw JSError(rt, "prefetch(url, priority) requires 2 args");
          std::string url = args[0].asString(rt).utf8(rt);
          double priority = args[1].asNumber();
          // 这里调用原生图片下载逻辑（伪代码）
          _memoryCache[url] = "cached_data_for_" + url;
          return Value(true);
        });
    }
    
    if (prop == "getFromCache") {
      return Function::createFromHostFunction(
        rt, name, 1,
        [this](Runtime& rt, const Value&, const Value* args, size_t count) -> Value {
          std::string url = args[0].asString(rt).utf8(rt);
          auto it = _memoryCache.find(url);
          if (it != _memoryCache.end()) {
            return Value(String::createFromUtf8(rt, it->second));
          }
          return Value::undefined();
        });
    }
    
    return Value::undefined();
  }
};
```

**第二步：在JS侧使用**

```javascript
// ImageCacheProvider.tsx
import { NativeModules } from 'react-native';

const { ImageCacheTurboModule } = NativeModules;

// 预加载图片（同步！）
export function prefetchImage(url: string, priority: number = 1): boolean {
  return ImageCacheTurboModule.prefetch(url, priority);
}

// 同步获取缓存
export function getCachedImage(url: string): string | undefined {
  return ImageCacheTurboModule.getFromCache(url);
}
```

## 底层原理：JSI的C++层面分析

JSI的核心在于`Runtime`类的设计。不同的JS引擎（Hermes、JavaScriptCore、V8）各自实现`Runtime`接口。以Hermes为例，`HermesRuntime`继承自`Runtime`，底层封装了Hermes的JS环境。

```cpp
// JSI Runtime 核心接口（简化版）
class Runtime {
public:
  // 创建一个新的对象
  virtual Object createObject() = 0;
  
  // 创建一个带宿主对象的"特殊对象"
  virtual Object createHostObject(std::shared_ptr<HostObject> ho) = 0;
  
  // 获取宿主对象（从宿主对象的get方法获取属性）
  virtual std::shared_ptr<HostObject> getHostObject(const Object& obj) = 0;
  
  // 调用一个函数
  virtual Value call(const Function& func, const Value* args, size_t count) = 0;
  
  // 字符串与C++字符串的转换
  virtual String createStringFromUtf8(const uint8_t* utf8, size_t length) = 0;
};
```

当你在JS中执行`NativeModules.Calculator.add(3, 5)`时，底层发生的事情是：

1. JS引擎从全局对象中查找`NativeModules`
2. `Calculator`在初始化时通过JSI注册了一个`HostObject`
3. JS引擎访问`add`属性，触发`HostObject::get`方法
4. `HostObject::get`返回一个由`createFromHostFunction`创建的`Function`对象
5. JS引擎直接调用这个`Function`，底层执行C++ lambda

这个链条中**没有任何JSON序列化**，也没有消息队列排队——JS引擎直接操作C++函数指针。这就是所谓的"零成本抽象"。

## 高频面试题解析

### 面试题1：JSI相比Bridge，解决了哪些核心问题？仍然存在什么挑战？

**解析：** JSI解决的核心问题有三个。第一，**通信延迟**：Bridge的序列化开销在每次交互中约0.5-2ms，JSI消除了这个过程。第二，**异步不确定性**：Bridge的异步消息队列导致无法在函数调用后立即获取结果，必须借助回调或Promise，增加了代码复杂度。第三，**内存效率**：Bridge在传递大数据时需要在JS和原生侧各维护一份JSON字符串的副本，JSI通过指针直接访问避免了冗余拷贝。

仍然存在的挑战包括：线程安全问题（JSI同步调用可能导致C++侧的多线程竞态条件）、JSI接口本身不支持异步（需要额外封装）、以及Hermes引擎在iOS上的兼容性限制。

### 面试题2：Fabric渲染器如何避免UI线程的频繁重绘？

**解析：** Fabric的关键设计是"原子提交"（Atomic Commit）。传统方式中，每次状态变更都会触发一次独立的视图更新，UI线程频繁地进行重排和重绘。Fabric将一轮渲染周期内的所有变更收集到Shadow Tree中，经过完整的diff计算后一次性打包提交到原生侧。UI线程只需要处理一批连续的视图变更，减少了重排次数。此外，Shadow Tree的布局计算运行在独立的Shadow线程上，不会占用UI线程的时间。

### 面试题3：如果项目从旧Bridge架构迁移到新架构，最容易踩的坑是什么？

**解析：** 最常见的坑是**同步调用的心智模型变化**。旧架构中所有Native调用都是异步的，开发者习惯用`await`或者回调来获取结果。迁移到Turbo Modules后，部分JSI绑定方法是同步的，如果开发者在JS层面阻塞主线程去等待一个耗时操作，会导致严重的掉帧。另一个常见问题是**模块懒加载**：Turbo Modules默认是懒加载的，首次访问时可能有一个微小的初始化延迟，需要在启动阶段做预热。

## 总结与扩展

JSI和Fabric的引入标志着React Native从"用JS写原生应用"的阶段进入了"JS和原生无感融合"的阶段。JSI作为连接层让两种语言的内存空间可以直接交互，Fabric则通过流水线化的渲染架构大幅提升了渲染效率。

展望未来，随着**React Native Fabric对macOS、Windows等更多平台的支持**不断加深，以及**静态Hermes字节码**与JSI的深度整合，React Native的启动性能和运行时性能将进一步逼近甚至部分超越纯原生开发。对于团队而言，熟练掌握新架构的开发范式（Turbo Modules + Fabric + Codegen）将成为React Native开发的核心竞争力。
