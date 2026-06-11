---
tags: [java, reference, memory-leak, weakreference, android]
difficulty: A
frequency: high
---

# Java 引用类型与内存泄漏

## 问题

"Java 的四种引用类型分别是什么？WeakReference 和 SoftReference 的区别？ReferenceQueue 有什么用？Handler 内存泄漏怎么解决？LeakCanary 是怎么检测泄漏的？Android 中常见的内存泄漏场景有哪些？"

## 核心答案（30 秒版本）

Java 引用强度递减：**强引用 > 软引用 > 弱引用 > 虚引用**。强引用不会被 GC；软引用在内存不足时回收（适合缓存）；弱引用在下次 GC 时必定回收（适合防泄漏）；虚引用无法获取对象，仅用于回收通知。ReferenceQueue 在引用对象被回收时收到通知。Handler 泄漏因为匿名内部类持有 Activity 引用 + MessageQueue 持有 Handler，解决方案是**静态内部类 + WeakReference**。LeakCanary 通过 WeakReference + ReferenceQueue 检测 Activity/Fragment 销毁后是否被回收。

## 深入解析

### 原理层（源码级）

**1. 四种引用类型对比**

```java
// 强引用 (Strong Reference)
Object obj = new Object();  // GC 永不回收，宁可 OOM

// 软引用 (Soft Reference)
SoftReference<Bitmap> softRef = new SoftReference<>(bitmap);
// 内存充足时不回收，内存不足时回收
// 适合：图片缓存、页面缓存

// 弱引用 (Weak Reference)
WeakReference<Activity> weakRef = new WeakReference<>(activity);
// 下次 GC 时必定回收，不论内存是否充足
// 适合：防止内存泄漏

// 虚引用 (Phantom Reference)
PhantomReference<Object> phantomRef = new PhantomReference<>(obj, queue);
// get() 永远返回 null，仅用于跟踪对象被回收的时机
// 适合：堆外内存管理（DirectByteBuffer 的 Cleaner）
```

**2. Reference 源码结构**

```java
public abstract class Reference<T> {
    private T referent;          // 引用的对象
    volatile ReferenceQueue<? super T> queue;  // 关联的队列
    volatile Reference next;     // 队列中的下一个

    // GC 线程发现对象可回收时：
    // 1. 将 referent 置为 null（弱/虚引用）或标记（软引用）
    // 2. 将 Reference 对象加入 ReferenceQueue
}

// ReferenceQueue：被回收对象的通知队列
ReferenceQueue<Object> queue = new ReferenceQueue<>();
WeakReference<Object> ref = new WeakReference<>(obj, queue);

// obj 被 GC 回收后，ref 会被加入 queue
Reference<?> polled = queue.poll();  // 非阻塞获取
Reference<?> removed = queue.remove(timeout);  // 阻塞等待
```

**3. SoftReference 回收策略**

```java
// JVM 回收软引用的策略（HotSpot）：
// clock - timestamp > freespace * SoftRefLRUPolicyMSPerMB
// clock: 当前时间
// timestamp: 上次访问时间
// freespace: 堆空闲空间(MB)
// SoftRefLRUPolicyMSPerMB: 每 MB 空闲空间允许存活的毫秒数（默认 1000）

// 示例：空闲 100MB，则软引用最多存活 100 * 1000 = 100秒
// Android 中此值通常更小，软引用回收更积极
```

**4. WeakHashMap 原理**

```java
// WeakHashMap 的 Entry 继承 WeakReference
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;

    Entry(Object key, V value, ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue);  // key 是弱引用
        this.value = value;
    }
}

// 每次操作时清理已回收的 Entry
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        Entry<K,V> e = (Entry<K,V>) x;
        // 从 table 中移除该 entry
    }
}
```

### 实战层（项目经验）

**1. Handler 内存泄漏与解决**

```java
// ❌ 泄漏写法：匿名内部类持有外部 Activity 引用
public class MainActivity extends Activity {
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 隐式持有 MainActivity.this
            updateUI();
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        handler.sendMessageDelayed(msg, 60_000);  // 60秒后执行
        // 如果用户在 60 秒内退出 Activity：
        // MessageQueue → Message → Handler → Activity（泄漏链）
    }
}

// ✅ 正确写法：静态内部类 + WeakReference
public class MainActivity extends Activity {
    private static class SafeHandler extends Handler {
        private final WeakReference<MainActivity> activityRef;

        SafeHandler(MainActivity activity) {
            this.activityRef = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = activityRef.get();
            if (activity != null && !activity.isFinishing()) {
                activity.updateUI();
            }
        }
    }

    private final SafeHandler handler = new SafeHandler(this);

    @Override
    protected void onDestroy() {
        super.onDestroy();
        handler.removeCallbacksAndMessages(null);  // 清除所有消息
    }
}
```

**2. LeakCanary 检测原理**

```java
// LeakCanary 核心流程
public class ActivityWatcher {
    private final Set<String> watchedKeys = ConcurrentHashMap.newKeySet();
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

    void onActivityDestroyed(Activity activity) {
        String key = UUID.randomUUID().toString();
        watchedKeys.add(key);

        // 1. 用 WeakReference + ReferenceQueue 监控 Activity
        KeyedWeakReference ref = new KeyedWeakReference(activity, key, queue);

        // 2. 5 秒后检查是否被回收
        mainHandler.postDelayed(() -> checkRetained(key), 5_000);
    }

    void checkRetained(String key) {
        // 3. 先清理已回收的引用
        removeWeaklyReachableReferences();

        if (watchedKeys.contains(key)) {
            // 4. 仍在 watchedKeys 中 = 未被回收 = 可能泄漏
            // 触发 GC 再检查一次
            Runtime.getRuntime().gc();
            removeWeaklyReachableReferences();

            if (watchedKeys.contains(key)) {
                // 5. 确认泄漏，dump heap 分析引用链
                heapDumper.dumpHeap();
            }
        }
    }

    void removeWeaklyReachableReferences() {
        KeyedWeakReference ref;
        while ((ref = (KeyedWeakReference) queue.poll()) != null) {
            watchedKeys.remove(ref.key);  // 已回收，移除监控
        }
    }
}
```

**3. Android 常见内存泄漏场景**

```java
// 场景 1: 单例持有 Activity Context
public class AppManager {
    private static AppManager instance;
    private Context context;

    private AppManager(Context context) {
        this.context = context;  // ❌ 如果传入 Activity
    }
    // ✅ 修复：使用 context.getApplicationContext()
}

// 场景 2: 匿名内部类/非静态内部类
button.setOnClickListener(new View.OnClickListener() {
    // 持有外部类引用
});
// ✅ 修复：使用 lambda（编译器优化）或静态类

// 场景 3: 未注销的监听器/广播
@Override
protected void onCreate(Bundle savedInstanceState) {
    sensorManager.registerListener(this, sensor, DELAY);
}
// ❌ 忘记在 onDestroy 中 unregisterListener
// ✅ 修复：在 onDestroy/onPause 中注销

// 场景 4: Coroutine/RxJava 未取消
lifecycleScope.launch {
    // ✅ 自动跟随生命周期取消
}
// ❌ GlobalScope.launch { ... }  // 不跟随生命周期

// 场景 5: WebView 泄漏
// WebView 内部持有 Activity 引用且难以释放
// ✅ 修复：WebView 放在单独进程，或手动 destroy + 从 ViewGroup 移除
@Override
protected void onDestroy() {
    if (webView != null) {
        ViewGroup parent = (ViewGroup) webView.getParent();
        if (parent != null) parent.removeView(webView);
        webView.stopLoading();
        webView.destroy();
        webView = null;
    }
    super.onDestroy();
}

// 场景 6: 集合类持有对象引用
private static List<Activity> activityStack = new ArrayList<>();
// ✅ 修复：使用 WeakReference 或在 onDestroy 中移除
```

**4. 软引用实现图片缓存**

```java
// 二级缓存：LruCache(强引用) + SoftReference(软引用)
public class ImageCache {
    // 一级缓存：LRU 强引用
    private final LruCache<String, Bitmap> lruCache;
    // 二级缓存：软引用
    private final Map<String, SoftReference<Bitmap>> softCache = new ConcurrentHashMap<>();

    public ImageCache(int maxSize) {
        lruCache = new LruCache<String, Bitmap>(maxSize) {
            @Override
            protected void entryRemoved(boolean evicted, String key,
                                         Bitmap oldValue, Bitmap newValue) {
                // LRU 淘汰时转入软引用缓存
                softCache.put(key, new SoftReference<>(oldValue));
            }
        };
    }

    public Bitmap get(String key) {
        // 先查 LRU
        Bitmap bitmap = lruCache.get(key);
        if (bitmap != null) return bitmap;

        // 再查软引用
        SoftReference<Bitmap> ref = softCache.get(key);
        if (ref != null) {
            bitmap = ref.get();
            if (bitmap != null) {
                lruCache.put(key, bitmap);  // 提升到 LRU
                softCache.remove(key);
                return bitmap;
            } else {
                softCache.remove(key);  // 已被回收
            }
        }
        return null;
    }
}
```

**5. Cleaner 机制（虚引用应用）**

```java
// DirectByteBuffer 使用虚引用释放堆外内存
// Java 9+ Cleaner API
public class NativeResource implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    private final Cleaner.Cleanable cleanable;
    private final long nativePtr;

    public NativeResource() {
        this.nativePtr = allocateNative();
        // 注册清理动作（通过虚引用 + ReferenceQueue 触发）
        this.cleanable = cleaner.register(this, () -> freeNative(nativePtr));
    }

    @Override
    public void close() {
        cleanable.clean();  // 主动释放
    }
    // 即使忘记 close，GC 时虚引用也会触发 freeNative
}
```

### 延伸问题

- [[JVM内存模型与垃圾回收]] — GC Roots 与引用类型的关系
- [[内存优化-LeakCanary与内存抖动]] — LeakCanary 2.0 的 Shark 分析引擎
- [[Handler消息机制]] — Handler 泄漏的完整引用链
- [[Kotlin协程与生命周期]] — lifecycleScope 如何自动防泄漏

## 记忆锚点

**强引用不回收、软引用内存不足时回收、弱引用下次 GC 必回收、虚引用仅做回收通知；LeakCanary = WeakReference + ReferenceQueue + 延迟检测；Handler 泄漏用静态类+弱引用+onDestroy 清消息三板斧。**
