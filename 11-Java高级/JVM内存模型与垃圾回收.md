---
tags: [java, jvm, gc, memory, android-art]
difficulty: S
frequency: high
---

# JVM 内存模型与垃圾回收

## 问题

"说一下 JVM 的内存区域划分，GC Roots 有哪些？Android ART 的垃圾回收和标准 JVM 有什么区别？GC 会怎样影响 App 流畅度？"

## 核心答案（30 秒版本）

JVM 运行时内存分为**堆（Heap）、虚拟机栈（VM Stack）、本地方法栈、程序计数器、方法区（Metaspace）**五大区域。GC Roots 包括栈帧中的局部变量、静态变量、JNI 引用等。Android ART 使用 **Concurrent Copying（CC）** 收集器，通过读屏障 + 并发复制实现近乎无停顿的 GC，相比 Dalvik 的 Stop-The-World 大幅降低了卡顿。

## 深入解析

### 原理层（源码级）

**1. 内存区域详解**

```
┌─────────────────────────────────────────┐
│              JVM 运行时数据区              │
├──────────┬──────────┬───────────────────┤
│ 线程私有  │ 虚拟机栈  │ 栈帧(局部变量表/   │
│          │          │ 操作数栈/动态链接)  │
│          ├──────────┤                   │
│          │ 程序计数器 │ 当前字节码行号     │
│          ├──────────┤                   │
│          │本地方法栈  │ Native 方法       │
├──────────┼──────────┼───────────────────┤
│ 线程共享  │   堆     │ 对象实例/数组      │
│          ├──────────┤                   │
│          │ 方法区    │ 类信息/常量池/     │
│          │(Metaspace)│ 静态变量          │
└──────────┴──────────┴───────────────────┘
```

**2. GC Roots 可达性分析**

```java
// GC Roots 包括：
// 1. 虚拟机栈中引用的对象
void method() {
    Object obj = new Object(); // obj 是 GC Root
}

// 2. 类静态属性引用的对象
static Object sInstance = new Object();

// 3. 常量引用的对象
static final Object CONSTANT = new Object();

// 4. JNI（Native 方法）引用的对象
// 5. 同步锁（synchronized）持有的对象
// 6. JVM 内部引用（Class 对象、异常对象、类加载器）
```

**3. 分代回收策略**

```
堆内存分代：
┌────────────────────────────────────────┐
│  Young Generation (1/3)                │
│  ┌────────┬─────────┬─────────┐       │
│  │  Eden  │   S0    │   S1    │       │
│  │  (8)   │  (1)    │  (1)    │       │
│  └────────┴─────────┴─────────┘       │
├────────────────────────────────────────┤
│  Old Generation (2/3)                  │
│  ┌────────────────────────────┐       │
│  │  Tenured Space             │       │
│  └────────────────────────────┘       │
└────────────────────────────────────────┘
```

- **Minor GC**：Eden 满时触发，存活对象复制到 Survivor，年龄+1
- **Major GC / Full GC**：Old 区满时触发，STW 时间长
- 晋升条件：年龄达到阈值（默认 15）或 Survivor 放不下

**4. Android ART GC 演进**

```
Dalvik (Android 4.x):
  - Mark-Sweep，全量 STW
  - GC 暂停 50-100ms，明显卡顿

ART (Android 5.0+):
  - Concurrent Mark Sweep (CMS)
  - 并发标记，STW 仅在 remark 阶段
  - 暂停降至 2-5ms

ART CC (Android 8.0+):
  - Concurrent Copying Collector
  - 读屏障（Read Barrier）替代写屏障
  - 并发复制 + 压缩，几乎无暂停（<1ms）
  - 支持 Region-based 分配
```

ART CC 核心源码路径：`art/runtime/gc/collector/concurrent_copying.cc`

```cpp
// 读屏障伪代码
Object* ReadBarrier::Mark(Object* obj) {
    if (obj != nullptr && obj->GetReadBarrierState() != ReadBarrier::GrayState()) {
        return Mark(obj);  // 转发到 to-space 副本
    }
    return obj;
}
```

### 实战层（项目经验）

**1. GC 导致卡顿的排查**

```bash
# 通过 logcat 观察 GC 日志
adb logcat -s art | grep -i "gc"
# 输出示例：
# Explicit concurrent copying GC freed 2MB, 45% free, 12MB/22MB, paused 0.1ms
```

**2. 减少 GC 压力的实践**

```java
// 反例：在 onDraw 中频繁创建对象
@Override
protected void onDraw(Canvas canvas) {
    Paint paint = new Paint();  // 每帧创建，触发 Young GC
    Rect rect = new Rect();
}

// 正例：对象复用
private final Paint mPaint = new Paint();
private final Rect mRect = new Rect();

@Override
protected void onDraw(Canvas canvas) {
    // 复用已有对象，零分配
}
```

**3. 大对象与内存抖动**

```java
// 使用对象池避免内存抖动
public class MessagePool {
    private static final int MAX_POOL_SIZE = 50;
    private static Message sPool;
    private static int sPoolSize = 0;

    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
}
```

**4. 监控 GC 对帧率的影响**

```java
// 通过 FrameMetrics 监控
window.addOnFrameMetricsAvailableListener((window, metrics, dropCount) -> {
    long totalDuration = metrics.getMetric(FrameMetrics.TOTAL_DURATION);
    if (totalDuration > 16_000_000) { // 超过 16ms
        // 可能存在 GC 暂停导致掉帧
        Log.w(TAG, "Janky frame: " + totalDuration / 1_000_000 + "ms");
    }
}, new Handler());
```

### 延伸问题

- [[Java引用类型与内存泄漏]] — 弱引用如何辅助 GC
- [[内存优化-LeakCanary与内存抖动]] — 内存泄漏检测原理
- [[类加载机制与双亲委派]] — 方法区中类信息的加载
- [[volatile与happens-before]] — JMM 与内存可见性的关系

## 记忆锚点

**GC Roots 是可达性分析的起点，ART CC 通过读屏障+并发复制实现亚毫秒暂停，Android 性能优化的核心是减少对象分配频率。**
