---
tags:
  - 性能优化
  - ANR
  - 稳定性
  - traces
difficulty: S
frequency: high
---

# ANR分析与治理

## 问题

> ANR 的触发条件是什么？线上 ANR 怎么监控和治理？traces.txt 怎么分析？

## 核心答案（30秒版本）

ANR 触发条件：Input 事件 5 秒无响应、BroadcastReceiver 前台 10 秒/后台 60 秒超时、Service 前台 20 秒/后台 200 秒超时、ContentProvider 10 秒超时。分析通过 `/data/anr/traces.txt` 查看主线程堆栈，关注 `BLOCKED`/`WAITING` 状态和锁信息。线上监控方案：FileObserver 监听 traces 文件生成、WatchDog 线程定时检测主线程响应、ANRWatchDog 开源方案。治理核心是将耗时操作移出主线程。

## 深入解析

### 原理层（源码级）

**ANR 触发机制（以 Input 为例）：**

```java
// frameworks/base/services/core/java/com/android/server/input/InputDispatcher.cpp
// 简化的 ANR 检测逻辑

// InputDispatcher 分发事件时启动计时
void InputDispatcher::dispatchOnce() {
    // 找到目标窗口
    InputTarget target = findFocusedWindowTarget();

    if (target == null) {
        // 无焦点窗口，等待
        if (currentTime - mNoFocusedWindowTimeoutTime > 5000ms) {
            onAnrLocked(/* no focused window */);
        }
    } else {
        // 发送事件到目标窗口
        dispatchEventLocked(target, event);
        // 启动 5 秒超时计时
        mAnrTimer.start(5000ms);
    }
}

// 窗口处理完事件后回调
void InputDispatcher::handleFinishedSignal() {
    // 取消超时计时
    mAnrTimer.cancel();
}

// 超时触发
void InputDispatcher::onAnrLocked(const sp<Connection>& connection) {
    // 通知 AMS 发生 ANR
    mPolicy->notifyANR(connection->inputChannel->getToken());
}
```

**AMS 处理 ANR：**

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
void appNotResponding(ProcessRecord app, String annotation) {
    // 1. 记录 ANR 时间和原因
    long anrTime = SystemClock.uptimeMillis();

    // 2. dump 所有线程堆栈到 /data/anr/traces.txt
    dumpStackTraces(tracesFile, firstPids, nativePids);

    // 3. 收集系统信息
    StringBuilder info = new StringBuilder();
    info.append("ANR in " + app.processName);
    info.append("Reason: " + annotation);
    info.append("CPU usage: " + processCpuTracker.printCurrentState());

    // 4. 弹出 ANR 对话框（或直接杀进程）
    if (showBackground || isForeground) {
        Message msg = Message.obtain();
        msg.what = SHOW_NOT_RESPONDING_UI_MSG;
        mUiHandler.sendMessage(msg);
    }
}
```

**traces.txt 关键信息解读：**

```
----- pid 12345 at 2024-01-15 10:30:45 -----
Cmd line: com.example.app
Build fingerprint: 'google/pixel6/...'

DALVIK THREADS (35):
"main" prio=5 tid=1 Blocked        ← 主线程状态：Blocked（被锁阻塞）
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x... self=0x...
  | sysTid=12345 nice=-10 cgrp=default sched=0/0 handle=0x...
  | state=S schedstat=( 0 0 0 ) utm=150 stm=30 core=2 HZ=100
  | stack=0x7ff...-0x7ff... stackSize=8192KB
  | held mutexes=
  at com.example.app.DataManager.getData(DataManager.java:45)
  - waiting to lock <0x0a1b2c3d> (a java.lang.Object) held by thread 15  ← 等待线程15持有的锁
  at com.example.app.MainActivity.onResume(MainActivity.java:32)
  at android.app.Activity.performResume(Activity.java:8100)
  ...

"AsyncTask #1" prio=5 tid=15 Runnable    ← 线程15：持有锁，正在执行
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x... self=0x...
  at com.example.app.DataManager.syncToServer(DataManager.java:120)  ← 网络操作持有锁
  - locked <0x0a1b2c3d> (a java.lang.Object)                         ← 持有的锁
  at com.example.app.DataManager$SyncTask.doInBackground(DataManager.java:95)
  ...
```

**各类型 ANR 超时常量：**

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
static final int SERVICE_TIMEOUT = 20 * 1000;      // 前台 Service 20s
static final int SERVICE_BACKGROUND_TIMEOUT = 200 * 1000; // 后台 Service 200s

// frameworks/base/services/core/java/com/android/server/am/BroadcastQueue.java
static final int BROADCAST_FG_TIMEOUT = 10 * 1000; // 前台广播 10s
static final int BROADCAST_BG_TIMEOUT = 60 * 1000; // 后台广播 60s

// ContentProvider
static final int CONTENT_PROVIDER_PUBLISH_TIMEOUT = 10 * 1000; // 10s
```

### 实战层（项目经验）

**1. ANRWatchDog 实现原理：**

```kotlin
// 核心思路：子线程定时向主线程 post 任务，检测是否执行
class ANRWatchDog(private val timeoutMs: Long = 5000) : Thread("ANR-WatchDog") {
    @Volatile
    private var tick = 0L

    @Volatile
    private var reported = false

    private val handler = Handler(Looper.getMainLooper())
    private val ticker = Runnable { tick = 0; reported = false }

    override fun run() {
        while (!isInterrupted) {
            tick = SystemClock.uptimeMillis()
            handler.post(ticker) // 向主线程 post 重置 tick 的任务

            try {
                Thread.sleep(timeoutMs)
            } catch (e: InterruptedException) {
                return
            }

            // 如果 tick 未被重置，说明主线程在 timeoutMs 内未执行 ticker
            if (tick != 0L && !reported) {
                reported = true
                val mainThread = Looper.getMainLooper().thread
                val anrError = ANRError(mainThread.stackTrace)
                // 上报 ANR
                onAnrDetected(anrError)
            }
        }
    }
}
```

**2. FileObserver 监控 traces 文件：**

```kotlin
// 监听 /data/anr/ 目录（需要 root 或系统权限）
// 线上方案通常用 ANRWatchDog 替代
class ANRFileObserver : FileObserver("/data/anr/", CLOSE_WRITE) {
    override fun onEvent(event: Int, path: String?) {
        if (path?.contains("traces") == true) {
            // 读取 traces 文件，提取本进程堆栈
            val traces = readTraces("/data/anr/$path")
            reportANR(traces)
        }
    }
}

// Android 11+ 可通过 ApplicationExitInfo 获取 ANR 信息
fun getAnrHistory(context: Context) {
    val am = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
    val exitInfos = am.getHistoricalProcessExitReasons(
        context.packageName, 0, 10)

    exitInfos.filter { it.reason == ApplicationExitInfo.REASON_ANR }
        .forEach { info ->
            val trace = info.traceInputStream?.bufferedReader()?.readText()
            val timestamp = info.timestamp
            val description = info.description
            // 上报分析
            reportHistoricalANR(trace, timestamp, description)
        }
}
```

**3. 主线程耗时操作排查清单：**

```kotlin
// 常见 ANR 原因及修复

// 1. SharedPreferences apply 在 onStop 同步等待
// 修复：迁移到 MMKV
val kv = MMKV.defaultMMKV()
kv.encode("key", "value") // 不会在 onStop 阻塞

// 2. 主线程数据库操作
// 修复：Room + 协程
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getUser(id: Long): User // suspend 函数，自动切线程
}

// 3. 主线程网络请求（StrictMode 可检测）
// 修复：协程 + Dispatchers.IO
viewModelScope.launch {
    val data = withContext(Dispatchers.IO) {
        apiService.fetchData()
    }
    updateUI(data)
}

// 4. 死锁
// 修复：统一锁顺序，使用 tryLock 带超时
val lock = ReentrantLock()
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { /* 操作 */ }
    finally { lock.unlock() }
} else {
    // 获取锁超时，记录日志
}

// 5. Binder 调用阻塞（跨进程通信）
// 修复：异步 Binder 调用
// AIDL 中使用 oneway 关键字
// oneway interface IMyService { void doWork(); }
```

**4. 线上 ANR 治理体系：**

```kotlin
// 分级治理策略
class ANRGovernance {
    enum class ANRLevel {
        P0, // 影响 > 1% 用户，立即修复
        P1, // 影响 0.1-1% 用户，本迭代修复
        P2, // 影响 < 0.1% 用户，排期修复
    }

    fun classifyANR(anrInfo: ANRInfo): ANRLevel {
        val affectedRatio = anrInfo.affectedUsers.toFloat() / totalUsers
        return when {
            affectedRatio > 0.01 -> ANRLevel.P0
            affectedRatio > 0.001 -> ANRLevel.P1
            else -> ANRLevel.P2
        }
    }

    // ANR 堆栈聚合（相同堆栈归为一类）
    fun aggregateANR(anrList: List<ANRInfo>): Map<String, List<ANRInfo>> {
        return anrList.groupBy { info ->
            // 取主线程 top 5 帧作为聚合 key
            info.mainThreadStack.take(5).joinToString("\n")
        }
    }
}
```

**5. StrictMode 开发期检测：**

```kotlin
// 仅在 Debug 模式开启
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()      // 主线程磁盘读
            .detectDiskWrites()     // 主线程磁盘写
            .detectNetwork()        // 主线程网络
            .detectCustomSlowCalls() // 自定义慢调用
            .penaltyLog()           // 打印日志
            .penaltyFlashScreen()   // 屏幕闪烁提示
            .build()
    )

    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects()
            .detectLeakedClosableObjects()
            .detectActivityLeaks()
            .penaltyLog()
            .build()
    )
}
```

### 延伸问题

- [[卡顿优化-Choreographer与掉帧检测]] — 卡顿与 ANR 的关系和区别？
- [[启动优化-耗时分析与优化手段]] — 启动阶段 ANR 的特殊处理？
- [[存储优化-IO调度与MMKV原理]] — SP apply 导致 ANR 的原理？
- [[内存优化-LeakCanary与内存抖动]] — 低内存导致的 ANR？
- [[电量优化-WakeLock与JobScheduler]] — 后台 Service 超时 ANR？

## 记忆锚点

ANR 四个超时：Input 5s、前台广播 10s、前台 Service 20s、后台广播 60s；分析看 traces.txt 主线程状态（Blocked/Waiting）+ 锁持有者；治理 = 耗时操作移出主线程 + 锁优化 + SP→MMKV。
