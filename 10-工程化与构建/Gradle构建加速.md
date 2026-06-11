---
tags: [android, 面试, 工程化, gradle, A]
difficulty: A
frequency: high
---

# Gradle 构建加速

## 问题

> 大型 Android 项目构建慢怎么优化？有哪些 Gradle 加速手段？如何分析构建瓶颈？

## 核心答案

Gradle 构建加速核心手段：开启并行编译（parallel）、启用构建缓存（build cache）、使用 Configuration Cache、减少不必要的 api 依赖传递、避免 buildSrc（改用 composite build）、升级到最新 AGP/Gradle 版本。分析工具用 `--scan` 生成 Build Scan 或 `--profile` 生成本地报告。

## 深入解析

### 原理层

**gradle.properties 优化配置：**
```properties
# 并行编译（无依赖的模块同时编译）
org.gradle.parallel=true

# 构建缓存（跨构建复用 Task 输出）
org.gradle.caching=true

# Configuration Cache（缓存配置阶段结果）
org.gradle.configuration-cache=true

# JVM 参数（增大堆内存）
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC

# Kotlin 增量编译
kotlin.incremental=true

# 非传递 R 类（减少 R.java 重编译）
android.nonTransitiveRClass=true
```

**各优化手段原理：**

| 手段 | 原理 | 效果 |
|------|------|------|
| parallel | 无依赖模块并行编译 | 多模块项目提升 30-50% |
| build cache | 缓存 Task 输出，输入不变则复用 | 增量构建提升 50%+ |
| configuration cache | 缓存配置阶段的 Task 图 | 配置阶段从秒级降到毫秒 |
| non-transitive R | 每个模块只生成自己的 R 类 | 减少 R.java 重编译 |
| implementation | 不传递依赖，减少重编译范围 | 改一个模块不触发下游重编译 |

**buildSrc 的问题：**
```
buildSrc/ 中任何改动 → 整个项目所有 Task 缓存失效 → 全量重编译

替代方案: composite build (build-logic/)
build-logic/ 中的改动 → 只影响使用了该 plugin 的模块
```

**Configuration Cache 原理：**
```
首次构建: 执行 settings.gradle + build.gradle → 序列化 Task 图到缓存
后续构建: 直接反序列化 Task 图 → 跳过配置阶段
失效条件: build.gradle 文件变化、gradle.properties 变化、环境变量变化
```

### 实战层

**分析构建瓶颈：**
```bash
# Build Scan（推荐，可视化分析）
./gradlew assembleDebug --scan

# 本地 Profile 报告
./gradlew assembleDebug --profile
# 输出 build/reports/profile/profile-xxx.html

# 查看 Task 执行时间
./gradlew assembleDebug --info | grep "Task.*took"
```

**常见瓶颈及解决：**
```
1. KAPT 慢 → 迁移到 KSP
2. 资源处理慢 → 开启 resourceShrinking、减少无用资源
3. Kotlin 编译慢 → 升级 Kotlin 版本、开启增量编译
4. 配置阶段慢 → Configuration Cache + 避免配置阶段执行命令
5. 依赖下载慢 → 使用国内镜像、开启 dependency cache
```

**远程构建缓存：**
```kotlin
// settings.gradle.kts
buildCache {
    local { isEnabled = true }
    remote<HttpBuildCache> {
        url = uri("https://cache.example.com/")
        isPush = System.getenv("CI") != null  // CI 推送缓存
        credentials {
            username = "cache-user"
            password = System.getenv("CACHE_PASSWORD")
        }
    }
}
// 团队成员和 CI 共享构建缓存
```

**模块化对构建速度的影响：**
- 模块越多，并行度越高
- 但模块间 api 依赖会降低并行度
- 最优：扁平依赖结构（feature 模块互不依赖）

### 延伸问题

- [[Gradle构建生命周期与Task依赖]]
- [[大型项目Gradle模块拆分策略]]
- [[KSP与KAPT对比]]

## 记忆锚点

Gradle 加速三板斧：parallel（并行）+ cache（缓存）+ configuration-cache（跳过配置）。分析用 --scan，最大的坑是 buildSrc 和 api 依赖传递。
