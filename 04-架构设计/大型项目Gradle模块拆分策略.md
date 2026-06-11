---
tags: [android, 面试, 架构, gradle, A]
difficulty: A
frequency: medium
---

# 大型项目 Gradle 模块拆分策略

## 问题

> 大型 Android 项目如何决定模块拆分粒度？按功能拆还是按层拆？模块数量多了之后如何管理编译速度？

## 核心答案

推荐按功能（feature）+ 按层（layer）混合拆分：水平分层（app/feature/domain/data/core）确保依赖方向，垂直按业务切分 feature 模块实现团队并行开发。模块粒度以"独立发布/独立测试"为标准。编译速度通过 Gradle 缓存、并行编译、避免不必要的 api 依赖传递来优化。

## 深入解析

### 原理层

**混合拆分模型：**
```
app/                          ← 壳模块，组装 feature
├── feature/
│   ├── feature-home/         ← 首页业务
│   ├── feature-user/         ← 用户业务
│   ├── feature-order/        ← 订单业务
│   └── feature-payment/      ← 支付业务
├── domain/
│   ├── domain-user/          ← 用户领域（UseCase + Entity）
│   └── domain-order/         ← 订单领域
├── data/
│   ├── data-user/            ← 用户数据（Repository 实现）
│   └── data-order/           ← 订单数据
└── core/
    ├── core-network/         ← 网络基础设施
    ├── core-database/        ← 数据库基础设施
    ├── core-ui/              ← 公共 UI 组件
    ├── core-common/          ← 工具类
    └── core-testing/         ← 测试工具
```

**拆分决策标准：**

| 信号 | 动作 |
|------|------|
| 两个团队同时修改同一模块 | 拆分 |
| 模块编译时间 > 30s | 考虑拆分 |
| 模块代码 > 10000 行 | 考虑拆分 |
| 模块只有一个类在用 | 不拆（过度拆分） |
| 改一行代码触发大量重编译 | 检查 api 依赖是否过多 |

**编译速度与模块关系：**
```
模块 A (api 依赖) → 模块 B → 模块 C
A 的公共 API 变化 → B 重编译 → C 重编译（传递）

模块 A (implementation 依赖) → 模块 B → 模块 C  
A 的内部变化 → B 重编译 → C 不受影响 ✅
```

**Gradle 并行编译：**
```
无依赖关系的模块可以并行编译：
feature-home ─┐
feature-user ─┼─ 并行编译
feature-order─┘
      ↓ (都完成后)
    app 模块编译
```

### 实战层

**Convention Plugin 统一配置：**
```kotlin
// build-logic/convention/src/main/kotlin/AndroidFeaturePlugin.kt
class AndroidFeaturePlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply {
            apply("com.android.library")
            apply("org.jetbrains.kotlin.android")
            apply("dagger.hilt.android.plugin")
        }
        extensions.configure<LibraryExtension> {
            compileSdk = 34
            defaultConfig { minSdk = 24 }
            // 统一配置...
        }
    }
}

// feature-home/build.gradle.kts
plugins {
    id("app.android.feature")  // 一行搞定所有配置
}
```

**编译加速策略：**
1. `implementation` 优先于 `api`（减少传递重编译）
2. 开启 Gradle Build Cache（`org.gradle.caching=true`）
3. 开启并行编译（`org.gradle.parallel=true`）
4. Configuration Cache（`org.gradle.configuration-cache=true`）
5. 模块 ABI 稳定（公共接口不频繁变化）
6. 避免 buildSrc（改动触发全量重编译），用 composite build

**Now in Android 参考架构：**
- Google 官方示例项目，展示了推荐的模块化方案
- 约 30 个模块，feature/domain/data/core 分层
- 使用 Convention Plugin + Version Catalog

### 延伸问题

- [[模块化依赖管理策略]]
- [[Gradle构建生命周期与Task依赖]]
- [[Gradle构建加速]]

## 记忆锚点

模块拆分 = 水平分层（feature/domain/data/core）+ 垂直切业务。粒度标准：能独立编译测试。编译速度靠 implementation 隔离 + 并行 + 缓存。
