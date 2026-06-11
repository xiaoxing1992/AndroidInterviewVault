---
tags:
  - 性能优化
  - 内存优化
  - LeakCanary
  - Bitmap
difficulty: A
frequency: high
---

# 内存优化-LeakCanary与内存抖动

## 问题

> LeakCanary 的原理是什么？内存抖动怎么检测和解决？大图检测怎么做？

## 核心答案（30秒版本）

LeakCanary 利用 WeakReference + ReferenceQueue 机制：Activity/Fragment 销毁后创建弱引用并关联 ReferenceQueue，GC 后检查 ReferenceQueue 是否收到回调，未收到说明对象未被回收即泄漏，再通过 dump hprof + Shark 库分析引用链。内存抖动通过 Profiler 观察锯齿状内存曲线定位，根因是循环中频繁创建临时对象。大图检测通过 hook ImageView 的 setImageDrawable 或运行时检查 Bitmap 尺寸与 View 尺寸比例。

## 深入解析

### 原理层（源码级）

**LeakCanary 核心流程：**

```
Activity.onDestroy → ObjectWatcher.expectWeaklyReachable
→ 创建 KeyedWeakReference(activity, key, referenceQueue)
→ 等待 5s → 触发 GC → 检查 referenceQueue
→ 未入队 = 泄漏 → dump hprof → Shark 分析引用链 → 通知
```

**源码关键实现：**

```kotlin
// leakcanary-object-watcher/src/main/java/leakcanary/ObjectWatcher.kt
class ObjectWatcher {
    private val watchedObjects = mutableMapOf<String, KeyedWeakReference>()
    private val queue = ReferenceQueue<Any>()

    fun expectWeaklyReachable(watchedObject: Any, description: String) {
        val key = UUID.randomUUID().toString()
        val reference = KeyedWeakReference(watchedObject, key, description, queue)
        watchedObjects[key] = reference

        // 5秒后检查
        checkRetainedExecutor.execute {
            moveToRetained(key)
        }
    }

    private fun moveToRetained(key: String) {
        // 先清理已回收的引用
        removeWeaklyReachableObjects()
        val retainedRef = watchedObjects[key]
        if (retainedRef != null) {
            // 仍在 watchedObjects 中 = 未被 GC 回收 = 泄漏嫌疑
            retainedRef.retainedUptimeMillis = SystemClock.uptimeMillis()
            onObjectRetainedListeners.forEach { it.onObjectRetained() }
        }
    }

    private fun removeWeaklyReachableObjects() {
        var ref: KeyedWeakReference?
        do {
            ref = queue.poll() as KeyedWeakReference?
            if (ref != null) {
                // 已被 GC 回收，从监控列表移除
                watchedObjects.remove(ref.key)
            }
        } while (ref != null)
    }
}
```

**WeakReference + ReferenceQueue 原理：**

```java
// JVM 层面
// 当 GC 发现一个对象只有 WeakReference 指向它时：
// 1. 将该对象标记为可回收
// 2. 将对应的 WeakReference 加入关联的 ReferenceQueue
// 3. 下次 GC 时回收该对象

// 关键：如果对象还有强引用链（泄漏），GC 不会回收它
// WeakReference 也不会被加入 ReferenceQueue
```

**内存抖动原理：**

内存抖动 = 短时间内大量对象创建和回收，导致频繁 GC（尤其是 Stop-The-World 的 GC），表现为：
- Memory Profiler 中锯齿状曲线
- CPU Profiler 中大量 GC 事件
- 用户感知到卡顿

```kotlin
// 典型反面案例：onDraw 中创建对象
class BadView(context: Context) : View(context) {
    override fun onDraw(canvas: Canvas) {
        // 每帧都创建新 Paint 和 Rect，导致内存抖动
        val paint = Paint().apply { color = Color.RED }
        val rect = Rect(0, 0, width, height)
        canvas.drawRect(rect, paint)
    }
}

// 正确做法：对象复用
class GoodView(context: Context) : View(context) {
    private val paint = Paint().apply { color = Color.RED }
    private val rect = Rect()

    override fun onDraw(canvas: Canvas) {
        rect.set(0, 0, width, height)
        canvas.drawRect(rect, paint)
    }
}
```

**Bitmap 内存计算：**

```
内存占用 = 宽 × 高 × 每像素字节数 × (设备 dpi / 资源目录 dpi)²

ARGB_8888: 4 bytes/pixel
RGB_565:   2 bytes/pixel

例：1080×1920 图片放在 xhdpi(320) 目录，在 xxhdpi(480) 设备加载
= 1080 × 1920 × 4 × (480/320)² = 1080 × 1920 × 4 × 2.25 ≈ 18.6MB
```

### 实战层（项目经验）

**1. 大图检测方案：**

```kotlin
// 运行时 Hook 方案：通过 ARTHook 或 Epic 框架 hook ImageView
object BigImageDetector {

    fun detect(imageView: ImageView) {
        val drawable = imageView.drawable ?: return
        val viewWidth = imageView.width
        val viewHeight = imageView.height
        if (viewWidth == 0 || viewHeight == 0) return

        val drawableWidth = drawable.intrinsicWidth
        val drawableHeight = drawable.intrinsicHeight

        // 图片像素面积超过 View 面积 2 倍以上告警
        if (drawableWidth * drawableHeight > viewWidth * viewHeight * 2) {
            Log.w("BigImage",
                "图片过大: drawable=${drawableWidth}x${drawableHeight}, " +
                "view=${viewWidth}x${viewHeight}, " +
                "位置: ${getViewLocation(imageView)}")
        }
    }
}

// Glide 全局监听方案
class BigImageGlideModule : AppGlideModule() {
    override fun registerComponents(context: Context, glide: Glide, registry: Registry) {
        // 注册自定义 ResourceTranscoder 检测大图
    }
}
```

**2. 内存泄漏常见场景与修复：**

```kotlin
// 场景1：匿名内部类持有 Activity 引用
class LeakActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // 错误：Handler 持有 Activity 引用
        val handler = object : Handler(Looper.getMainLooper()) {
            override fun handleMessage(msg: Message) {
                updateUI() // 隐式持有外部类引用
            }
        }
        handler.sendMessageDelayed(Message.obtain(), 60_000)
    }
}

// 修复：静态内部类 + WeakReference
class SafeActivity : AppCompatActivity() {
    private class SafeHandler(activity: SafeActivity) : Handler(Looper.getMainLooper()) {
        private val activityRef = WeakReference(activity)
        override fun handleMessage(msg: Message) {
            activityRef.get()?.updateUI()
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacksAndMessages(null)
    }
}
```

**3. Bitmap 优化实战：**

```kotlin
// 按需采样加载
fun decodeSampledBitmap(res: Resources, resId: Int,
                         reqWidth: Int, reqHeight: Int): Bitmap {
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true // 只读尺寸不加载像素
    }
    BitmapFactory.decodeResource(res, resId, options)

    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)
    options.inJustDecodeBounds = false
    return BitmapFactory.decodeResource(res, resId, options)
}

fun calculateInSampleSize(options: BitmapFactory.Options,
                           reqWidth: Int, reqHeight: Int): Int {
    val (height, width) = options.outHeight to options.outWidth
    var inSampleSize = 1
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        while (halfHeight / inSampleSize >= reqHeight
            && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

**4. 线上内存监控：**

```kotlin
// 低内存预警
class MemoryMonitor {
    private val runtime = Runtime.getRuntime()

    fun checkMemoryState(): MemoryState {
        val maxMem = runtime.maxMemory()
        val usedMem = runtime.totalMemory() - runtime.freeMemory()
        val ratio = usedMem.toFloat() / maxMem

        return when {
            ratio > 0.9 -> MemoryState.CRITICAL  // 触发主动 GC + dump
            ratio > 0.75 -> MemoryState.WARNING  // 清理缓存
            else -> MemoryState.NORMAL
        }
    }
}
```

### 延伸问题

- [[卡顿优化-Choreographer与掉帧检测]] — 内存抖动导致的 GC 卡顿如何关联分析？
- [[启动优化-耗时分析与优化手段]] — 启动阶段的内存峰值如何控制？
- [[ANR分析与治理]] — 内存不足导致的 ANR 如何定位？
- [[存储优化-IO调度与MMKV原理]] — Bitmap 磁盘缓存策略？

## 记忆锚点

LeakCanary = WeakReference 观察 + ReferenceQueue 判活 + Shark 分析引用链；内存抖动 = 循环创建对象 → 对象复用解决；大图 = 像素面积 vs View 面积比例检测。
