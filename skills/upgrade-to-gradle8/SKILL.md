---
name: upgrade-to-gradle8
description: >-
  Upgrade Android/Gradle projects to Gradle 8 (wrapper 8.11.1-all) and AGP 8.9.1.
  Applies remote gradle8-compact.gradle, gradle.properties R/R8 flags, namespace,
  SDK conventions, and targetSdk 34/35/36 runtime compat (WindowInsets edge-to-edge,
  predictive back / onBackPressed, FGS types, media permissions, 16KB page size).
  Use when upgrading to Gradle 8, AGP 8, migrating from AGP 7.x / Gradle 7,
  fixing AGP 8 build errors, targetSdk 35/36 UI/back issues, or Gradle8 / AGP8 升级.
---

# 升级到 Gradle 8

将 Android 项目从 AGP 7.x / Gradle 7 升级到 **AGP 8.9.1 + Gradle 8.11.1**。

## 版本锁定（必须）

| 组件 | 版本 |
|------|------|
| Gradle Wrapper | `gradle-8.11.1-all` |
| AGP (`com.android.tools.build:gradle`) | `8.9.1` |

不得擅自更换上述版本；若需调整，先与用户确认。

## 工作流

```
Task Progress:
- [ ] 1. 升级 Gradle Wrapper
- [ ] 2. 升级 AGP 到 8.9.1
- [ ] 3. apply 远程 gradle8-compact.gradle
- [ ] 4. 调整 gradle.properties
- [ ] 5. 补齐 namespace / 清理 buildToolsVersion / 统一 SDK（minSdk 锁 21）
- [ ] 6. 处理 targetSdk 34/35/36 运行时兼容（Insets / Back / 权限等）
- [ ] 7. 同步并修复编译错误
```

## 1. 兼容脚本（远程依赖）

根 `build.gradle` **必须**通过远程 URL apply 兼容脚本，不要把脚本拷进业务仓库：

```gradle
apply from: 'https://raw.githubusercontent.com/hss01248/gradle-scripts/refs/heads/main/gradle/gradle8-compact.gradle'
```

该脚本统一处理：

- `buildFeatures.buildConfig = true`（AGP 8 默认关闭）
- `viewBinding = true`
- Java / Kotlin `jvmTarget` → 17
- application / library 统一 `appcompat`
- application 统一 `multidex`、冲突依赖 exclude、R8 dontwarn（若根目录有 `gradle8-r8-dontwarn.pro`）

后续兼容开关优先追加到该脚本，不要分散改各 module。

### 1.1 AspectJX / Transform API（典型坑）

- **现象**：同步或打包时报 `API 'android.registerTransform' is removed`，堆栈里有 `AJXPlugin` / `io.github.wurensen.android-aspectjx` / `com.hujiang.android-aspectjx`。
- **原因**：AGP 8 移除 Transform API；旧 `remote3.gradle`（如 `flipperUtil/gradle7_5`、`gradle8`）仍会给 application 自动 apply AspectJX。
- **处理**：把 flipper 远程脚本换成 **无 AspectJ** 分支：

```gradle
apply from: 'https://raw.githubusercontent.com/hss01248/flipperUtil/gradle7_5_no_aspectj/remote3.gradle?a=3'
```

- 可选兜底（即使脚本仍读开关）：在 `gradle.properties` 加 `flipper_use_aspectjx=false`（以及不需要 DoKit 时 `flipper_use_dokit=false`）。
- **不要**再尝试升级旧 AspectJX 插件硬扛 AGP 8.9.1；本 skill 约定直接换 no_aspectj 远程脚本。

### 1.2 仓库配置：继续用 buildscript / remote 脚本，不要上 settings DRM

- **不要**在 `settings.gradle` 写：

```gradle
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    // 或 FAIL_ON_PROJECT_REPOS
    repositories { ... }
}
```

- **原因**：`PREFER_SETTINGS` / `FAIL_ON_PROJECT_REPOS` 会拦截后续 `allprojects { repositories }`、`assignRepos(project)` 等脚本注入，直接报错或仓库失效。
- **正确做法**：维持原先 `buildscript` + 远程脚本（如 `remote3.gradle` 的 `assignRepos` / `allprojects.repositories`）管依赖仓库；`settings.gradle` 只保留 `include` / `rootProject.name`。
- 若短暂出现 `no repositories are defined`，先确认已 apply 带仓库注入的远程脚本，而不是改成 settings DRM。

### 1.3 JDK 17

- AGP 8.9.1 **必须 JDK 17+**。本机若默认仍是 JDK 11，构建前设置：
  `export JAVA_HOME=$(/usr/libexec/java_home -v 17)`（macOS）。

## 2. 从 AGP 7.x 升级到 8.9.1 的实际经验总结

### 1. Gradle Wrapper 升级

- `gradle/wrapper/gradle-wrapper.properties` 中 `distributionUrl` 升级到 `gradle-8.11.1-all.zip`
- AGP 8.9.1 要求 Gradle 8.11+

```properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.11.1-all.zip
```

### 2. gradle.properties 调整

- JVM 参数：`org.gradle.jvmargs=-Xmx6144m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8 -XX:+UseParallelGC`
- 移除已废弃的 `-XX:MaxPermSize`（Java 8+ 无此参数）
- 添加 R 资源兼容配置：
  - `android.nonFinalResIds=false`（保持 R.id 为 final，兼容 switch 语句）
  - `android.nonTransitiveRClass=false`（保持传递 R class，兼容旧代码）
  - `android.enableR8.fullMode=false`（关闭 R8 full mode，避免混淆问题）
- TLS 配置（如项目需要）：
  - `systemProp.jdk.tls.disabledAlgorithms=""`
  - `systemProp.jdk.tls.client.protocols=TLSv1.2`

推荐写入示例：

```properties
org.gradle.jvmargs=-Xmx6144m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8 -XX:+UseParallelGC

android.nonFinalResIds=false
android.nonTransitiveRClass=false
android.enableR8.fullMode=false

# 仅在项目确实需要时添加
# systemProp.jdk.tls.disabledAlgorithms=
# systemProp.jdk.tls.client.protocols=TLSv1.2
```

## 3. AGP 升级

根 `build.gradle`：

```gradle
classpath "com.android.tools.build:gradle:8.9.1"
```

## 4. Namespace（AGP 8.0+ 强制）

- 所有 `com.android.application` / `com.android.library` module 必须在 `build.gradle` 声明 `namespace`
- 取值优先用原 `AndroidManifest.xml` 的 `package`；迁移后可移除 Manifest 的 `package`
- 纯 JVM module 不需要

```gradle
android {
    namespace 'com.example.mymodule'
}
```

## 5. 移除 buildToolsVersion

- AGP 8.9.1 自带默认 Build Tools，不要写 `buildToolsVersion`
- 若写了低于最低要求的版本会 warning 并被忽略；处理方式是**删除该行**

## 6. SDK 版本约定

| 配置 | 值 |
|------|----|
| `compileSdkVersion` | 36 |
| `targetSdkVersion` | 36 |
| `minSdkVersion` | **21（锁死）** |

```gradle
android {
    compileSdkVersion 36
    defaultConfig {
        minSdkVersion 21
        targetSdkVersion 36
    }
}
```

- **`minSdkVersion` 必须锁死在 21，不得升级到 24 或更高**。`minSdk >= 23` 时 AGP/打包链路默认不再压缩 `.so`（按未压缩页面对齐存放），会导致 APK 体积接近翻倍。升级 Gradle/AGP 时若发现某处把 minSdk 提到 24，一律改回 21。

## 7. nonTransitiveRClass 说明

- **现象**：Gradle7 能编过，AGP8 报 `cannot find symbol: variable id / location: class R`
- **原因**：AGP 8 默认 `android.nonTransitiveRClass=true`，本 module 的 `R` 不再合并依赖库资源
- **处理**：按上文在 `gradle.properties` 设 `android.nonTransitiveRClass=false`
- **长期可选改法**：引用依赖库自己的 `R`，例如 `com.hss01248.dialog.R.id.xxx`

## 8. 高 targetSdk 运行时兼容（Android 14/15/16）

`compileSdk/targetSdk=36` 后，很多问题是**行为变更**而非编译错误。按优先级处理：

### 8.1 Android 15（API 35）强制 Edge-to-Edge / WindowInsets

- **现象**：状态栏/导航栏区域内容被遮挡、顶部 Toolbar 顶到刘海、底部按钮被手势条挡住；`fitsSystemWindows` / 旧 `statusBarColor` 失效。
- **原因**：`targetSdk >= 35` 且运行在 Android 15+ 时，系统强制 edge-to-edge，内容绘制到系统栏后方，必须自行消费 `WindowInsets`。
- **短期兼容（仅 target 35 有效）**：主题里临时 opt-out：

```xml
<!-- res/values-v35/themes.xml；targetSdk 36 在 Android 16 上会被忽略 -->
<style name="AppTheme" parent="...">
    <item name="android:windowOptOutEdgeToEdgeEnforcement" tools:targetApi="35">true</item>
</style>
```

- **正式做法（target 36 必须）**：根布局处理 insets，例如：

```kotlin
ViewCompat.setOnApplyWindowInsetsListener(root) { v, insets ->
    val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
    v.setPadding(bars.left, bars.top, bars.right, bars.bottom)
    insets
}
```

或使用 `WindowCompat.enableEdgeToEdge(window)` + 给 Toolbar / 底部栏分别加 padding/margin。WebView、全屏播放、沉浸式页要单独测。

### 8.2 Android 16（API 36）Predictive Back / `onBackPressed` 不再回调

- **现象**：系统返回/侧滑返回直接退出 Activity；标题栏返回（自己调 `AgentWeb.back()` 等）正常，系统返回失效。WebView / Fragment 返回栈被跳过。
- **原因**：`targetSdk >= 36` 时 Predictive Back 默认开启，`Activity.onBackPressed()` / `KEYCODE_BACK` **不再分发**。
- **短期兼容（推荐先做）**：在 `AndroidManifest.xml` 的 `<application>`（或单个 `<activity>`）关闭：

```xml
<application
    android:enableOnBackInvokedCallback="false"
    ...>
```

- **正式做法**：迁移到 `OnBackPressedDispatcher` / `OnBackPressedCallback`（或 Navigation / Compose `BackHandler`），再视情况打开 callback。

### 8.3 其他经常遇到的兼容（按 API）

| API | 问题 | 常见处理 |
|-----|------|----------|
| **33** | 通知不弹；相册读不到 | 申请 `POST_NOTIFICATIONS`；`READ_EXTERNAL_STORAGE` 换 `READ_MEDIA_IMAGES/VIDEO/AUDIO` 或 Photo Picker，旧存储权限加 `maxSdkVersion=32` 如果是上架 Google play，要求更严，非相册管理，文件管理类不允许申请`READ_MEDIA_IMAGES/VIDEO，普通选图要改造成使用 Android photo picker 库，可以使用这个封装好的： https://github.com/hss01248/media-picker |
| **34** | 前台服务启动崩溃 | 每个 FGS 声明 `android:foregroundServiceType` + 对应 `FOREGROUND_SERVICE_*` 权限 |
| **34** | 广播/组件找不到、安全异常 | `exported` 必须显式；隐式 Intent 只能打到 exported 组件；动态广播按需导出 |
| **34** | 精确闹钟失败 | `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM`，并引导用户开权限 |
| **34** | 相册只授部分图 | 处理 `READ_MEDIA_VISUAL_USER_SELECTED` / Photo Picker 部分授权 |
| **34** | 全屏 Intent 通知被拒 | 校验 `canUseFullScreenIntent()`，非通话/闹钟类引导设置页 |
| **35** | Edge-to-Edge 遮挡 | 见 8.1；勿依赖已失效的 status/navigation bar color API |
| **35** | 16KB 页大小设备装包/native 崩溃 | NDK / `.so` 按 16KB 对齐；升级 AGP 8.5.1+（本 skill 用 8.9.1）并验证三方 so |
| **35** | 后台启动限制更严、屏幕共享等 | 复查从后台拉起 Activity / 媒体投影相关逻辑 |
| **36** | Predictive Back | 见 8.2 |
| **36** | `windowOptOutEdgeToEdgeEnforcement` 失效 | 必须真正处理 WindowInsets |
| **36** | 大屏/平板方向锁定被忽略 | 勿再依赖强制 `portrait`；改自适应布局或官方大屏策略 |
| **通用** | `PendingIntent` 崩溃 | 必须带 `FLAG_IMMUTABLE` 或 `FLAG_MUTABLE` |
| **通用** | 文件分享 `FileUriExposedException` | 一律 `FileProvider` |
| **通用** | 安装/更新未知来源 APK | 走 `PackageInstaller` + 请求安装权限 |
| **通用** | `Http` 明文失败 | `usesCleartextTraffic` / network security config（仅调试或必要域名） |
| **AGP8** | duplicate class / R8 missing class | 由 `gradle8-compact.gradle` exclude + `gradle8-r8-dontwarn.pro` 兜底 |
| **AGP8** | Java/Kotlin 版本 | 统一 JDK 17（脚本已设 `VERSION_17` / `jvmTarget=17`） |

### 8.4 Manifest / 主题检查清单

升级后在 **application module** 核对：

- [ ] `android:enableOnBackInvokedCallback="false"`（未完成 Back 迁移前）
- [ ] 主题是否处理 edge-to-edge / WindowInsets（target 36 不能只靠 opt-out）
- [ ] 所有 Service 的 `foregroundServiceType` + 权限
- [ ] 通知 / 媒体权限是否按 33+ 拆分
- [ ] 所有 `PendingIntent` 带 mutability flag
- [ ] 三方 SDK（推送、支付、地图、WebView 壳）是否支持 target 35/36

## 验证

升级完成后执行：

```bash
./gradlew assembleDebug
```

真机/模拟器建议在 **API 35、36** 上回归：启动页遮挡、系统返回、WebView 返回、通知、相册、前台服务。

若失败，优先检查：namespace 是否齐全、是否已 apply 远程兼容脚本、`gradle.properties` 三项 R/R8 开关是否到位、Manifest Back/Insets 兼容是否加上。
