---
tags: [android, 面试, 工程化, gradle, A]
difficulty: A
frequency: high
---

# Gradle 构建生命周期与 Task 依赖

## 问题

> Gradle 构建的三个阶段是什么？Task 之间的依赖关系如何定义？doFirst/doLast 的执行时机是什么？如何自定义 Gradle Task？

## 核心答案

Gradle 构建分三个阶段：初始化（确定参与构建的项目）→ 配置（构建所有项目的 Task DAG）→ 执行（按依赖顺序执行目标 Task）。Task 依赖通过 `dependsOn`、`mustRunAfter`、`finalizedBy` 定义。配置阶段所有 Task 的配置代码都会执行（无论是否被需要），只有执行阶段才运行 Task 的 action（doFirst/doLast）。

## 深入解析

### 原理层

**三个阶段：**
```
1. 初始化阶段 (Initialization)
   → 执行 settings.gradle.kts
   → 确定哪些项目参与构建（include）
   → 创建 Project 实例

2. 配置阶段 (Configuration)
   → 执行所有项目的 build.gradle.kts
   → 创建和配置所有 Task（但不执行 action）
   → 构建 Task 依赖图（DAG）
   → 注意：所有 build.gradle 都会执行，即使只构建一个模块

3. 执行阶段 (Execution)
   → 根据命令行指定的 Task，按 DAG 顺序执行
   → 执行 Task 的 doFirst → action → doLast
```

**Task 依赖关系：**
```kotlin
tasks.register("taskA") {
    doLast { println("A") }
}

tasks.register("taskB") {
    dependsOn("taskA")  // B 依赖 A，A 先执行
    doLast { println("B") }
}

tasks.register("taskC") {
    mustRunAfter("taskA")  // 如果 A 和 C 都要执行，A 先执行（弱依赖）
    doLast { println("C") }
}

tasks.register("taskD") {
    finalizedBy("taskE")  // D 执行后一定执行 E（无论 D 成功失败）
    doLast { println("D") }
}
```

**配置阶段 vs 执行阶段：**
```kotlin
tasks.register("myTask") {
    // 这里是配置阶段代码（每次构建都执行）
    val inputFile = file("input.txt")
    
    doLast {
        // 这里是执行阶段代码（只有 myTask 被执行时才运行）
        println(inputFile.readText())
    }
}

// ❌ 常见错误：在配置阶段做耗时操作
tasks.register("badTask") {
    val result = Runtime.getRuntime().exec("slow-command")  // 配置阶段就执行了！
}
```

**自定义 Task 类：**
```kotlin
abstract class GenerateReportTask : DefaultTask() {
    @get:InputFile
    abstract val inputFile: RegularFileProperty
    
    @get:OutputFile
    abstract val outputFile: RegularFileProperty
    
    @TaskAction
    fun generate() {
        val data = inputFile.get().asFile.readText()
        outputFile.get().asFile.writeText(processData(data))
    }
}

tasks.register<GenerateReportTask>("generateReport") {
    inputFile.set(file("data.json"))
    outputFile.set(file("build/report.html"))
}
```

**Android 构建 Task 链（简化）：**
```
preBuild → generateSources → compileKotlin → compileJava 
→ mergeResources → processManifest → packageResources 
→ dexBuilder → mergeExtDex → mergeDex 
→ packageDebug → assembleDebug
```

### 实战层

**避免配置阶段耗时：**
```kotlin
// ✅ 使用 Provider API 延迟计算
val gitHash = providers.exec { commandLine("git", "rev-parse", "--short", "HEAD") }
    .standardOutput.asText.map { it.trim() }

// 只在需要时才执行 git 命令
tasks.register("printHash") {
    doLast { println(gitHash.get()) }
}
```

**增量构建（UP-TO-DATE）：**
- 声明 `@InputFiles` 和 `@OutputFiles`
- Gradle 比较输入输出的 hash，未变化则跳过（UP-TO-DATE）
- 这就是为什么自定义 Task 要正确声明输入输出

**Build Scan 分析：**
```bash
./gradlew assembleDebug --scan
# 生成构建分析报告，查看各阶段耗时和 Task 执行时间
```

### 延伸问题

- [[Gradle构建加速]]
- [[Transform API与字节码插桩]]
- [[大型项目Gradle模块拆分策略]]

## 记忆锚点

Gradle 三阶段：初始化（谁参与）→ 配置（建 Task 图，所有 build.gradle 都跑）→ 执行（按图跑 Task）。配置阶段别干活，执行阶段才是正事。
