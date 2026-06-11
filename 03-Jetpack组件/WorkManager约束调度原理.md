---
tags: [android, 面试, jetpack, workmanager, A]
difficulty: A
frequency: medium
---

# WorkManager 约束调度原理

## 问题

> WorkManager 是如何保证任务在约束条件满足时执行的？它底层使用了什么机制？与 AlarmManager、JobScheduler 的关系是什么？

## 核心答案

WorkManager 是对系统调度 API 的统一封装，根据 API 级别自动选择底层实现：API 23+ 使用 JobScheduler，低版本使用 AlarmManager + BroadcastReceiver。任务信息持久化到 Room 数据库，约束条件（网络、电量、存储）通过系统回调触发检查，满足时执行任务。即使 App 被杀或设备重启，任务也能恢复执行。

## 深入解析

### 原理层

**架构分层：**
```
WorkManager API (统一接口)
    │
    ├── WorkDatabase (Room) — 持久化任务信息
    │
    ├── Scheduler (调度器选择)
    │   ├── GreedyScheduler — 立即可执行的任务
    │   ├── SystemJobScheduler (API 23+) — 委托给 JobScheduler
    │   └── SystemAlarmScheduler (API < 23) — AlarmManager + BroadcastReceiver
    │
    └── Worker / CoroutineWorker — 实际执行逻辑
```

**任务入队流程：**
```kotlin
val request = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .setRequiresBatteryNotLow(true)
        .build())
    .build()

WorkManager.getInstance(context).enqueue(request)
// 1. 将 WorkSpec 写入 Room 数据库
// 2. 通知 Scheduler 调度
// 3. Scheduler 根据约束注册系统回调
```

**约束检查机制：**
```java
// ConstraintTracker 监听系统状态变化
class NetworkStateTracker : ConstraintTracker<NetworkState> {
    // 注册 ConnectivityManager.NetworkCallback
    // 网络状态变化时通知 WorkManager
}

class BatteryNotLowTracker : ConstraintTracker<Boolean> {
    // 注册 ACTION_BATTERY_CHANGED BroadcastReceiver
}

// 所有约束满足 → 触发 Worker 执行
```

**任务持久化（Room 数据库）：**
```
WorkDatabase
├── WorkSpec 表 — 任务定义（类名、约束、状态、重试策略）
├── Dependency 表 — 任务链依赖关系
├── WorkTag 表 — 标签
└── WorkProgress 表 — 进度
```

**任务状态机：**
```
ENQUEUED → RUNNING → SUCCEEDED
                  → FAILED
                  → CANCELLED
BLOCKED (等待前置任务完成)
```

**Worker 执行环境：**
```kotlin
// Worker 在后台线程执行（Executor 线程池）
class UploadWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        // 在后台线程执行
        return Result.success()  // 或 retry() / failure()
    }
}

// CoroutineWorker 在 Dispatchers.Default 执行
class UploadWorker(...) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        // 协程环境，可以调用 suspend 函数
    }
}
```

### 实战层

- **任务链**：`beginWith(A).then(B).then(C).enqueue()` — A 完成后执行 B，B 完成后执行 C
- **唯一任务**：`enqueueUniqueWork` 避免重复入队（KEEP/REPLACE/APPEND 策略）
- **周期任务**：`PeriodicWorkRequest` 最小间隔 15 分钟（系统限制）
- **进度上报**：`setProgress(Data)` + `getWorkInfoByIdLiveData` 观察进度
- **坑点**：国产 ROM 可能杀后台导致 WorkManager 不可靠，需要引导用户关闭电池优化
- **测试**：`WorkManagerTestInitHelper` + `TestWorkerBuilder` 单元测试

### 延伸问题

- [[电量优化-WakeLock与JobScheduler]]
- [[HandlerThread-IntentService-WorkManager对比]]
- [[Lifecycle感知组件实现]]

## 记忆锚点

WorkManager = Room 持久化 + 系统调度器（JobScheduler/AlarmManager）+ 约束监听。杀进程重启都能恢复，因为任务存在数据库里。
