---
name: upgrade-to-gradle8
description: >-
  Upgrade Android/Gradle projects to Gradle 8 (wrapper 8.11.1-all) and AGP 8.9.1.
  Applies remote gradle8-compact.gradle compatibility script, updates gradle.properties
  R/R8 flags, namespace, buildConfig, and SDK conventions.
  Use when upgrading to Gradle 8, AGP 8, migrating from AGP 7.x / Gradle 7,
  fixing AGP 8 build errors, or when the user mentions Gradle8 / AGP8 升级.
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
- [ ] 5. 补齐 namespace / 清理 buildToolsVersion / 统一 SDK
- [ ] 6. 同步并修复编译错误
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

## 验证

升级完成后执行：

```bash
./gradlew assembleDebug
```

若失败，优先检查：namespace 是否齐全、是否已 apply 远程兼容脚本、`gradle.properties` 三项 R/R8 开关是否到位。
