---
title: AndroidManifest.xml 完全解读——跨平台开发者必知的清单文件
date: 2026-10-13
categories: ["原生扩展", "Android"]
tags: ["原生扩展", "Android", "AndroidManifest", "清单文件", "插件开发", "权限声明"]
description: 深入解析 AndroidManifest.xml 的结构、标签含义与配置要点，帮助 Flutter/RN/鸿蒙开发者理解为什么集成本地插件时总需要修改这个文件。
---

## 一句话概括

AndroidManifest.xml 是 Android 应用的"身份证"和"说明书"，它向系统声明应用的身份、组件、权限和兼容性信息，跨平台开发者在集成本地插件时几乎必然需要修改它。

## 背景与意义

### 什么是 AndroidManifest.xml

AndroidManifest.xml 是每个 Android 应用必须包含的清单文件，位于 `app/src/main/` 目录下。它的名字"Manifest"（清单）恰如其分——这是一份向 Android 系统宣告"我是什么、我有什么、我需要什么"的声明文件。

对于跨平台开发者来说，这个文件往往是个"既熟悉又陌生"的存在。熟悉是因为在集成 Flutter 插件、React Native 原生模块或鸿蒙的 Android 兼容层时，总是被要求在 `AndroidManifest.xml` 中添加某些配置；陌生是因为很少有人系统地讲解过这个文件里每个标签到底在做什么。

### 为什么跨平台开发者需要理解它

想象一下这个场景：你在 Flutter 项目中引入了 `image_picker` 插件来选择照片，编译时一切正常，但运行时应用直接崩溃。排错半天，最后发现 `AndroidManifest.xml` 里忘了声明存储权限。或者更隐蔽的问题——你的 React Native 应用在 Android 12 以上的设备上打开相机时闪退，因为新系统要求 `<activity>` 的 `exported` 属性必须显式声明。

这些问题在跨平台开发中屡见不鲜。原因很直接：无论是 Flutter、React Native 还是鸿蒙应用，当它们调用原生 Android 功能时，最终编译成的 APK/AAB 仍然是一个标准的 Android 应用，而 Android 系统通过 `AndroidManifest.xml` 来了解这个应用的所有信息。

数据层面来看，根据 Google Play Console 的统计，超过 30% 的应用上架驳回与 `AndroidManifest.xml` 配置不当有关。其中权限声明错误和 Activity 配置不完整是最常见的两类问题。

### 文件的生成与合并

在 Flutter 或 RN 项目中，`AndroidManifest.xml` 并非只有一个。除了主模块的清单文件，每个第三方库和插件也都自带自己的清单文件。在编译时，Android Gradle Plugin（AGP）会通过"清单合并"（Manifest Merger）机制，将所有清单文件合并成一个最终文件。

这就是为什么你有时会发现最终编译出的清单文件中包含了一些你从来没有写过的内容——那是插件自动贡献进来的。

了解清单合并规则非常重要：
- 主模块的清单文件优先级最高
- 库模块的清单文件优先级次之
- 合并规则遵循"后定义者覆盖前定义者"的原则
- 如果存在冲突且无法自动解决，合并会失败并报错

这也解释了为什么跨平台开发者经常需要在主模块的 `AndroidManifest.xml` 中声明 `tools:replace="android:xxx"` 来主动覆盖插件中的某些属性。

## 核心知识点拆解

### `<manifest>`——应用的根节点

`<manifest>` 是整个清单文件的根元素，它定义了应用的基本身份信息。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.myapp"
    android:versionCode="2"
    android:versionName="1.1.0">
```

#### package（包名）

包名是应用在系统中的唯一标识符。一旦应用发布到 Google Play 后，包名就不可更改——如果改了，Google Play 会将其视为一个全新的应用。

包名在开发中的多方面作用：
- 用于生成 `R.java` 资源引用类的包名路径
- 用于构建 `BuildConfig` 类的包名路径
- 作为 Intent 路由的目标标识
- 作为应用在设备文件系统中的数据目录名（`/data/data/<package>/`）

**跨平台开发者常见问题**：Flutter 项目创建时默认使用 `com.example.appname` 作为包名，如果发布前忘记修改，可能导致后续维护时的品牌一致性问题和混淆规则配置困难。建议在项目创建之初就确定好包名。

#### versionCode 与 versionName

- `versionCode`：一个整数，用于内部版本号比较。每次发布新版本时必须递增。Google Play 以此判断是否有新版本。
- `versionName`：一个字符串，展示给用户看的版本号，如 "2.0.1"。可以任意格式，没有数值上的限制。

**最佳实践**：在 Flutter 中，如果你使用 `pubspec.yaml` 中的 `version` 字段，它会自动映射到这两个属性。很多新手直接在 `AndroidManifest.xml` 中修改版本号，结果下一次 `flutter build` 时又被覆盖回去——这是因为 Flutter 的构建脚本会从 `pubspec.yaml` 读取版本信息并注入到清单文件中。

### `<application>`——应用级别的配置

`<application>` 标签是整个应用的配置中心，它的属性影响应用全局的行为和外观。

```xml
<application
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme"
    android:supportsRtl="true"
    android:allowBackup="true"
    android:usesCleartextTraffic="false"
    android:networkSecurityConfig="@xml/network_security_config"
    android:name=".MyApplication">
```

#### 核心属性详解

**android:icon 和 android:label**
- `icon`：应用的图标资源引用。通常使用 `@mipmap/` 前缀引用 mipmap 资源目录下的图片。
- `label`：应用名，会在桌面、任务栏、应用详情页等多处显示。

**跨平台问题**：在 Flutter 中替换应用图标时，很多人去 Android 目录下手动替换图片，但 Flutter 2.0 之后推荐使用 `flutter_launcher_icons` 工具自动生成各密度版本的图标。手动替换容易漏掉某些密度规格。

**android:theme**
- 定义应用默认的主题样式。Android 使用 Material Design 主题体系，通过 style 资源文件定义。
- 常见的主题有 `Theme.AppCompat.Light.NoActionBar`、`MaterialComponents.DayNight.NoActionBar` 等。
- 没有 ActionBar 的主题（NoActionBar 系列）在现代 Android 开发中更常见，因为推荐使用 Toolbar 作为自定义顶部栏。

**跨平台实战**：RN 中的 `StatusBar` 组件和 Flutter 中的 `SystemChrome` 都是在运行时动态修改状态栏样式，但应用启动时的"闪屏"阶段，样式完全由 `android:theme` 控制。很多开发者发现闪屏阶段状态栏颜色异常，就是因为主题配置不当。

**android:supportsRtl**
- 是否支持从右到左的布局方向（阿拉伯语、希伯来语等）。
- 设为 `true` 后，系统会自动镜像布局。
- 跨平台框架基本都已内置 RTL 支持（Flutter 的 `Directionality`、RN 的 `I18nManager`），但原生层面的 RTL 需要靠这个开关启用。

**android:allowBackup**
- 是否允许应用数据自动备份到 Google Drive。
- Android 12 以上，如果目标 SDK >= 31，默认为 `true`。
- 这对跨平台应用的数据持久化有直接影响——若用户在不同设备间切换，备份机制可能导致数据冲突或过期数据恢复问题。

**android:usesCleartextTraffic**
- 是否允许明文 HTTP 流量。Android 9（API 28）开始默认禁止。
- 开发测试阶段可设为 `true`，但发布版本必须设为 `false` 或配置 `networkSecurityConfig`。

**跨平台典型问题**：很多 Flutter/RN 应用在调试 HTTP API 时一切正常，但 release 版本中网络请求统统失败，终端日志提示 `CLEARTEXT communication not permitted`。排查时才发现 `usesCleartextTraffic` 默认为 `false`，而开发时依赖的是 debug 构建自动添加的豁免。

**android:name**
- 自定义 `Application` 子类的全限定类名。这在三方库初始化时非常常见。
- 例如 `FirebaseApp`、某些推送 SDK 都需要自定义 Application 类。

**跨平台冲突问题**：当需要同时使用 Firebase 和 JPush 时，两个库可能都有各自的 Application 初始化逻辑。这时需要你自己编写一个 Application 子类，在其中手动初始化多个库，而不能在清单文件中指定多个不同的 Application。

### `<activity>`——Activity 注册

Activity 是 Android 应用的界面入口。每个 Activity 都必须在清单文件中显式注册，这是 Android 安全模型的一部分——系统只允许启动那些已声明的 Activity。

```xml
<activity
    android:name=".MainActivity"
    android:exported="true"
    android:launchMode="singleTop"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

#### android:exported
- 指定其他应用是否可以启动这个 Activity。
- 从 Android 12（API 31）开始，如果 Activity 包含 intent-filter，则 `exported` 必须显式声明，否则编译报错。
- 这是 Android 团队加强安全性的措施之一。

#### android:launchMode
标准的四种启动模式：
1. **standard**（默认）：每次启动都创建新实例。
2. **singleTop**：如果目标 Activity 在栈顶，直接复用；否则创建新实例。
3. **singleTask**：如果栈中存在该 Activity，将栈中该 Activity 之上的其他 Activity 全部出栈，复用该 Activity。
4. **singleInstance**：该 Activity 独自在一个新栈中运行。

**跨平台场景**：Flutter 的 `MainActivity` 通常设置为 `singleTop`，因为每次点击通知或外部链接时，希望能复用已有的 Flutter 引擎实例，而不是创建新的。如果你设置成了 `standard`，每次外部跳转都会启动一个新的 Flutter 引擎，不仅浪费内存，还可能导致严重的性能问题。

#### intent-filter
Intent Filter 声明了 Activity 可以响应的隐式 Intent。最常见的用途是：
- 声明哪个 Activity 是应用的入口（`MAIN` + `LAUNCHER`）
- 声明应用可以处理的链接类型（`VIEW` + 特定 URL Scheme 或 Web URL）

**深度链接（Deep Link）配置**：

```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data
        android:scheme="https"
        android:host="www.myapp.com"
        android:pathPrefix="/product" />
</intent-filter>
```

在 Flutter 中，深度链接的配置需要同时调整 `AndroidManifest.xml` 中的 intent-filter 和 Flutter 端的路由处理。RN 则依赖 `react-native-deep-link` 或 `react-navigation` 的深度链接模块。

#### windowSoftInputMode
控制软键盘弹出时界面的调整方式：
- `adjustResize`：界面大小被压缩，为键盘留出空间（适合可滚动的页面）
- `adjustPan`：界面整体上移，使焦点可见（适合固定大小的页面）

### `<service>`——后台服务注册

Service 是 Android 的四大组件之一，用于在后台执行长时间运行的操作。

```xml
<service
    android:name=".MyFirebaseMessagingService"
    android:exported="false">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT" />
    </intent-filter>
</service>
```

在跨平台应用中，Service 最常见的用途是：
- 推送消息处理（Firebase Cloud Messaging）
- 后台定位更新
- 文件下载任务

**跨平台注意事项**：
- Android 8 以上引入了后台执行限制，前台 Service 必须在通知栏显示通知。
- Android 12 引入了"精确闹钟"权限，后台任务的调度策略变得更加复杂。
- Flutter 的 `flutter_background_service` 插件实际上就是在原生层注册了一个 Service。
- RN 的 `react-native-background-fetch` 同样依赖于原生的 Service 或 JobScheduler。

### `<receiver>`——广播接收器注册

BroadcastReceiver 用于监听系统级或应用级的广播事件。

```xml
<receiver
    android:name=".BootReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

**常见应用场景**：
- 监听设备启动完成广播（`BOOT_COMPLETED`）来启动后台服务
- 监听网络状态变化
- 监听屏幕亮灭

Android 8 开始，大部分隐式广播（非定向到具体应用的广播）已经不再生效。跨平台开发者需要特别注意：如果你以为注册了某个广播接收器就能在后台收到相应事件，很可能在 Android 8+ 设备上完全不起作用。

### 权限声明——`<uses-permission>` vs `<permission>`

这是跨平台开发者最常接触的部分，也是最容易出错的地方。

#### `<uses-permission>`
向系统申请某个权限。应用本身没有权利直接使用敏感功能，必须通过 `uses-permission` 声明需要的权限。

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
```

#### `<permission>`
定义自己的权限。只有当你开发了一个提供服务的应用，且需要其他应用通过你的权限来访问时，才需要使用这个标签。

```xml
<permission
    android:name="com.example.myapp.permission.MY_CUSTOM_PERMISSION"
    android:protectionLevel="signature" />
```

#### 关键区别
- `uses-permission`：**请求**使用系统或其他应用的权限
- `permission`：**定义**自己的权限给别人使用

跨平台开发者只需要关心 `uses-permission`，几乎不会用到 `permission`。

#### `<uses-feature>`——声明硬件依赖

```xml
<uses-feature android:name="android.hardware.camera" android:required="true" />
```

这个标签告诉 Google Play：应用需要特定的硬件特性。如果 `required="true"`，Google Play 会向缺少该硬件的设备隐藏你的应用。

**跨平台陷阱**：如果你的应用只是"可选"使用相机（比如只在某个小功能中用到了拍照），应该设为 `required="false"` 并运行时检查硬件可用性。否则你的应用会无缘无故丢失大量潜在用户。

### 清单合并（Manifest Merger）深入

清单合并在跨平台中特别重要，因为第三方插件和库会带来它们自己的清单配置。

**合并优先级（从高到低）**：
1. 主模块的 `AndroidManifest.xml`
2. 构建变体的清单文件（`src/<flavor>/AndroidManifest.xml`）
3. 依赖库的清单文件

**冲突解决**：
- 高优先级属性覆盖低优先级属性
- 如果未设置合并策略且存在冲突，合并会成功但产生警告
- 使用 `tools:replace` 声明要覆盖的属性

**调试技巧**：合并后的最终清单文件可以在以下位置找到：
```
app/build/intermediates/merged_manifests/<variant>/AndroidManifest.xml
```

在 Flutter 中可以通过 `flutter build apk --debug` 然后查看该文件，验证所有插件声明是否被正确合并。

## 实战案例

### 案例一：Flutter 集成相机插件

**场景**：在 Flutter 项目中集成 `image_picker` 插件，让用户可以选择照片或拍照。

**需要修改的配置**：

```xml
<!-- Android 12+ 不需要单独声明相机权限 -->
<uses-permission android:name="android.permission.CAMERA" />
<!-- 因为 image_picker 清单文件中已声明 -->
```

但如果在 Android 10 以下设备上需要读取图片文件，还需：

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
```

**容易忽略的点**：
- Android 10（API 29）引入了分区存储（Scoped Storage），不需要存储权限就能访问自己应用创建的文件。
- Android 13（API 33）引入了细粒度的媒体权限，读取图片、视频、音频的权限被拆分成三个独立的权限。
- 不正确的 `maxSdkVersion` 设置可能导致在旧设备上无法使用。

### 案例二：React Native 集成微信分享

**场景**：RN 应用中集成微信分享功能，需要配置微信回调 Activity。

**核心配置**：

```xml
<activity
    android:name=".wxapi.WXEntryActivity"
    android:exported="true"
    android:launchMode="singleTask"
    android:taskAffinity="com.example.myapp"
    android:theme="@android:style/Theme.Translucent.NoTitleBar" />
```

微信 SDK 通过隐式 Intent 与这个 Activity 通信，所以必须在清单文件中准确注册。同时，微信开放平台注册应用时使用的包名必须与 `AndroidManifest.xml` 中的 `package` 完全一致。

**常见错误**：
- 包名拼写错误（大小写敏感）
- 忘记给签名应用签名（微信要求签名一致性）
- `WXEntryActivity` 路径不对（必须是 `{package}.wxapi.WXEntryActivity`）

### 案例三：鸿蒙应用兼容 Android 层

**场景**：鸿蒙应用通过兼容层运行 Android 模块，需要同时满足两个平台的要求。

**关键配置点**：
- 鸿蒙的 `AndroidManifest.xml` 需要声明 `com.huawei.hms` 相关的权限和组件
- 生命周期配置需要考虑两个平台的差异
- 部分权限名称在两个平台上可能不同

## 常见问题

### Q1: 编译报错"Manifest merger failed"

**原因**：多个库的清单文件存在冲突且无法自动合并。

**解决方案**：
1. 查看详细的合并错误日志：`./gradlew processDebugManifest --stacktrace`
2. 在应用的主 `AndroidManifest.xml` 中添加 `tools:replace` 或 `tools:remove`：
```xml
<manifest xmlns:tools="http://schemas.android.com/tools">
    <application
        android:label="@string/app_name"
        tools:replace="android:label">
```

### Q2: 应用安装后不显示图标

**可能原因**：
1. 没有 `MAIN/LAUNCHER` 的 intent-filter
2. `exported="false"` 导致系统无法启动启动 Activity
3. 清单文件合并后 `LAUNCHER` 设置被覆盖

**排查方法**：查看合并后的清单文件，确认入口 Activity 的配置。

### Q3: 发布后发现权限过多

**原因**：插件自动引入了很多权限。例如 Firebase Analytics 插件可能会引入 `RECEIVE_BOOT_COMPLETED` 权限。

**解决方案**：
1. 审核合并后的最终权限列表
2. 使用 `tools:node="remove"` 移除不需要的权限：
```xml
<uses-permission
    android:name="android.permission.RECEIVE_BOOT_COMPLETED"
    tools:node="remove" />
```

### Q4: 深链接（Deep Link）无法跳转

**排查步骤**：
1. 确认 `intent-filter` 中的 data 标签配置正确
2. 在 Android 12+ 上需要额外的验证步骤
3. 检查其他应用是否注册了相同的 URL Scheme
4. 使用 `adb` 命令测试深链接：
```bash
adb shell am start -W -a android.intent.action.VIEW -d "https://www.myapp.com/product/123" com.example.myapp
```

## 总结

AndroidManifest.xml 是 Android 应用的基石，对跨平台开发者来说，它是打通 Flutter/RN/鸿蒙原生能力的关键桥梁。理解了这个文件的结构和每个标签的含义，你将能：

1. **快速定位问题**：当插件集成报错或功能异常时，能迅速判断是否与清单配置有关
2. **精准配置权限**：只声明应用真正需要的权限，减少安全审计风险
3. **理解合并机制**：知道为什么某些配置会自动出现，以及如何主动覆盖
4. **优化用户体验**：通过正确的 Activity 和 Service 配置，提升应用的稳定性和响应性

建议在工作中经常查看合并后的最终清单文件（`build/intermediates/merged_manifests/`），这是了解你的应用"真正"在系统层面展示了什么信息的最好方式。

记住一个核心原则：**Android 系统只信任清单文件中声明的内容**。你可以在代码中做任何事，但如果清单文件中没有相应的声明，系统会无情地拒绝你的请求。
