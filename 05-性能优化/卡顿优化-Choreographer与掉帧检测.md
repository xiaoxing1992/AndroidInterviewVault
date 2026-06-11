---
tags:
  - 性能优化
  - 卡顿优化
  - Choreographer
  - VSYNC
difficulty: S
frequency: high
---

# 卡顿优化-Choreographer与掉帧检测

## 问题

> Choreographer 的工作原理是什么？怎么检测掉帧？线上卡顿监控方案怎么设计？

## 核心答案（30秒版本）

Choreographer 是 Android 渲染链路的调度者，接收 VSYNC 信号后按 Input → Animation → Traversal 顺序执行回调。每个 VSYNC 周期 16.6ms（60Hz），若一帧处理超时则掉帧。检测方案：Choreographer.FrameCallback 计算帧间隔、Looper Printer 监控主线程消息耗时（BlockCanary 原理）、Matrix 的 LooperAnrTracer 结合 ANR 检测。线上方案核心是采集卡顿时的堆栈 + 系统状态。

## 深入解析

### 原理层（源码级）

**Choreographer 核心架构：**

```
VSYNC 信号 (来自 SurfaceFlinger)
    ↓
DisplayEventReceiver.onVsync()
    ↓
Choreographer.doFrame()
    ↓
┌─────────────────────────────────────────┐
│ CALLBACK_INPUT      → 输入事件处理       │
│ CALLBACK_ANIMATION  → 动画计算           │
│ CALLBACK_INSETS_ANIMATION → 窗口动画     │
│ CALLBACK_TRAVERSAL  → View 三大流程      │
│ CALLBACK_COMMIT     → 提交帧数据         │
└─────────────────────────────────────────┘
    ↓
RenderThread → GPU 渲染 → SurfaceFlinger 合成
```

**Choreographer 源码关键路径：**

```java
// frameworks/base/core/java/android/view/Choreographer.java
public final class Choreographer {
    private static final long DEFAULT_FRAME_DELAY = 10; // ms

    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            long intendedFrameTimeNanos = frameTimeNanos;
            startNanos = System.nanoTime();
            // 计算掉帧数
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    // "Skipped XX frames! The application may be doing too much work on its main thread."
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!");
                }
            }
        }

        // 按顺序执行回调
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
    }
}
```

**VSYNC 机制：**

```
                    VSYNC 信号 (硬件中断)
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    VSYNC-app      VSYNC-sf       Display 刷新
   (App 开始绘制)  (SF 开始合成)   (显示上一帧)

Triple Buffering:
  Frame N:   [CPU绘制] [GPU渲染]
  Frame N+1:          [CPU绘制] [GPU渲染]
  Display:                      [显示N] [显示N+1]
```

**Looper Printer 监控原理（BlockCanary 核心）：**

```java
// frameworks/base/core/java/android/os/Looper.java
public static void loop() {
    for (;;) {
        Message msg = queue.next();
        // 消息处理前打印
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback);
        }

        msg.target.dispatchMessage(msg); // 实际处理

        // 消息处理后打印
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
    }
}
```

**BlockCanary 利用这个机制：**

```kotlin
class BlockDetectByPrinter : Printer {
    private var startTime = 0L
    private var startThreadTime = 0L

    override fun println(x: String) {
        if (x.startsWith(">>>>> Dispatching")) {
            startTime = System.currentTimeMillis()
            startThreadTime = SystemClock.currentThreadTimeMillis()
            // 启动采样线程，定时抓取主线程堆栈
            stackSampler.startDump()
        }
        if (x.startsWith("<<<<< Finished")) {
            val wallTime = System.currentTimeMillis() - startTime
            val cpuTime = SystemClock.currentThreadTimeMillis() - startThreadTime
            if (wallTime > THRESHOLD) { // 默认阈值 500ms
                // 卡顿！收集堆栈
                val stacks = stackSampler.getStacks(startTime, System.currentTimeMillis())
                notifyBlockEvent(wallTime, cpuTime, stacks)
            }
            stackSampler.stopDump()
        }
    }
}
```

### 实战层（项目经验）

**1. FrameCallback 掉帧检测：**

```kotlin
object FPSMonitor {
    private var lastFrameTimeNanos = 0L
    private var frameCount = 0
    private val frameCallback = object : Choreographer.FrameCallback {
        override fun doFrame(frameTimeNanos: Long) {
            if (lastFrameTimeNanos != 0L) {
                val diff = (frameTimeNanos - lastFrameTimeNanos) / 1_000_000 // ms
                val droppedFrames = (diff / 16.6).toInt() - 1
                if (droppedFrames > 0) {
                    onFrameDropped(droppedFrames, diff)
                }
            }
            lastFrameTimeNanos = frameTimeNanos
            frameCount++
            Choreographer.getInstance().postFrameCallback(this)
        }
    }

    fun start() {
        Choreographer.getInstance().postFrameCallback(frameCallback)
        // 每秒计算 FPS
        handler.postDelayed(fpsRunnable, 1000)
    }

    private fun onFrameDropped(count: Int, costMs: Long) {
        when {
            count >= 42 -> reportFrozen(costMs)   // 冻帧 > 700ms
            count >= 6 -> reportJank(costMs)      // 严重卡顿 > 100ms
            count >= 3 -> reportSlightJank(costMs) // 轻微卡顿 > 50ms
        }
    }
}
```

**2. 线上卡顿监控完整方案：**

```kotlin
class JankMonitor {
    private val stackSampler = StackSampler(
        thread = Looper.getMainLooper().thread,
        intervalMs = 50 // 每 50ms 采样一次
    )

    // 堆栈采样器
    class StackSampler(private val thread: Thread, private val intervalMs: Long) {
        private val stackMap = LinkedHashMap<Long, String>()

        fun startDump() {
            timer.scheduleAtFixedRate(object : TimerTask() {
                override fun run() {
                    val stack = thread.stackTrace
                        .joinToString("\n") { "\tat $it" }
                    synchronized(stackMap) {
                        stackMap[System.currentTimeMillis()] = stack
                    }
                }
            }, 0, intervalMs)
        }

        fun getStacks(startTime: Long, endTime: Long): List<String> {
            synchronized(stackMap) {
                return stackMap.filter { it.key in startTime..endTime }
                    .values.toList()
            }
        }
    }

    // 卡顿信息上报
    data class JankInfo(
        val wallTime: Long,
        val cpuTime: Long,
        val stacks: List<String>,
        val memoryInfo: MemoryInfo,
        val cpuUsage: Float,
        val scene: String // 当前页面
    )
}
```

**3. Matrix 卡顿检测方案（字节码插桩）：**

```kotlin
// Matrix 通过 Gradle Plugin 在编译期对每个方法插桩
// 插桩前
fun businessMethod() {
    // 业务逻辑
}

// 插桩后
fun businessMethod() {
    AppMethodBeat.i(methodId) // 方法进入，记录时间戳
    // 业务逻辑
    AppMethodBeat.o(methodId) // 方法退出，记录时间戳
}

// AppMethodBeat 使用 long[] 数组记录方法进出时间
// 卡顿发生时，回溯数组即可得到精确的方法调用链和耗时
object AppMethodBeat {
    private val buffer = LongArray(100_000) // 环形缓冲区
    private var index = 0

    fun i(methodId: Int) {
        // 高 32 位存 methodId，低 32 位存时间偏移
        buffer[index++] = (methodId.toLong() shl 32) or
            (SystemClock.uptimeMillis() - startTime)
    }
}
```

**4. 卡顿归因分析：**

```kotlin
// 区分 CPU 卡顿 vs IO 卡顿 vs 锁等待
fun analyzeJankCause(wallTime: Long, cpuTime: Long): JankCause {
    val cpuRatio = cpuTime.toFloat() / wallTime
    return when {
        cpuRatio > 0.8 -> JankCause.CPU_BINDIND  // CPU 密集
        cpuRatio < 0.3 -> JankCause.IO_OR_LOCK   // IO 或锁等待
        else -> JankCause.MIXED
    }
}
```

### 延伸问题

- [[启动优化-耗时分析与优化手段]] — 启动阶段的掉帧如何优化？
- [[ANR分析与治理]] — 卡顿升级为 ANR 的阈值和条件？
- [[内存优化-LeakCanary与内存抖动]] — GC 导致的卡顿如何区分？
- [[电量优化-WakeLock与JobScheduler]] — 高频渲染对电量的影响？

## 记忆锚点

Choreographer = VSYNC 驱动的帧调度器；掉帧检测三板斧：FrameCallback 算帧间隔、Looper Printer 抓消息耗时（BlockCanary）、字节码插桩精确到方法（Matrix）。
