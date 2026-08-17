---
title: build.gradle.kts 配置实战——跨平台开发者的 Gradle 构建指南
date: 2026-10-14
categories: ["原生扩展", "Android"]
tags: ["原生扩展", "Android", "Gradle", "build.gradle.kts", "AGP", "构建配置", "插件开发"]
description: 系统讲解 Android 项目 build.gradle.kts 配置的方方面面，从项目级到模块级的配置要点、依赖管理、多渠道打包，帮助跨平台开发者掌控构建流程。
---

## 一句话概括

build.gradle.kts 是 Android 项目的构建配置入口，它定义了从依赖库到编译参数的整套构建规则，跨平台开发者掌握它就能独立解决大部分原生构建问题。

## 背景与意义

### Gradle 与 build.gradle.kts 是什么

Android 项目使用 Gradle 作为构建系统。Gradle 是一个基于 Groovy 或 Kotlin 的自动化构建工具，而 `build.gradle.kts` 就是用 Kotlin DSL 编写的构建脚本文件（对应传统的 `build.gradle` 是 Groovy DSL 版本）。

从 Android Studio 4.2 开始，官方推荐使用 Kotlin DSL（即 `.kts` 后缀）来编写构建脚本，因为它提供了更好的 IDE 支持——代码补全、类型检查、编译时错误检测，这些在 Groovy 版本的构建脚本中是无法享受的。

### 为什么跨平台开发者需要理解 Gradle 配置

Flutter 开发者常有这样的经历：执行 `flutter run` 时突然报错，终端显示一堆 Gradle 相关的错误信息。这些报错可能涉及插件版本冲突、AGP 版本不兼容、Java 版本不正确等。如果没有 Gradle 配置的基础知识，看到这些错误往往只能网上盲目搜索，效率极低。

React Native 场景下，执行 `npx react-native run-android` 报错后，很多开发者选择"重启大法"（clean → rebuild），但这治标不治本。真正的问题藏在 `build.gradle.kts` 的配置细节中。

对于使用 Flutter 3.x/RN 0.70+ 的开发者来说，Android 构建配置已经越来越复杂：
- AGP 从 7.x 升级到 8.x 引入了大量的 API 变更
- Kotlin 版本需要与 AGP 版本严格匹配
- Java 17 成为必须（AGP 8.0+）
- Android 13/14 的目标 SDK 要求不断更新

理解 `build.gradle.kts` 就是在理解现代 Android 构建体系的底层逻辑，它能让你不再是"尝试-失败-搜索"的被动模式，而是主动掌控构建过程。

### Groovy DSL vs Kotlin DSL

| 特性 | Groovy DSL (.gradle) | Kotlin DSL (.kts) |
|------|---------------------|-------------------|
| 类型安全 | 否 | 是 |
| IDE 补全 | 有限 | 强 |
| 学习曲线 | 低（脚本化） | 中（需 Kotlin 基础） |
| 编译报错消息 | 运行时发现 | 编译时发现 |
| 与 Gradle API 交互 | 动态 | 静态类型化 |

如果你还在使用 `.gradle` 文件，建议迁移到 `.kts`。迁移过程并不复杂，改后缀名 + 修复少量语法即可。

## 核心知识点拆解

### 项目级 build.gradle.kts（Project-Level）

项目级 `build.gradle.kts` 位于项目根目录，它定义了整个项目的全局配置。

```kotlin
// 项目根目录的 build.gradle.kts
plugins {
    id("com.android.application") version "8.1.4" apply false
    id("com.android.library") version "8.1.4" apply false
    id("org.jetbrains.kotlin.android") version "1.9.20" apply false
    // Flutter 项目还会包含 Flutter 的插件
    // RN 项目可能会包含 react-native 相关插件
}
```

#### plugins 块的作用

- `apply false` 表示声明插件但不在此模块应用。真正的"应用"在模块级 `build.gradle.kts` 中完成。
- Android 相关的核心插件有两个：
  - `com.android.application`：构建 Android 应用的插件
  - `com.android.library`：构建 Android 库的插件

**在 Flutter 中**：Flutter 项目创建后，项目级 `build.gradle.kts` 会被自动配置，但版本号由 Flutter SDK 管理。如果手动修改 `com.android.application` 的版本，会遇到 AGP 版本与 Flutter 支持版本不兼容的问题。

**在 RN 中**：RN 0.71+ 使用 AGP 8.x，项目级 `build.gradle.kts` 会使用 `react-native` 定义的插件。

#### repositories 块（仓库配置）

`repositories` 定义了 Gradle 从哪里下载依赖库。

```kotlin
// settings.gradle.kts（较新版本中 repositories 移到了这里）
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()          // Google Maven 仓库
        mavenCentral()    // Maven Central 仓库
        maven { url = uri("https://jitpack.io") } // JitPack 仓库
        // 某些插件的私有仓库
        maven { url = uri("https://example.com/repo") }
    }
}
```

**注意**：Android Gradle Plugin 7.0+ 引入了 `dependencyResolutionManagement`，将仓库管理集中到 `settings.gradle.kts` 中。如果项目结构较老，仓库配置可能仍在项目级 `build.gradle.kts` 中。

**国内镜像**：由于网络原因，国内开发者常需要配置镜像源：

```kotlin
maven { url = uri("https://maven.aliyun.com/repository/public") }
maven { url = uri("https://maven.aliyun.com/repository/google") }
maven { url = uri("https://maven.aliyun.com/repository/gradle-plugin") }
```

### 模块级 build.gradle.kts（Module-Level）

模块级 `build.gradle.kts` 位于 `app/` 目录下，这是跨平台开发者需要修改最多的文件。

```kotlin
// app/build.gradle.kts
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    // Flutter 项目会自动添加
    id("dev.flutter.flutter-gradle-plugin")
    // RN 项目会添加
    // id("com.facebook.react")
}

android {
    namespace = "com.example.myapp"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 21
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("debug") // 实际应使用正式签名
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation(project(":flutter"))
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    // ... 其他依赖
}
```

#### android {} 块

这是模块配置的核心区域，包含了所有与 Android 编译相关的配置。

**compileSdk**
- 指定编译时使用的 Android API 版本。
- 设为最新的稳定版 SDK 是好的做法，可以使用最新的 API 但不需要强制用户升级设备。
- 编译 SDK 版本不影响应用的兼容性。

**minSdk**
- 应用支持的最低 Android API 级别。
- 当前大多数应用设置 `minSdk = 21`（Android 5.0），覆盖了 95%+ 的设备。
- 设置过低（如 `minSdk = 16`）会增加兼容性测试的工作量，且可能无法使用新的 API。

**跨平台决策参考**：
- Flutter 官方推荐的 `minSdk` 是 21
- RN 0.72+ 的 `minSdk` 也提升到了 21
- 如果你的应用需要支持更低版本，需要额外注意三方库的兼容性

**targetSdk**
- 应用"目标"运行的 Android 版本。
- 最重要的一点是：当 `targetSdk` 提升到某个版本时，该版本引入的所有行为变更都将应用于你的应用。
- 例如 `targetSdk = 33` 后，`READ_EXTERNAL_STORAGE` 权限不再生效，需要用细粒度媒体权限替代。
- Google Play 要求新应用的上传目标 SDK 必须在当前版本一年内。

**跨平台"血泪教训"**：很多 Flutter/RN 应用在运行正常时不会主动提升 `targetSdk`，直到被迫更新时才一次性追赶多个版本。这种情况下，多个版本的行为变更叠加在一起，定位问题变得非常困难。建议每年至少提升一次 `targetSdk`，每次只提升一个版本。

#### versionCode 与 versionName

在跨平台框架中，版本号的管理方式有所不同：

**Flutter**：`pubspec.yaml` 中的 `version: 1.0.0+1` 会自动映射到 `versionName` 和 `versionCode`。不要在模块级 `build.gradle.kts` 中重复设置，否则会被覆盖。

**RN**：版本管理在 `package.json` 中定义，但 Android 端的版本需要同时更新 `build.gradle.kts`。很多自动化 CI 工具会通过脚本同步这两个位置。

### buildTypes——构建类型

`buildTypes` 定义了不同构建模式的配置，默认有 `debug` 和 `release` 两种。

```kotlin
buildTypes {
    debug {
        // debug 模式默认使用 debug.keystore 签名
        isDebuggable = true
        applicationIdSuffix = ".debug"
    }
    release {
        // 是否启用代码混淆
        isMinifyEnabled = true
        // 是否启用资源压缩
        isShrinkResources = true
        // ProGuard 混淆规则文件
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro"
        )
        // 签名配置（生产环境必须配置正式的 release 签名）
        signingConfig = signingConfigs.getByName("release")
    }
}
```

#### 签名配置

签名是 Android 应用的身份验证机制。

```kotlin
android {
    signingConfigs {
        release {
            storeFile = file("release.keystore")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }
}
```

**安全最佳实践**：永远不要在版本控制系统中提交签名密码。使用环境变量或 CI 系统的密钥管理功能。

**跨平台 CI/CD 注意**：Flutter 的 `flutter build apk --release` 和 RN 的 `npx react-native build-android` 都使用 `release` 构建类型。如果签名配置不正确，生成的 APK 无法在 Google Play 上发布。

#### minifyEnabled 与 ProGuard/R8

- `isMinifyEnabled = true` 启用代码混淆和压缩。
- R8 是 Android 官方推荐的代码压缩工具（从 Android Gradle Plugin 3.4.0 开始替代 ProGuard）。
- 混淆会重命名类、方法和变量名，减小 APK 体积并增加逆向难度。
- 混淆可能导致反射调用失败、序列化异常等问题。

**跨平台常见问题**：
- Flutter 插件中的 Java/Kotlin 代码如果使用了反射，必须添加混淆规则
- RN 原生模块中的方法如果被混淆，会破坏 JS 与 Native 的通信桥梁
- 一些 JSON 解析库（Gson、Moshi）的模型类需要添加 `@Keep` 注解或混淆保留规则

#### proguard-rules.pro 典型规则

```
# 保留 Flutter 引擎相关类
-keep class io.flutter.app.** { *; }
-keep class io.flutter.plugin.** { *; }
-keep class io.flutter.util.** { *; }
-keep class io.flutter.view.** { *; }

# 保留 RN 原生模块
-keep class com.facebook.react.** { *; }

# 保留数据模型类（JSON 序列化/反序列化）
-keep class com.example.myapp.model.** { *; }

# 保留使用 @Keep 注解的类
-keep @androidx.annotation.Keep class * { *; }
-keepclassmembers class * {
    @androidx.annotation.Keep *;
}
```

### dependencies——依赖管理

`dependencies` 块声明模块需要的各种依赖库。

#### 依赖配置类型

| 关键字 | 作用 | 传递性 | 使用场景 |
|--------|------|--------|----------|
| `implementation` | 编译和运行时可见，但对外部隐藏 | 非传递 | 大部分依赖 |
| `api` | 编译和运行时可见，对外部也可见 | 传递 | 库的公共 API 依赖 |
| `compileOnly` | 仅在编译时可见 | 非传递 | 注解处理器、编译时工具 |
| `runtimeOnly` | 仅在运行时可见 | 非传递 | 运行时实现 |
| `annotationProcessor` | 注解处理器 | N/A | Room、Glide 等 |

**`implementation` vs `api`**：这是最需要理解的区别。

- `implementation("lib-a")`：lib-a 在编译时可用，但不会泄露给依赖当前模块的其他模块。
- `api("lib-a")`：lib-a 既对当前模块可见，也对依赖当前模块的其他模块可见。

**最佳实践**：绝大多数依赖使用 `implementation`。只在库模块需要暴露依赖给调用方时使用 `api`。使用 `implementation` 可以显著加快增量构建速度，因为当 `implementation` 依赖变化时，只有直接使用它的模块需要重新编译。

#### 版本管理

**方法一：直接在 dependencies 中写版本号**

```kotlin
implementation("androidx.core:core-ktx:1.12.0")
```

最直接但难以集中管理，在多模块项目中容易产生版本冲突。

**方法二：使用 version catalog（推荐）**

在 `gradle/libs.versions.toml` 中集中定义：

```toml
[versions]
agp = "8.1.4"
kotlin = "1.9.20"
core-ktx = "1.12.0"
appcompat = "1.6.1"

[libraries]
core-ktx = { module = "androidx.core:core-ktx", version.ref = "core-ktx" }
appcompat = { module = "androidx.appcompat:appcompat", version.ref = "appcompat" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
```

然后在 `build.gradle.kts` 中引用：

```kotlin
dependencies {
    implementation(libs.core.ktx)
    implementation(libs.appcompat)
}
```

Version Catalog 的好处：
- 单点维护所有版本号
- 自动生成类型安全的访问器（`libs.core.ktx`）
- 跨模块共享版本定义
- 检测版本冲突

### 多渠道打包——flavorDimensions + productFlavors

多渠道打包用于构建不同版本的应用（如免费版/付费版、国内版/海外版）。

```kotlin
android {
    flavorDimensions += "version"

    productFlavors {
        create("free") {
            dimension = "version"
            applicationIdSuffix = ".free"
            versionNameSuffix = "-free"
        }
        create("paid") {
            dimension = "version"
            applicationIdSuffix = ".paid"
            versionNameSuffix = "-paid"
        }
    }
}
```

#### 多维度配置

```kotlin
flavorDimensions += listOf("version", "market")

productFlavors {
    create("free") { dimension = "version" }
    create("paid") { dimension = "version" }
    create("googlePlay") { dimension = "market" }
    create("china") { dimension = "market" }
}
```

上面的配置会生成 4 种变体（2 × 2）：
- `freeGooglePlayDebug` / `freeGooglePlayRelease`
- `freeChinaDebug` / `freeChinaRelease`
- `paidGooglePlayDebug` / `paidGooglePlayRelease`
- `paidChinaDebug` / `paidChinaRelease`

#### 变体过滤与自定义

```kotlin
android {
    variantFilter {
        if (name == "freeChinaRelease") {
            ignore = true  // 忽略某些不需要的变体
        }
    }
}
```

**在跨平台项目中的应用**：Flutter 项目中可以通过 flavor 来配置不同的 API 基础地址、应用图标或功能集。RN 项目同理，可以利用 `react-native-config` 插件配合 flavor 实现环境切换。

### AGP 版本兼容性

Android Gradle Plugin（AGP）版本与 Gradle 版本、Java 版本之间存在严格的对应关系。

| AGP 版本 | 最低 Gradle 版本 | 推荐 JDK 版本 | 说明 |
|----------|-----------------|---------------|------|
| 7.0 | 7.0 | JDK 11 | Java 8 兼容层支持 |
| 7.4 | 7.5 | JDK 11+ | 最后一个支持 API 31 以下 |
| 8.0 | 8.0 | JDK 17 | 编译版本提升至 33 |
| 8.1 | 8.0 | JDK 17 | 需要 compileSdk 33+ |
| 8.2 | 8.2 | JDK 17 | API 34 支持 |
| 8.3+ | 8.4+ | JDK 17+ | 持续演进 |

**跨平台版本矩阵参考**：

Flutter 与 AGP 的兼容关系（以常见版本为例）：
- Flutter 3.10+ → AGP 7.4+、Gradle 8.0+
- Flutter 3.13+ → AGP 8.0+、Gradle 8.2+
- Flutter 3.16+ → AGP 8.1+、Gradle 8.3+

RN 与 AGP 的兼容关系：
- RN 0.71 → AGP 7.4
- RN 0.72 → AGP 8.0
- RN 0.73+ → AGP 8.1+

**降级指南**：如果你的项目因为某些原因需要使用较老版本的 AGP：
1. 确保 Gradle 版本与 AGP 要求匹配
2. 确保 JDK 版本满足最低要求
3. 检查所有插件是否与新 AGP 兼容
4. 注意某些功能在旧版本 AGP 中可能不可用（如 version catalog）

## 实战案例

### 案例一：Flutter 项目集成 Firebase

**场景**：在 Flutter 项目中集成 Firebase 推送和 Analytics。

**项目级 build.gradle.kts 修改**：

```kotlin
plugins {
    id("com.android.application") version "8.1.4" apply false
    id("com.google.gms.google-services") version "4.4.0" apply false
    id("com.google.firebase.crashlytics") version "2.9.9" apply false
}
```

**模块级 build.gradle.kts 修改**：

```kotlin
plugins {
    id("com.android.application")
    id("com.google.gms.google-services")
    id("com.google.firebase.crashlytics")
}

dependencies {
    implementation(platform("com.google.firebase:firebase-bom:32.7.0"))
    implementation("com.google.firebase:firebase-analytics")
    implementation("com.google.firebase:firebase-messaging")
    implementation("com.google.firebase:firebase-crashlytics")
}
```

**需要注意**：
1. Firebase BOM（Bill of Materials）可以自动管理 Firebase SDK 的版本兼容性。
2. `google-services.json` 文件需要放在 `app/` 目录下。
3. 在 CI/CD 构建时，需要确保该文件存在于构建环境中，不要提交到版本控制。

### 案例二：React Native 项目接入微信支付

**场景**：在 RN 项目中接入微信支付，需要依赖微信 SDK 并进行 ProGuard 配置。

```kotlin
dependencies {
    implementation("com.tencent.mm.opensdk:wechat-sdk-android-without-mta:6.8.0")
}
```

**混淆规则（proguard-rules.pro）**：
```
-keep class com.tencent.mm.opensdk.** { *; }
-keep class com.tencent.wxop.** { *; }
-keep class net.sourceforge.pinyin4j.** { *; }
```

### 案例三：构建变体——区分开发与生产环境

```kotlin
flavorDimensions += "environment"

productFlavors {
    create("development") {
        dimension = "environment"
        applicationIdSuffix = ".dev"
        versionNameSuffix = "-dev"
    }
    create("staging") {
        dimension = "environment"
        applicationIdSuffix = ".staging"
        versionNameSuffix = "-staging"
    }
    create("production") {
        dimension = "environment"
        // 生产环境不加后缀
    }
}
```

配合 `src/<flavor>/res/values/config.xml` 可以定义不同环境的 API 地址。

## 常见问题

### Q1: 编译报错"Invoke-customs are only supported starting with Android R"

**原因**：项目使用了 Java 8+ 特性但 `minSdk` 设置过低。

**解决方案**：在 `android {}` 块中添加：

```kotlin
compileOptions {
    isCoreLibraryDesugaringEnabled = true
}
dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

这需要在 `minSdk` 较低的设备上使用 Java 新特性时添加。

### Q2: "Could not find method implementation() for arguments"

**原因**：依赖声明格式错误或使用了错误的 Gradle/Groovy 语法。

**解决**：确认使用的是正确的 DSL 语法：
- Groovy: `implementation 'com.example:lib:1.0.0'`
- Kotlin: `implementation("com.example:lib:1.0.0")`

### Q3: 插件版本冲突

**症状**：多个插件依赖了同一个库的不同版本。

**解决方案**：
1. 使用 `./gradlew app:dependencies` 查看依赖树
2. 使用 `force` 强制指定版本：
```kotlin
configurations.all {
    resolutionStrategy {
        force("androidx.core:core-ktx:1.12.0")
    }
}
```
3. 使用 BOM 统一管理版本（如 Firebase BOM）

### Q4: 编译速度太慢

**优化建议**：
1. 在 `gradle.properties` 中启用构建缓存和并行编译：
```properties
org.gradle.caching=true
org.gradle.parallel=true
org.gradle.jvmargs=-Xmx4096m
```
2. 将不需要的依赖从 `api` 改为 `implementation`
3. 使用 version catalog 减少重复配置
4. 考虑使用 Gradle 构建扫描分析瓶颈

## 总结

build.gradle.kts 是 Android 构建体系的核心，对跨平台开发者来说，掌握它是摆脱"半吊子"原生技能状态的关键一步。

回顾本文的核心要点：
1. **项目级配置**负责插件声明和全局仓库配置，是构建的起点
2. **模块级配置**是"主战场"，包含 SDK 版本、构建类型、签名、混淆和依赖管理
3. **依赖配置**要理解 `implementation` 和 `api` 的区别，优先使用前者
4. **多渠道打包**让同一个代码库输出多个变体，是 CI/CD 的基础
5. **版本兼容性**是永恒的难题，AGP、Gradle、JDK 的版本组合必须精确匹配

当你在 Flutter 或 RN 项目中遇到 Gradle 报错时，不要害怕去读那些看似复杂的错误信息。大多数 Gradle 错误的错误信息都直接指明了问题所在——版本冲突、缺少依赖、语法错误等。掌握 `build.gradle.kts` 就是掌握了 Android 构建的翻译词典，让你能理解 Gradle 在说什么，以及如何回应它。
