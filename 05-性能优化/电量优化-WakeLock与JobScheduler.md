---
tags:
  - 性能优化
  - 电量优化
  - WakeLock
  - JobScheduler
difficulty: A
frequency: medium
---

# 电量优化-WakeLock与JobScheduler

## 问题

> 你们 App 做过电量优化吗？WakeLock 使用有什么注意事项？Doze 模式下怎么保证任务执行？

## 核心答案（30秒版本）

电量优化核心是减少 CPU 唤醒次数和持续时间。WakeLock 必须配合 timeout 使用并在 finally 中释放，避免永久持锁导致设备无法休眠。Doze 模式下普通 AlarmManager 和 JobScheduler 会被延迟，需要用 `setExactAndAllowWhileIdle()` 或 FCM 高优先级消息突破限制。推荐用 WorkManager 统一管理后台任务，它内部根据 API Level 自动选择 JobScheduler/AlarmManager/GCMNetworkManager。

## 深入解析

### 原理层（源码级）

**Android 电量消耗模型：**

```
电量消耗 = CPU 唤醒时间 × CPU 功率
         + 网络传输量 × 网络功率
         + GPS 使用时间 × GPS 功率
         + 屏幕亮度 × 屏幕功率
         + WakeLock 持有时间 × 对应组件功率

关键：CPU 从休眠到唤醒的切换本身就消耗大量电量
      → 批量处理优于频繁唤醒
```

**WakeLock 层级与源码：**

```java
// frameworks/base/core/java/android/os/PowerManager.java
public final class PowerManager {
    // WakeLock 类型（从高到低）
    public static final int FULL_WAKE_LOCK = 0x0000001a;      // CPU+屏幕+键盘
    public static final int SCREEN_BRIGHT_WAKE_LOCK = 0x0000000a; // CPU+屏幕亮
    public static final int SCREEN_DIM_WAKE_LOCK = 0x00000006;    // CPU+屏幕暗
    public static final int PARTIAL_WAKE_LOCK = 0x00000001;       // 仅 CPU（最常用）

    public WakeLock newWakeLock(int levelAndFlags, String tag) {
        // tag 用于 Battery Historian 中定位问题
        return new WakeLock(levelAndFlags, tag);
    }
}

// WakeLock 内部实现
public final class WakeLock {
    private int mInternalCount; // 引用计数
    private boolean mHeld;

    public void acquire(long timeout) {
        synchronized (mToken) {
            mInternalCount++;
            mHeld = true;
            // 通过 Binder 调用 PowerManagerService
            mService.acquireWakeLock(mToken, mFlags, mTag, timeout);
        }
        // timeout 后自动释放
        mHandler.postDelayed(mReleaser, timeout);
    }

    public void release() {
        synchronized (mToken) {
            mInternalCount--;
            if (mInternalCount == 0) {
                mService.releaseWakeLock(mToken);
                mHeld = false;
            }
        }
    }
}
```

**Doze 模式状态机：**

```
屏幕关闭 + 静止不动 + 未充电
    ↓ (30分钟)
IDLE_PENDING → 进入 Doze 轻度休眠
    ↓ (逐步延长间隔: 1h → 2h → 4h → 6h)
IDLE → 深度 Doze
    ↓ (维护窗口)
IDLE_MAINTENANCE → 短暂允许网络/Job 执行
    ↓
回到 IDLE

Doze 限制：
- 网络访问暂停
- AlarmManager 延迟（非 exact）
- JobScheduler 延迟
- WiFi 扫描暂停
- SyncAdapter 暂停
- WakeLock 被忽略（PARTIAL_WAKE_LOCK 除外的都无效）
```

**JobScheduler 源码调度逻辑：**

```java
// frameworks/base/services/core/java/com/android/server/job/JobSchedulerService.java
public class JobSchedulerService extends SystemService {

    // 判断 Job 是否满足执行条件
    private boolean isReadyToBeExecutedLocked(JobStatus job) {
        // 检查所有约束条件
        return job.isConstraintSatisfied(CONSTRAINT_TIMING_DELAY)
            && job.isConstraintSatisfied(CONSTRAINT_DEADLINE)
            && job.isConstraintSatisfied(CONSTRAINT_CONNECTIVITY)
            && job.isConstraintSatisfied(CONSTRAINT_IDLE)
            && job.isConstraintSatisfied(CONSTRAINT_CHARGING)
            && job.isConstraintSatisfied(CONSTRAINT_BATTERY_NOT_LOW);
    }

    // Doze 模式下的处理
    void maybeRunPendingJobsLocked() {
        if (mDeviceIdleMode) {
            // Doze 模式下，只执行标记为 expedited 的 Job
            // 或等待维护窗口
            return;
        }
        assignJobsToContextsLocked();
    }
}
```

### 实战层（项目经验）

**1. WakeLock 安全使用模式：**

```kotlin
// 错误用法 - 可能永久持锁
fun badWakeLock() {
    val wakeLock = powerManager.newWakeLock(
        PowerManager.PARTIAL_WAKE_LOCK, "MyApp:MyWakeLock")
    wakeLock.acquire() // 无 timeout！危险！
    doWork() // 如果抛异常，锁永远不会释放
}

// 正确用法 - timeout + try-finally
fun safeWakeLock() {
    val wakeLock = powerManager.newWakeLock(
        PowerManager.PARTIAL_WAKE_LOCK, "MyApp:SyncWork")
    try {
        wakeLock.acquire(10 * 60 * 1000L) // 最多持有 10 分钟
        doWork()
    } finally {
        if (wakeLock.isHeld) {
            wakeLock.release()
        }
    }
}

// 最佳实践 - 使用 WorkManager 自动管理
class SyncWorker(context: Context, params: WorkerParameters)
    : CoroutineWorker(context, params) {
    // WorkManager 内部自动管理 WakeLock
    override suspend fun doWork(): Result {
        return try {
            syncData()
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

**2. WorkManager 替代方案（推荐）：**

```kotlin
// 定义约束条件
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)
    .setRequiresCharging(false)
    .build()

// 一次性任务
val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(constraints)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork(
    "data_sync",
    ExistingWorkPolicy.REPLACE,
    syncRequest
)

// 周期性任务（最小间隔 15 分钟）
val periodicRequest = PeriodicWorkRequestBuilder<CleanupWorker>(
    1, TimeUnit.HOURS,
    15, TimeUnit.MINUTES // flex interval
).setConstraints(constraints).build()
```

**3. 电量消耗分析工具链：**

```bash
# 1. 重置电量统计
adb shell dumpsys batterystats --reset

# 2. 断开 USB（避免充电影响统计）
adb shell dumpsys battery unplug

# 3. 执行测试场景...

# 4. 导出电量数据
adb shell dumpsys batterystats > batterystats.txt
adb bugreport > bugreport.zip

# 5. 使用 Battery Historian 分析
# https://bathist.ef.lc/ (在线版)
# 关注：WakeLock 持有时间、网络唤醒次数、Alarm 触发频率
```

**4. Doze 模式适配：**

```kotlin
// 检查是否在电池优化白名单中
fun isIgnoringBatteryOptimizations(context: Context): Boolean {
    val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
    return pm.isIgnoringBatteryOptimizations(context.packageName)
}

// 引导用户加入白名单（谨慎使用，可能被 Google Play 拒绝）
fun requestIgnoreBatteryOptimization(activity: Activity) {
    if (!isIgnoringBatteryOptimizations(activity)) {
        val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
            data = Uri.parse("package:${activity.packageName}")
        }
        activity.startActivity(intent)
    }
}

// Doze 模式下的精确闹钟（慎用）
val alarmManager = getSystemService(Context.ALARM_SERVICE) as AlarmManager
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    alarmManager.setExactAndAllowWhileIdle(
        AlarmManager.ELAPSED_REALTIME_WAKEUP,
        SystemClock.elapsedRealtime() + intervalMs,
        pendingIntent
    )
}
```

**5. 网络请求批量化：**

```kotlin
// 将多个小请求合并为批量请求，减少无线电唤醒次数
class BatchRequestManager {
    private val pendingRequests = mutableListOf<Request>()
    private val batchInterval = 30_000L // 30秒批量发送

    fun addRequest(request: Request) {
        pendingRequests.add(request)
        scheduleBatch()
    }

    private fun scheduleBatch() {
        handler.removeCallbacks(batchRunnable)
        if (pendingRequests.size >= MAX_BATCH_SIZE) {
            executeBatch() // 达到上限立即发送
        } else {
            handler.postDelayed(batchRunnable, batchInterval)
        }
    }

    private fun executeBatch() {
        val batch = pendingRequests.toList()
        pendingRequests.clear()
        // 一次网络连接发送所有请求
        apiService.batchRequest(batch)
    }
}
```

### 延伸问题

- [[ANR分析与治理]] — 后台任务超时导致的 ANR？
- [[网络优化-连接复用与协议升级]] — 网络请求对电量的影响？
- [[启动优化-耗时分析与优化手段]] — 启动阶段的后台任务调度？
- [[存储优化-IO调度与MMKV原理]] — IO 操作对电量的影响？

## 记忆锚点

电量优化 = 减少唤醒（批量化）+ 安全持锁（timeout + finally）+ 适配 Doze（WorkManager 统一调度）；WakeLock 三原则：必须 timeout、必须 finally 释放、优先用 WorkManager 替代。
