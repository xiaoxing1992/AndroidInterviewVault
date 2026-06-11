---
tags: [handler-thread, intent-service, work-manager, background-task, lifecycle]
difficulty: A
frequency: medium
---

# HandlerThread vs IntentService vs WorkManager 对比

## 问题

"HandlerThread、IntentService、WorkManager 三者的适用场景和区别是什么？为什么 IntentService 在 Android 8.0 后被废弃？WorkManager 是怎么保证任务可靠执行的？后台任务方案是怎么演进的？"

## 核心答案（30秒版本）

HandlerThread 是带 Looper 的串行任务线程，适合按序处理消息；IntentService 是 HandlerThread + Service 的封装，处理完自动停止，但 Android 8.0 后台限制导致其不可靠被废弃；WorkManager 是 Jetpack 提供的后台任务调度框架，整合了 JobScheduler/AlarmManager/BroadcastReceiver，支持持久化、约束、重试、链式任务，是当前推荐方案。演进路径：Service → IntentService → JobScheduler → WorkManager。

## 深入解析

### 原理层（源码级）

**HandlerThread 源码：**

```java
public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();                // 准备 Looper
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();                  // 唤醒等待 getLooper 的线程
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();                    // 进入消息循环
        mTid = -1;
    }

    public Looper getLooper() {
        // 阻塞等待 Looper 创建完成
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try { wait(); } catch (InterruptedException e) { }
            }
        }
        return mLooper;
    }

    public boolean quit() {
        // 处理完所有消息后退出（推荐）
        Looper looper = getLooper();
        if (looper != null) { looper.quit(); return true; }
        return false;
    }

    public boolean quitSafely() {
        // 立即退出，丢弃未处理消息
        // ...
    }
}
```

**HandlerThread 使用：**

```kotlin
class FileWriteService {
    private val handlerThread = HandlerThread("file-writer").apply { start() }
    private val handler = Handler(handlerThread.looper)

    fun writeAsync(file: File, data: String) {
        handler.post {
            file.appendText(data)  // 串行写入，避免并发冲突
        }
    }

    fun destroy() {
        handlerThread.quitSafely()
    }
}
```

**IntentService 源码（关键点）：**

```java
public abstract class IntentService extends Service {
    private volatile Looper mServiceLooper;
    private volatile ServiceHandler mServiceHandler;

    @Override
    public void onCreate() {
        // 内部就是 HandlerThread
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();
        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    private final class ServiceHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent) msg.obj);  // 子类实现的处理逻辑
            stopSelf(msg.arg1);                // 处理完自动停止
        }
    }

    @Override
    public void onDestroy() {
        mServiceLooper.quit();  // 清理
    }
}
```

**为什么 IntentService 被废弃？**

```
Android 8.0（API 26）后台执行限制：
1. 应用进入后台后，无法启动后台 Service（startService 失败）
   → IllegalStateException: Not allowed to start service Intent
2. 即使能启动，后台执行时间限制为 1 分钟
3. Doze 模式下，CPU/网络受限

JobIntentService（兼容方案，已废弃）：
- 内部用 JobScheduler（API 26+）或 startWakefulService（API < 26）
- 但仍有局限性

最终方案：WorkManager
```

**WorkManager 架构：**

```
                    WorkManager API
                          ↓
                    WorkManagerImpl
                          ↓
        ┌──────────────────┴──────────────────┐
        ↓                                      ↓
   WorkDatabase                        Schedulers
   (Room 持久化)                       ┌────┴────┐
   - WorkSpec                          ↓         ↓
   - WorkTag                  GreedyScheduler  SystemJobScheduler
   - Dependency               (前台立即执行)   (API 23+ JobScheduler)
                                              SystemAlarmScheduler
                                              (API < 23 AlarmManager)
```

**WorkManager 源码核心：**

```kotlin
// 1. 定义 Worker
class UploadWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            uploadData()
            Result.success()
        } catch (e: IOException) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}

// 2. 构建 WorkRequest（约束 + 重试策略）
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.UNMETERED)  // 仅 WiFi
    .setRequiresCharging(true)                       // 充电时
    .setRequiresBatteryNotLow(true)                  // 电量充足
    .setRequiresStorageNotLow(true)
    .build()

val workRequest = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(constraints)
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.SECONDS)
    .setInputData(workDataOf("file_id" to "abc123"))
    .addTag("upload")
    .build()

// 3. 提交（持久化到 Room）
WorkManager.getInstance(context).enqueue(workRequest)

// 4. 链式任务
WorkManager.getInstance(context)
    .beginWith(compressWork)
    .then(uploadWork)
    .then(notifyWork)
    .enqueue()

// 5. 唯一任务（避免重复入队）
WorkManager.getInstance(context).enqueueUniqueWork(
    "sync-user-data",
    ExistingWorkPolicy.KEEP,  // 已有同名任务则保留
    syncWork
)
```

**WorkManager 持久化保证：**

```
任务流程：
1. enqueue() → 写入 WorkDatabase（SQLite）
2. 根据约束选择 Scheduler
3. 满足约束 → 执行 Worker
4. 失败 → 根据 BackoffPolicy 重试
5. 进程被杀 / 设备重启 → 启动时从 DB 恢复任务

可靠性等级：
- OneTimeWorkRequest: 至少执行一次
- PeriodicWorkRequest: 周期性执行（最小间隔 15 分钟）
- 即使应用未运行也会执行（满足约束时）
```

### 实战层（项目经验）

**演进路径与选型：**

| 方案 | 适用版本 | 适用场景 | 现状 |
|------|---------|---------|------|
| Thread + Handler | 全版本 | 简单异步 | 仍可用，但推荐协程 |
| HandlerThread | 全版本 | 串行任务（日志、SP写入） | 仍可用 |
| AsyncTask | API 3+ | 简单后台任务 | API 30 废弃 |
| Service | 全版本 | 长时任务 | 受后台限制 |
| IntentService | API 3+ | 队列式后台任务 | API 30 废弃 |
| JobScheduler | API 21+ | 系统级调度 | 推荐用 WorkManager 封装 |
| Firebase JobDispatcher | 2018前 | GMS 设备 | 已废弃 |
| WorkManager | API 14+ | 可靠后台任务 | 推荐 |
| 协程 + ViewModel | 全版本 | UI 相关异步 | 推荐 |
| 前台 Service | 全版本 | 用户感知任务 | 必须显示通知 |

**实战决策树：**

```
后台任务需求？
├─ UI 相关（页面内异步）→ 协程 + viewModelScope
├─ 用户感知（音乐播放、导航）→ 前台 Service + 通知
├─ 即时执行 + 不持久化（一次性事件）→ Thread / Coroutine
├─ 串行处理（顺序写日志）→ HandlerThread + Handler / Coroutine + Channel
└─ 可延迟 + 需可靠执行
   ├─ 周期任务（定时同步）→ WorkManager.PeriodicWorkRequest
   ├─ 一次性 + 网络/充电约束 → WorkManager.OneTimeWorkRequest
   └─ 链式任务（A→B→C）→ WorkManager.beginWith().then()
```

**实战场景示例：**

```kotlin
// 场景1：日志写入（HandlerThread）
object Logger {
    private val thread = HandlerThread("logger", Process.THREAD_PRIORITY_BACKGROUND)
        .apply { start() }
    private val handler = Handler(thread.looper)
    private val logFile = File(context.filesDir, "app.log")

    fun log(tag: String, msg: String) {
        handler.post {
            // 串行写入，无需加锁
            logFile.appendText("${System.currentTimeMillis()} $tag: $msg\n")
        }
    }
}

// 场景2：图片上传（WorkManager）
class ImageUploadWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        val imageUri = inputData.getString("uri") ?: return@withContext Result.failure()

        try {
            val response = uploadImage(Uri.parse(imageUri))
            // 输出数据传给下一个 Worker
            Result.success(workDataOf("upload_url" to response.url))
        } catch (e: IOException) {
            // 网络异常重试
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// 触发上传
fun scheduleUpload(uri: Uri) {
    val request = OneTimeWorkRequestBuilder<ImageUploadWorker>()
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .setInputData(workDataOf("uri" to uri.toString()))
        .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
        .build()

    WorkManager.getInstance(context).enqueueUniqueWork(
        "upload_${uri.lastPathSegment}",
        ExistingWorkPolicy.REPLACE,
        request
    )
}

// 场景3：周期性数据同步
class SyncWorker(context: Context, params: WorkerParameters) :
    CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        repository.syncToServer()
        return Result.success()
    }
}

// 应用启动时注册（每 15 分钟，需联网）
fun scheduleSync() {
    val request = PeriodicWorkRequestBuilder<SyncWorker>(15, TimeUnit.MINUTES)
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .build()

    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "data_sync",
        ExistingPeriodicWorkPolicy.KEEP,
        request
    )
}
```

**WorkManager 踩坑：**

1. **进程被杀后约束未触发**：部分国产 ROM（小米、华为）会杀掉应用进程，导致 WorkManager 无法被唤醒
   - 缓解：申请白名单 / 关闭电池优化

2. **PeriodicWorkRequest 最小 15 分钟**：低于这个间隔的任务会被忽略
   - 替代：用 OneTimeWorkRequest + setInitialDelay 实现自调度

3. **大数据传递**：Data 限制 10KB
   - 替代：传递文件路径或数据库 ID，Worker 内部读取

4. **测试困难**：依赖系统调度
   - 使用 WorkManagerTestInitHelper + TestDriver 加速测试

```kotlin
@Before
fun setup() {
    val config = Configuration.Builder()
        .setMinimumLoggingLevel(Log.DEBUG)
        .setExecutor(SynchronousExecutor())
        .build()
    WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
}

@Test
fun testWorker() {
    val request = OneTimeWorkRequestBuilder<MyWorker>().build()
    val workManager = WorkManager.getInstance(context)
    workManager.enqueue(request).result.get()

    // 立即满足约束并执行
    val testDriver = WorkManagerTestInitHelper.getTestDriver(context)
    testDriver?.setAllConstraintsMet(request.id)

    val workInfo = workManager.getWorkInfoById(request.id).get()
    assertEquals(WorkInfo.State.SUCCEEDED, workInfo.state)
}
```

### 延伸问题

- [[协程vs线程vsRxJava对比]] — 协程对后台任务方案的影响
- [[协程作用域与结构化并发]] — CoroutineWorker 的作用域管理
- [[线程池原理与参数调优]] — WorkManager 内部的线程池配置

## 记忆锚点

HandlerThread 是"专属流水线工人"，IntentService 是"自带遣散机制的流水线"（处理完自动下班，但被 8.0 政策辞退），WorkManager 是"现代任务调度中心"——综合考虑电量、网络、充电状态后再派活，还能保存任务清单防丢失。
