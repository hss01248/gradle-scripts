# gradle-scripts

Android Gradle / AGP 升级相关的**远程脚本**与 **Cursor Skill**。业务仓库通过 URL `apply from` 引用脚本；Agent 通过 skill 按固定清单完成 Gradle 8 / AGP 8.9.1 升级。

## 仓库结构

```
gradle/
  gradle8-compact.gradle   # AGP 8 兼容开关（buildConfig / Java17 / multidex 等）
  add_firebase.gradle      # 有 google-services.json 时自动接 Analytics / Crashlytics / Performance
skills/
  upgrade-to-gradle8/      # Cursor Skill：升级步骤与 targetSdk 兼容经验
```

## Gradle 脚本用法

根 `build.gradle` 用远程 URL apply，**不要**把脚本拷进业务仓库。

**优先用 jsDelivr CDN**（国内更稳）；GitHub raw 作备选。`@main` 跟最新 main；要锁版本可改成 `@<commit>` 或 `@<tag>`。

| 脚本 | jsDelivr | GitHub raw |
|------|----------|------------|
| compact | `https://cdn.jsdelivr.net/gh/hss01248/gradle-scripts@main/gradle/gradle8-compact.gradle` | `https://raw.githubusercontent.com/hss01248/gradle-scripts/refs/heads/main/gradle/gradle8-compact.gradle` |
| firebase | `https://cdn.jsdelivr.net/gh/hss01248/gradle-scripts@main/gradle/add_firebase.gradle` | `https://raw.githubusercontent.com/hss01248/gradle-scripts/refs/heads/main/gradle/add_firebase.gradle` |

### `gradle8-compact.gradle`

AGP 8 兼容集中配置：统一开启 `buildConfig` / `viewBinding`、Java/Kotlin 17、appcompat、application 的 multidex / 冲突 exclude，以及 minify 时附加 R8 dontwarn（若根目录有 `gradle8-r8-dontwarn.pro`）。

```gradle
// 推荐：jsDelivr
apply from: 'https://cdn.jsdelivr.net/gh/hss01248/gradle-scripts@main/gradle/gradle8-compact.gradle'

// 备选：GitHub raw
// apply from: 'https://raw.githubusercontent.com/hss01248/gradle-scripts/refs/heads/main/gradle/gradle8-compact.gradle'
```

### `add_firebase.gradle`

前提：application 模块已有 `google-services.json`。脚本自动接入 Analytics / Crashlytics / Performance（含 perf 插桩）。

```gradle
// 推荐：jsDelivr
apply from: 'https://cdn.jsdelivr.net/gh/hss01248/gradle-scripts@main/gradle/add_firebase.gradle'

// 备选：GitHub raw
// apply from: 'https://raw.githubusercontent.com/hss01248/gradle-scripts/refs/heads/main/gradle/add_firebase.gradle'
```

Crashlytics NDK 默认关闭。需要采集 native 崩溃时再打开（每个 ABI 约多 0.8~2MB；`firebase-crashlytics-ndk` 声明 minSdk 24，项目若锁 minSdk 21 需 Manifest `tools:overrideLibrary`）：

```gradle
// URL query（CDN / raw 均可；脚本从 apply 行解析参数）
apply from: 'https://cdn.jsdelivr.net/gh/hss01248/gradle-scripts@main/gradle/add_firebase.gradle?firebase_ndk=true'

// 或 gradle.properties / -P
// firebase_ndk=true
```

本地文件 apply 不要带 `?query`（会被当成文件名）；本地请用 `-P` / `ext` / 脚本内默认值。

jsDelivr 对 `@main` 有短缓存；改完脚本若业务工程仍拉到旧内容，可把 `@main` 换成具体 commit，或在 URL 后加无意义 query（如 `?v=2`）强制换地址。

## Skill 用法

路径：[`skills/upgrade-to-gradle8/SKILL.md`](skills/upgrade-to-gradle8/SKILL.md)

### 做什么

指导 Agent / 人工把 Android 工程从 AGP 7.x / Gradle 7 升到：

| 组件 | 版本 |
|------|------|
| Gradle Wrapper | `8.11.1-all` |
| AGP | `8.9.1` |
| compileSdk / targetSdk | `36` |
| minSdk | **21（锁死，禁止提到 24+）** |

并覆盖：namespace、`gradle.properties` R/R8 开关、AspectJX 替换、仓库不要上 settings DRM、JDK 17，以及 targetSdk 34/35/36 运行时兼容（WindowInsets、Predictive Back、FGS、媒体权限等）。

### 在 Cursor 里怎么用

1. 把本仓库的 `skills/upgrade-to-gradle8` 放到可用 skills 目录（本仓库内、项目 `.cursor/skills/`，或个人 `~/.cursor/skills/`）。
2. 对话里说明要「升级到 Gradle 8 / AGP 8」，或直接点名 skill `upgrade-to-gradle8`。
3. Agent 按 skill 工作流改目标工程：Wrapper → AGP → 远程 apply `gradle8-compact.gradle` → `gradle.properties` → SDK/namespace → 运行时兼容。

完整步骤与排错以 skill 正文为准；脚本只承载可复用的 Gradle 逻辑，经验与 checklist 放在 skill。

## 版本约定（摘要）

- **minSdk 必须 21**：`minSdk >= 23` 时 `.so` 默认不压缩，APK 体积接近翻倍。
- Firebase / GMS 声明更高 minSdk 时，用 Manifest `tools:overrideLibrary` 放行，**不要**抬高 minSdk。
- AGP 8.9.1 需要 **JDK 17+**。
- Predictive Back（target 36）：未迁移前 Manifest 加 `android:enableOnBackInvokedCallback="false"`。
- Edge-to-edge（target 35+）：真正处理 WindowInsets；target 36 上主题 opt-out 已失效。
