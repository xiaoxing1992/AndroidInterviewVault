---
tags: [android, 面试, kotlin, 编译器, A]
difficulty: A
frequency: medium
---

# KSP 与 KAPT 对比

## 问题

> KSP 和 KAPT 分别是什么？为什么 Google 推荐从 KAPT 迁移到 KSP？两者在编译速度和能力上有什么差异？

## 核心答案

KAPT（Kotlin Annotation Processing Tool）通过生成 Java Stub 来复用 Java 的 APT 机制，需要额外的 stub 生成阶段，编译慢。KSP（Kotlin Symbol Processing）直接分析 Kotlin 编译器的符号信息，跳过 stub 生成，编译速度提升 2x+，且能访问 Kotlin 特有的语法信息（如 suspend、sealed、extension）。

## 深入解析

### 原理层

**KAPT 工作流程：**
```
Kotlin 源码 → kotlinc 生成 Java Stubs → javac APT 处理注解 → 生成代码 → 编译
```
- 必须生成 Java Stub（将 Kotlin 代码转为 Java 声明），耗时占编译的 1/3
- 注解处理器只能看到 Java 视角的 API，丢失 Kotlin 特有信息
- 基于 `javax.annotation.processing` API

**KSP 工作流程：**
```
Kotlin 源码 → kotlinc 前端解析 → KSP 处理符号 → 生成代码 → 继续编译
```
- 直接挂载在 Kotlin 编译器前端，无需 stub 生成
- 通过 `SymbolProcessor` 接口访问 `KSNode` 树
- 支持增量处理（只处理变化的文件）

**API 对比：**
```kotlin
// KAPT - Java APT 风格
class MyProcessor : AbstractProcessor() {
    override fun process(annotations: Set<TypeElement>, 
                        roundEnv: RoundEnvironment): Boolean { ... }
}

// KSP - Kotlin 原生风格
class MyProcessor : SymbolProcessor {
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver.getSymbolsWithAnnotation("com.example.MyAnnotation")
        symbols.filterIsInstance<KSClassDeclaration>().forEach { classDecl ->
            // 可以直接访问 Kotlin 特有信息
            classDecl.getSealedSubclasses()
            classDecl.primaryConstructor
        }
        return emptyList() // 返回无法处理的符号
    }
}
```

**性能对比（官方数据）：**
| 指标 | KAPT | KSP |
|------|------|-----|
| 全量编译 | 基准 | 快 2x |
| 增量编译 | 不支持/有限 | 原生支持 |
| Stub 生成 | 必须 | 无需 |
| Kotlin 信息 | 丢失 | 完整保留 |

### 实战层

**主流库迁移状态：**
- Room → 已支持 KSP（推荐）
- Hilt/Dagger → 已支持 KSP
- Moshi → 已支持 KSP
- Glide → 已支持 KSP
- Retrofit → 不需要注解处理

**迁移步骤：**
```kotlin
// build.gradle.kts
// 移除
// id("kotlin-kapt")
// kapt("androidx.room:room-compiler:2.6.0")

// 替换为
id("com.google.devtools.ksp")
ksp("androidx.room:room-compiler:2.6.0")
```

**坑点：**
- KSP 不支持所有 KAPT 能做的事（如修改已有类的字节码）
- 部分老旧库仍只支持 KAPT，需要 KAPT 和 KSP 共存
- KSP 的增量处理需要处理器正确声明依赖关系

### 延伸问题

- [[Hilt依赖注入原理]]
- [[Room编译时生成原理]]
- [[Transform API与字节码插桩]]

## 记忆锚点

KAPT = 先翻译成 Java 再用 Java 的注解处理器（绕路）。KSP = 直接读 Kotlin 编译器的语法树（直达）。快 2 倍因为省了 stub 生成。
