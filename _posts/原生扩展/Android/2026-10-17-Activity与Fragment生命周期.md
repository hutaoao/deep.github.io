---
layout: post
title: "Activity 与 Fragment 生命周期——跨平台开发者的生命周期管理必修课"
date: 2026-10-17
categories: ["原生扩展", "Android"]
tags: ["Activity", "Fragment", "生命周期", "ViewModel", "状态保存", "屏幕旋转", "内存泄漏"]
description: >
  Activity/Fragment 的生命周期是系统管理资源的节拍器：创建→可见→前台→后台→销毁；
  屏幕旋转会重建、进程会被回收，跨平台插件的相机/定位/上下文丢失，根因几乎都在生命周期错配。
---

## 一句话概括

生命周期就是 Android 系统"管理组件生死"的一套回调：**创建（onCreate）→ 可见（onStart）→ 前台可交互（onResume）→ 失去焦点（onPause）→ 不可见（onStop）→ 销毁（onDestroy）**。Fragment 在这套之上多了一层"视图"的创建与销毁。

对跨平台开发者，它直接关系到**内存泄漏、上下文丢失、屏幕旋转白屏、切后台回来崩溃**这些最棘手的运行时问题。Flutter 的 `WidgetsBindingObserver`、RN 的 `AppState` 本质都是对这套原生生命周期的上层封装——插件一旦直接碰原生 API，生命周期知识就是"偶发崩溃"和"稳定运行"的分水岭。

一句话结论：**资源在 onResume 注册、onPause 注销；数据用 ViewModel 扛旋转；永远别假设 Activity 不会死。**

## 核心知识点

### 1. Activity 生命周期：七个回调

```text
启动：    onCreate → onStart → onResume
退后台：  onPause → onStop
回前台：  onRestart → onStart → onResume
销毁：    onPause → onStop → onDestroy
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // ✅ 只做一次性初始化：绑定 ViewModel、读 Intent、恢复状态
        // ❌ 别做耗时 IO、别持有长生命周期对象
    }
    override fun onStart() { super.onStart() }      // 即将可见
    override fun onResume() {                        // 拿到焦点、可交互
        super.onResume()
        // ✅ 开动画、传感器、定位监听、相机预览
    }
    override fun onPause() {                         // 失去焦点（几百 ms 内必须返回）
        super.onPause()
        // ✅ 暂停动画、停传感器；⚠️ 不能做耗时操作
    }
    override fun onStop() { super.onStop() }         // 完全不可见：释放重型资源
    override fun onDestroy() {                       // 真正销毁：解绑、取消协程
        super.onDestroy()
    }
}
```

关键：**`onPause` 执行时间极短**，系统只给几百毫秒，耗时操作会卡住页面切换甚至被系统杀掉。

### 2. Fragment 生命周期：多一层"视图"生命周期

Fragment 嵌在 Activity 内，除了自身生命周期，还有独立的**视图**生命周期：

```text
onAttach → onCreate → onCreateView → onViewCreated
→ onStart → onResume → onPause → onStop
→ onDestroyView → onDestroy → onDetach
```

| 回调 | 时机 | 该做什么 |
|---|---|---|
| `onAttach` | 与 Activity 关联 | 拿 Activity 引用（尽量延后） |
| `onCreate` | 初始化（非 UI 数据） | ViewModel 绑定 |
| `onCreateView` | 创建 UI 返回 View | 只负责 inflate 布局 |
| `onViewCreated` | 视图已建好 | ✅ 找控件、设监听、开协程 |
| `onDestroyView` | 视图被销毁 | 解绑 View 相关引用，防泄漏 |
| `onDetach` | 与 Activity 解除 | 释放 Activity 引用 |

⚠️ 易错点：**`onCreateView`~`onDestroyView` 会在旋转时走一遍，但 `onDestroy`~`onAttach` 不一定**。所以"视图相关引用"在 `onDestroyView` 清，"全局资源"在 `onDestroy` 清，别混。

### 3. 屏幕旋转：整页销毁重建

旋转触发 `orientation` + `screenSize` 配置变更，系统**默认销毁并重建 Activity**（Fragment 跟着重建）：

```text
旋转前：onPause → onStop → onDestroy
旋转后：onCreate → onStart → onResume
```

三个应对层次：

```kotlin
// ① 简单状态：onSaveInstanceState（Bundle，有大小上限）
override fun onSaveInstanceState(out: Bundle) {
    super.onSaveInstanceState(out)
    out.putInt("counter", counter)
}
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    counter = savedInstanceState?.getInt("counter", 0) ?: 0
}

// ② 复杂/异步数据：ViewModel（旋转时实例存活，不重请求）
class MainViewModel : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    fun load() = viewModelScope.launch { _users.value = repo.fetch() }
}
// Activity 里：无论重建几次，拿到的是同一个 ViewModel
private val vm: MainViewModel by viewModels()
```

| 方案 | 适用 | 抗旋转 | 抗进程被杀 |
|---|---|---|---|
| `onSaveInstanceState` | 简单字段 | ✅ | ✅ |
| `ViewModel` | UI 数据/业务逻辑 | ✅ | ❌ |
| `SavedStateHandle` | 需持久的小状态 | ✅ | ✅ |
| `Room`/`DataStore` | 长期数据 | ✅ | ✅ |

```kotlin
// ③ 不推荐：自行处理配置变更（破坏标准生命周期，易出不可预期行为）
// android:configChanges="orientation|screenSize"
```

### 4. ViewModel + SavedStateHandle

`ViewModel` 在 Activity `finish`（真正退出）或进程被杀时才清除，**旋转不清除**。需要"进程被杀也能恢复"的小状态，用 `SavedStateHandle`：

```kotlin
class CounterViewModel(private val saved: SavedStateHandle) : ViewModel() {
    val count = saved.getStateFlow("count", 0)
    fun inc() = saved["count"] = count.value + 1
}
```

### 5. 生命周期感知组件：现代写法 DefaultLifecycleObserver

`@OnLifecycleEvent` 已废弃，现代用 `DefaultLifecycleObserver`：

```kotlin
class LocationWatcher : DefaultLifecycleObserver {
    override fun onResume(owner: LifecycleOwner) {
        // ✅ 页面可见时开始定位
    }
    override fun onPause(owner: LifecycleOwner) {
        // ✅ 页面不可见时停止，防泄漏
    }
}

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(LocationWatcher())
    }
}
```

跨平台插件用它能自动随页面开关资源，不用手写注册/注销，大幅减少泄漏。

### 6. Fragment 事务与返回栈

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.container, DetailFragment())
    .addToBackStack("detail")   // 让用户按返回键能回到上一个
    .commit()
```

- 在 Fragment 内用 `parentFragmentManager`（取代旧 `getFragmentManager`）；嵌套 Fragment 用 `childFragmentManager`。
- `commit()` 是异步的，`commitNow()` 才是同步（慎用，避免在生命周期回调用）。
- 旋转重建后，FragmentManager 会**自动恢复事务栈**，无需手动重加。

## 其实你每天都在用

- **相机预览（CameraX）**：`cameraProvider.bindToLifecycle(this, ...)` 把预览绑到 Lifecycle，页面切走自动停、回来自动开，不用手管。
- **屏幕旋转不白屏**：列表数据放 `ViewModel`，旋转后不重新请求网络，用户无感。
- **广播/传感器**：用 `DefaultLifecycleObserver` 在 `onResume` 注册、`onPause` 注销，避免常驻泄漏。
- **网络回调安全**：Fragment 里更新 UI 前判 `isAdded()`/`context != null`；Flutter 里判 `mounted`。
- **切后台再回来**：`onStop → onStart → onResume`，用 `onSaveInstanceState` 或 ViewModel 还原用户输入、滚动位置。
- **插件持有 Activity**：原生模块别把 Activity 存成静态变量，否则 Activity 销毁了还被引用 → 内存泄漏；用 `getCurrentActivity()` 或弱引用。
- **原生弹窗 `DialogFragment`**：它的显隐、旋转恢复都跟着 FragmentManager 走，比普通 Dialog 稳。

## 常见误解（FAQ）

**❌ 误区一："屏幕旋转只是布局重排，Activity 没重建"**

错，而且是最高频的认知错误。旋转属于配置变更，系统**默认把整个 Activity 销毁再重建**（Fragment 一并重建）。所有实例变量、输入内容、进行中的请求都会丢——除非用 `ViewModel` 或 `onSaveInstanceState` 保住。想"不重建"去设 `configChanges` 是反模式，会引入更多隐患。

**❌ 误区二："`onSaveInstanceState` 能存任意大的数据"**

`Bundle` 跨进程传递有大小上限（约 1MB，超了直接 `TransactionTooLargeException`）。它只适合存简单字段（int/String/Parcelable 小对象）。大列表、Bitmap、网络结果请交给 `ViewModel` 或 `Room`/`DataStore`，别硬塞 Bundle。

**❌ 误区三："Fragment 里 `getContext()`/`requireContext()` 永远不为 null"**

Fragment 在 `onDetach` 后 Context 可能变 null。直接用 `requireContext()` 在 detach 状态下会抛异常；稳妥写法是 `if (isAdded() && context != null)` 再操作。把 Fragment 引用长时间持有也更危险——它随时可能脱离 Activity。

**❌ 误区四："ViewModel 是永生的，进程被杀也有数据"**

`ViewModel` 只扛**配置变更（旋转）**，它不抗**进程被系统回收**。用户切走很久、内存不足时系统杀进程，ViewModel 一起没。需要"杀进程也能恢复"的小状态用 `SavedStateHandle`，大数据用持久化存储。

**❌ 误区五："把 Activity 存成静态变量，插件随时用更方便"**

这是典型内存泄漏。静态变量持有 Activity，导致它该回收时回收不掉，长时间积累 OOM。原生模块要拿 Context，用 `getApplicationContext()`（不随页面死）或 `getCurrentActivity()`（可能为空需判空），**绝不静态持有 Activity**。

## 一句话总结

生命周期是 Android "按节拍回收资源"的底层机制：Activity 七回调、Fragment 多一层视图生命周期、屏幕旋转会重建、进程会被系统杀；记住"onResume 注册 / onPause 注销、ViewModel 扛旋转、SavedStateHandle 扛被杀、绝不静态持有 Activity"，你的插件就从"偶发崩溃"跨进"稳定运行"。
