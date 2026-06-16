---
title: Activity 与 Fragment 生命周期——跨平台开发者的生命周期管理必修课
date: 2026-09-27
categories: ["原生扩展", "Android"]
tags: ["原生扩展", "Android", "Activity", "Fragment", "生命周期", "屏幕旋转", "内存泄漏", "插件开发"]
description: 系统讲解 Android Activity 与 Fragment 的生命周期机制、屏幕旋转时重建、状态保存与恢复，以及跨平台集成中常见的生命周期相关问题。
---

## 一句话概括

Activity 和 Fragment 的生命周期是 Android 应用运行的基础机制，它定义了组件从创建到销毁的完整过程，跨平台开发者理解它就能避免内存泄漏、上下文丢失等棘手的运行时问题。

## 背景与意义

### 为什么生命周期如此重要

生命周期是 Android 操作系统管理应用资源的核心机制。与桌面应用不同，移动设备资源有限，系统需要在不同场景下灵活分配和回收资源。

举个最常见的例子：你正在使用某个 App，突然手机来了电话。通话结束后，你希望 App 回到刚才的页面而不是从头开始。这个"回来"的过程，就是 Activity 生命周期管理的功劳。

对于跨平台开发者来说，理解生命周期尤为重要，原因有三：

1. **插件执行上下文依赖**：Flutter/RN 插件中的原生代码常常依赖 Activity 实例。如果 Activity 被销毁了但插件还在运行，就会触发空指针异常。
2. **资源释放时机**：相机、传感器、定位监听等资源必须在合适的生命周期回点释放，否则会导致严重的内存泄漏。
3. **用户体验连续性**：屏幕旋转、应用切换、配置变更等操作都会触发生命周期变化。处理不好，用户会频繁遇到"应用重新启动了"的糟糕体验。

### Flutter 与 RN 中的生命周期映射

Flutter 的 `WidgetsBindingObserver` 和 RN 的 `AppState` 实际上都是对 Android Activity 生命周期的上层抽象：

**Flutter 映射**：
```
AppLifecycleState.resumed → Activity.onResume()
AppLifecycleState.paused  → Activity.onPause()
AppLifecycleState.inactive → Activity 可见但不可交互
AppLifecycleState.detached → Activity.onDestroy()
```

**RN 映射**：
```
AppState.active     → Activity.onResume()
AppState.background → Activity.onPause() / onStop()
AppState.inactive   → 从其他 Activity 返回但未完全恢复
```

当你在 Flutter 中监听 `AppLifecycleState` 发生变化时，底层实际上是 Activity 的生命周期方法在驱动。理解原生生命周期有助于定位"为什么我的 Flutter 插件在应用切到后台再回来时崩溃了"这类问题。

### Activity vs Fragment

Activity 是 Android 应用的"页面容器"，而 Fragment 是 Activity 内部的可复用"组件模块"。

| 特性 | Activity | Fragment |
|------|----------|----------|
| 独立性 | 独立运行 | 依赖 Activity |
| 生命周期 | 系统完全管理 | 受宿主 Activity 影响 |
| 复用性 | 不能复用 | 可以嵌套和复用在不同的 Activity 中 |
| 销毁重建 | 系统可随时销毁 | 跟随 Activity 或 FragmentManager |
| UI 管理 | 独立 UI | 可以拥有独立 UI |

在 Flutter/RN 项目中，通常只有一个 Activity（`MainActivity`），所有页面切换都在框架层完成。而第三方插件可能使用 Fragment 来实现特定功能（如 Google Maps）。

## 核心知识点拆解

### Activity 的 7 个生命周期方法

Activity 包含 7 个关键生命周期方法，按执行顺序排列：

```
应用启动：onCreate() → onStart() → onResume()
应用进入后台：onPause() → onStop()
应用回到前台：onRestart() → onStart() → onResume()
应用销毁：onPause() → onStop() → onDestroy()
```

#### 1. onCreate()

Activity 创建时调用，仅执行一次。

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    
    // 在这里进行初始化
    // - 初始化 UI 组件
    // - 绑定 ViewModel
    // - 恢复之前保存的状态（从 savedInstanceState Bundle 中读取）
    // - 获取传人的 Intent 参数
    
    if (savedInstanceState != null) {
        // 从保存的状态中恢复
        val count = savedInstanceState.getInt("counter", 0)
        counterView.text = count.toString()
    }
}
```

**跨平台重点**：
- Flutter 的 `MainActivity` 中的 `onCreate` 方法负责初始化 Flutter 引擎
- 如果在 `onCreate` 之前调用了 Flutter 引擎相关的方法（如通过 MethodChannel 发送消息），会因为引擎未初始化而崩溃
- `savedInstanceState` 是保存的状态数据，`null` 表示首次启动，非 `null` 表示重建

#### 2. onStart()

Activity 即将可见时调用。此时 Activity 已经创建完成，UI 即将展示给用户。

```kotlin
override fun onStart() {
    super.onStart()
    // 申请权限、绑定服务、注册广播接收器等
    // 这些操作应该在 Activity 可见后执行
}
```

#### 3. onResume()

Activity 获得焦点，可以接收用户输入时调用。

```kotlin
override fun onResume() {
    super.onResume()
    // 开始动画、传感器监听、位置更新等
    // Activity 在此状态下可以正常交互
}
```

**跨平台场景**：
- Flutter 的 `AppLifecycleState.resumed` 对应这个方法
- 在插件中，如果你需要在应用回到前台时重新加载某些数据，应该在 `onResume` 中处理
- 注意不要在 `onResume` 中执行耗时操作，因为这会延迟用户界面的展示

#### 4. onPause()

Activity 失去焦点但部分可见时调用。这是 Activity 即将进入后台的信号。

```kotlin
override fun onPause() {
    super.onPause()
    // 暂停不应该继续的操作
    // - 暂停动画
    // - 释放不需要的传感器
    // - 保存未保存的数据（但不推荐在 onPause 做繁重的 IO 操作）
    // - 取消正在进行的网络请求
}
```

**重要**：`onPause` 的执行时间非常短（Android 系统给它几百毫秒的时间），不能在这里做耗时操作。如果操作超出时间限制，系统可能直接终止活动进程。

#### 5. onStop()

Activity 完全不可见时调用。新的 Activity 已经覆盖了当前 Activity。

```kotlin
override fun onStop() {
    super.onStop()
    // 释放更多资源
    // - 停止后台服务
    // - 取消 BroadcastReceiver
    // - 停止位置更新
    // - 释放相机、MediaPlayer 等重型资源
}
```

与 Flutter 的对应：`AppLifecycleState.paused` 或 `AppLifecycleState.inactive`。

#### 6. onDestroy()

Activity 即将销毁时调用。这是 Activity 生命周期的最后一个回调（除了 `onRestart` 路径）。

```kotlin
override fun onDestroy() {
    super.onDestroy()
    // - 清理所有需要释放的资源
    // - 取消协程 Job
    // - 解除绑定 Service
    // - 清理可能存在的循环引用
    // - 注意：如果 Activity 调用 finish() 销毁，或系统为了回收内存而销毁
}
```

#### 7. onRestart()

Activity 从停止状态重新启动时调用（`onStop` → `onRestart` → `onStart` → `onResume`）。

```kotlin
override fun onRestart() {
    super.onRestart()
    // 重新加载之前释放的资源
    // 更新 UI 状态（因为数据可能在后台被改变了）
}
```

### 屏幕旋转时的生命周期重建

屏幕旋转是 Activity 生命周期中最具"戏剧性"的场景——它会让 Activity 从头到尾走一遍完整的销毁重建流程。

```
旋转前：onPause() → onStop() → onDestroy()
旋转后：onCreate() → onStart() → onResume()
```

#### 为什么屏幕旋转会重建 Activity

Android 的默认行为是：配置变更（Configuration Change）会导致当前 Activity 被销毁并重建。屏幕旋转触发了 `orientation` 和 `screenSize` 两种配置的变更，所以 Activity 被重建了。

这意味着：
1. Activity 中的所有实例变量丢失
2. UI 状态（输入框文本、滚动位置等）消失
3. 所有正在进行的工作（网络请求、定时器）中断

#### 应对策略一：onSaveInstanceState 恢复简单状态

```kotlin
// 保存状态
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putString("username", usernameInput.text.toString())
    outState.putInt("counter", counter)
}

// 恢复状态（在 onCreate 中）
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)
    
    if (savedInstanceState != null) {
        val savedUsername = savedInstanceState.getString("username", "")
        usernameInput.setText(savedUsername)
        counter = savedInstanceState.getInt("counter", 0)
    }
}
```

**限制**：
- `Bundle` 适合保存简单类型（String、int、boolean、Parcelable、Serializable）
- 不适合保存大量数据（受 IPC 大小限制，通常 500KB-1MB）
- 不适合保存 Bitmap、大数据列表

#### 应对策略二：ViewModel 处理复杂数据

ViewModel 是 Android Jetpack 提供的生命周期感知组件，它在 Activity 重建时保持存活。

```kotlin
class MyViewModel : ViewModel() {
    private val _userList = MutableLiveData<List<User>>()
    val userList: LiveData<List<User>> = _userList
    
    private val repository = UserRepository()
    
    fun loadUsers() {
        viewModelScope.launch {
            _userList.value = repository.fetchUsers()
        }
    }
}

// 在 Activity 中使用
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        // 无论 Activity 重建多少次，ViewModel 始终是同一个实例
        viewModel.loadUsers()
        viewModel.userList.observe(this) { users ->
            // 更新 UI
        }
    }
}
```

**ViewModel 生命周期**：ViewModel 在关联的 Activity 真正 finish（不是重建）或 Fragment detach 时才被清除。这意味着屏幕旋转时数据完好无损。

#### 应对策略三：自己处理配置变更（不推荐）

```xml
<activity
    android:name=".MainActivity"
    android:configChanges="orientation|screenSize|screenLayout"
    android:label="@string/app_name" />
```

这样设置后，屏幕旋转时不会重建 Activity，但 Activity 需要自行处理配置变更后的布局调整。**不推荐**这种方法，因为它破坏了 Android 标准生命周期，容易导致不可预期的行为。

#### 跨平台场景中的屏幕旋转

在 Flutter 中：
- Flutter Engine 自动处理屏幕旋转，开发者不需要关心
- `MediaQuery.of(context).orientation` 可以获取当前方向
- `SystemChrome.setPreferredOrientations()` 可以锁定方向

在 RN 中：
- 需要 `Dimensions` API 或 `useWindowDimensions` hook 监听方向变化
- `react-native-orientation-locker` 插件提供了方向控制功能

问题是：**如果你的插件在原生层使用了持久的 Context 或 View 引用**，屏幕旋转导致的 Activity 重建会让这些引用失效，从而导致插件崩溃。

### Fragment 生命周期

Fragment 的生命周期比 Activity 更复杂，因为它是在 Activity 内部运行的可管理组件。

```
Fragment 完整生命周期：
onAttach() → onCreate() → onCreateView() → onViewCreated() → onStart() → onResume()
→ onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
```

#### Fragment 生命周期方法详解

| 方法 | 触发时机 | 用途 |
|------|---------|------|
| `onAttach()` | Fragment 与 Activity 关联时 | 获取 Activity 引用 |
| `onCreate()` | Fragment 初始化时 | 初始化非 UI 相关数据 |
| `onCreateView()` | 创建 Fragment 的 UI 布局 | 返回 View 对象 |
| `onViewCreated()` | 视图创建完成后 | 初始化 UI 组件、设置监听器 |
| `onStart()` | Fragment 可见时 | 开始交互前准备 |
| `onResume()` | Fragment 获得焦点时 | 开始动画、传感器 |
| `onPause()` | Fragment 失去焦点时 | 暂停动画 |
| `onStop()` | Fragment 不可见时 | 释放资源 |
| `onDestroyView()` | Fragment 的视图被销毁时 | 清理视图相关引用 |
| `onDestroy()` | Fragment 被销毁时 | 清理所有资源 |
| `onDetach()` | Fragment 与 Activity 解除关联时 | 释放 Activity 引用 |

#### Fragment 生命周期与 Activity 的对应关系

```
Activity状态         Fragment回调
                    ↓
onCreate()  ───→    onAttach()
                    onCreate()
                    onCreateView()
                    onViewCreated()
                    ↓
onStart()   ───→    onStart()
                    ↓
onResume()  ───→    onResume()
                    ↓
onPause()   ───→    onPause()
                    ↓
onStop()    ───→    onStop()
                    ↓
onDestroy() ───→    onDestroyView()
                    onDestroy()
                    onDetach()
```

**关键理解**：
- Fragment 的 `onCreate` 和 Activity 的 `onCreate` 都在创建阶段执行，但 Activity 的 `onCreate` 先执行
- Fragment 的视图生命周期（`onCreateView` ~ `onDestroyView`）与 Activity 完全绑定的
- 如果 Activity 被销毁重建，Fragment 也会被销毁重建

#### 跨平台中的 Fragment

Fragment 在跨平台插件中常见于：
1. **Google Maps**：使用 `SupportMapFragment` 来显示地图
2. **相机预览**：部分相机插件使用 `CameraX` 的 `PreviewView` 搭配 Fragment
3. **复杂对话框**：使用 `DialogFragment` 展示原生弹窗

**Fragment 的隐藏陷阱**：`getContext()` 可能返回 `null`。因为 Fragment 可能在 `onDetach` 后被持有引用，但 Context 已经变为 `null`。在 Fragment 中调用方法时**永远要先检查 `getContext() != null`**。

### 生命周期状态保存的完整方案

真正可靠的跨平台应用，需要组合使用多种状态保存机制：

```
状态保存层次结构（从易失到持久）：
1. onSaveInstanceState Bundle ── 最轻量，适合简单状态
2. ViewModel ── 适合 UI 相关数据和业务逻辑
3. SavedStateHandle ── ViewModel 的扩展，通过 Bundle 持久化
4. Room / DataStore ── 持久化存储，适合需要长期保存的数据
```

#### SavedStateHandle 示例

```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    val counter = savedStateHandle.getLiveData<Int>("counter", 0)
    
    fun increment() {
        counter.value = (counter.value ?: 0) + 1
    }
}
```

SavedStateHandle 结合了 Bundle 的自动保存和 ViewModel 的存活特性——在进程被系统杀死后也能恢复数据。

### 生命周期感知组件

Android Jetpack 提供了 `Lifecycle` 库，允许组件感知生命周期变化。

```kotlin
class MyLocationProvider(
    private val lifecycle: Lifecycle
) : LifecycleObserver {
    
    private var locationManager: LocationManager? = null
    private var isListening = false
    
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun startLocationUpdates() {
        if (!isListening) {
            locationManager?.requestLocationUpdates(...)
            isListening = true
        }
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun stopLocationUpdates() {
        if (isListening) {
            locationManager?.removeUpdates(this)
            isListening = false
        }
    }
}

// 在 Activity 中使用
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val provider = MyLocationProvider(lifecycle)
        lifecycle.addObserver(provider)
    }
}
```

**跨平台价值**：如果你在开发 Flutter/RN 插件，使用生命周期感知组件可以避免手动管理资源释放，大幅度削减内存泄漏问题。

## 实战案例

### 案例一：Flutter 插件中的 Camera 内存泄漏

**场景**：在 Flutter 插件中使用了 Android CameraX API 进行相机预览，但在页面切换后，相机资源没有释放。

**问题代码**：
```kotlin
class CameraPlugin : FlutterPlugin, MethodCallHandler, LifecycleObserver {
    private var activity: Activity? = null
    private var cameraProvider: ProcessCameraProvider? = null
    
    override fun onMethodCall(call: MethodCall, result: Result) {
        if (call.method == "startCamera") {
            startCamera()
        }
    }
    
    private fun startCamera() {
        // 创建 Preview 并绑定 Lifecycle
        val preview = Preview.Builder().build()
        val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA
        cameraProvider?.bindToLifecycle(
            activity as LifecycleOwner,
            cameraSelector,
            preview
        )
        // ❌ 问题：没有在 Activity 销毁时 unbind 相机
        // CameraX 会持续持有相机资源
    }
}
```

**修复方案**：
```kotlin
class CameraPlugin : FlutterPlugin, MethodCallHandler, LifecycleObserver {
    private var activity: Activity? = null
    private var cameraProvider: ProcessCameraProvider? = null
    private var camera: Camera? = null
    
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun onDestroy() {
        // 释放相机资源
        cameraProvider?.unbindAll()
        camera = null
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun onPause() {
        // 暂停相机预览
        camera?.cameraInfo?.displayState?.let { state ->
            if (state.value == CameraInfo.DISPLAY_STATE_DISABLED) {
                // 页面不可见时释放资源
            }
        }
    }
}
```

### 案例二：RN 原生模块中的 Context 丢失

**场景**：React Native 原生模块使用了 GlobalContext，但在 Activity 重建后崩溃。

**问题代码**：
```java
public class MyModule extends ReactContextBaseJavaModule {
    private static Context globalContext;
    
    public MyModule(ReactApplicationContext reactContext) {
        super(reactContext);
        // ❌ 将 Context 保存为静态变量
        globalContext = reactContext;
    }
    
    public void doSomething() {
        // ❌ 当 Activity 重建后，globalContext 指向已销毁的旧实例
        // 访问视图或资源时可能崩溃
        Toast.makeText(globalContext, "Hello", Toast.LENGTH_SHORT).show();
    }
}
```

**修复方案**：
```java
public class MyModule extends ReactContextBaseJavaModule {
    // ✓ 使用弱引用避免持有已销毁的 Context
    private WeakReference<ReactApplicationContext> weakContext;
    
    public MyModule(ReactApplicationContext reactContext) {
        super(reactContext);
        weakContext = new WeakReference<>(reactContext);
    }
    
    public void doSomething() {
        ReactApplicationContext context = weakContext.get();
        if (context != null && !context.isDestroyed()) {
            Toast.makeText(context, "Hello", Toast.LENGTH_SHORT).show();
        }
    }
}
```

**最佳实践**：在原生模块中，永远不要将 Context 保存为静态变量。始终使用 `getCurrentActivity()` 或 `getApplicationContext()` 获取可靠的 Context 引用。

### 案例三：屏幕旋转导致的网络请求中断

**场景**：Flutter 应用在竖屏状态下发起了一个网络请求，用户旋转屏幕导致请求回调中的 Context 无效。

**Flutter 端解决方案**（使用 `AutomaticKeepAliveClientMixin` 或 `PageStorage`）：

```dart
class MyPage extends StatefulWidget {
  @override
  State<MyPage> createState() => _MyPageState();
}

class _MyPageState extends State<MyPage> with AutomaticKeepAliveClientMixin {
  String? data;
  
  @override
  void initState() {
    super.initState();
    loadData();
  }
  
  Future<void> loadData() async {
    final result = await fetchDataFromApi();
    if (mounted) {
      // ✓ 检查 mounted 状态，确保 Widget 仍然在树中
      setState(() {
        data = result;
      });
    }
  }
  
  @override
  bool get wantKeepAlive => true;
}
```

但最好的做法是使用状态管理库（Riverpod、Provider、Bloc）将数据加载逻辑与 Widget 生命周期解耦。

## 常见问题

### Q1: 应用切到后台再回来时黑屏

**原因**：Activity 被系统销毁释放了内存，回到前台时重建时出现了问题。

**排查**：
1. 检查 `onSaveInstanceState` 是否保存了足够的状态
2. 检查 `onCreate` 中是否正确处理了 `savedInstanceState == null` 和 `!= null` 的情况
3. 确认没有在 `onCreate` 之前调用引擎相关功能

### Q2: "Can't create handler inside thread that has not called Looper.prepare"

**原因**：在后台线程中尝试创建 Handler，但该线程没有 Looper。

**常见场景**：在插件中执行了异步操作并尝试更新 UI。

**解决方案**：
```kotlin
// 确保所有 UI 操作在主线程执行
runOnUiThread {
    // 更新 UI
}

// 或使用 Handler(Looper.getMainLooper())
Handler(Looper.getMainLooper()).post {
    // 在主线程执行
}
```

### Q3: 内存泄漏——Activity 无法被 GC 回收

**原因**：Activity 被某个长生命周期对象持有引用，导致无法回收。

**常见泄漏场景**：
1. 静态变量持有了 Activity 实例
2. 内部类（匿名内部类）隐式持有外部类的引用
3. 未取消注册的 BroadcastReceiver
4. 未停止的 AsyncTask

**检测工具**：
- Android Studio 的 Memory Profiler
- LeakCanary（最常用的内存泄漏检测库）
- 手动通过 `adb` 命令查看内存分配

### Q4: Fragment 的 getContext() 返回 null

**原因**：Fragment 已经被 detach，但代码仍然尝试访问 Context。

**解决方案**：
```java
// 永远在使用 Context 前检查
if (getContext() != null && isAdded()) {
    // 安全操作
}

// 或使用 requireContext()（如果 Fragment 非 detach 状态）
// 但 requireContext 会在 Context 为 null 时抛出异常
```

## 总结

Activity 与 Fragment 的生命周期是 Android 开发中最基础也是最重要的概念。对于跨平台开发者，理解生命周期能帮助你：

1. **编写健壮的插件**：正确地初始化和释放资源，避免内存泄漏
2. **确保状态连续性**：屏幕旋转、应用切换后的状态恢复
3. **提供流畅的用户体验**：不再发生"应用切回来就崩溃"的糟糕体验
4. **高效定位问题**：知道崩溃发生在哪个生命周期阶段，快速定位原因

生命周期管理的核心原则可以总结为：
- **在 onResume 中注册，在 onPause 中注销**：适用于广播、传感器、位置监听
- **在 onCreate 中初始化，在 onDestroy 中清理**：适用于数据库、文件句柄、重型对象
- **始终假设 Activity 可能被销毁**：不要保存 Activity 引用在静态变量中
- **使用生命周期感知组件**：Jetpack 的 Lifecycle/LiveData/ViewModel 是现代 Android 开发的标准方式

在 Flutter 和 RN 框架中，大多数生命周期管理已经被框架封装，但当你的插件直接调用原生 API 时，这些原生层面的生命周期知识就成了区分你的插件是"偶尔崩溃"还是"稳定运行"的关键分水岭。
