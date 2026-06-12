---
tags: [android, 面试, framework, context, system_service, A]
difficulty: A
frequency: medium
---

# ContextImpl.mServiceCache 缓存机制详解

## 问题

> getSystemService() 的缓存在 ContextImpl.mServiceCache 中，那每个 Activity 都有自己的缓存吗？10 个 Activity 就 10 份缓存？这不是浪费吗？

## 核心答案

是的，每个 ContextImpl 实例有自己的 mServiceCache，10 个 Activity 就有 10 份。但实际影响很小：缓存存的是 Manager 引用（几十字节），不是对象深拷贝；Manager 内部的 Binder 代理在进程内是同一个对象；Manager 本身很轻量。设计上选择"每个 Context 各自缓存"是因为简单无需同步、Activity 销毁时缓存自然释放不会内存泄漏。用 `applicationContext.getSystemService()` 可以只走 Application 的全局一份缓存。

## 深入解析

### 原理层

**缓存位置的层级关系：**

```text
App 进程内：

  Application（Application Context）
    └── ContextImpl          ← mServiceCache[0..N]（进程内全局一份）

  Activity1
    └── ContextImpl          ← mServiceCache[0..N]（Activity1 自己的一份）

  Activity2
    └── ContextImpl          ← mServiceCache[0..N]（Activity2 自己的一份）

  Service1
    └── ContextImpl          ← mServiceCache[0..N]（Service1 自己的一份）
```

**CachedServiceFetcher 的实现：**

```java
static class CachedServiceFetcher<T> implements ServiceFetcher<T> {
    private final int mCacheIndex;  // 静态分配的数组下标

    @Override
    @SuppressWarnings("unchecked")
    public final T getService(ContextImpl ctx) {
        final Object[] cache = ctx.mServiceCache;  // 从当前 ContextImpl 取缓存数组

        // 先查缓存
        T result = (T) cache[mCacheIndex];
        if (result != null) {
            return result;  // 命中，直接返回
        }

        // 未命中，创建 Manager 并缓存
        result = createService(ctx);
        cache[mCacheIndex] = result;
        return result;
    }
}
```

`mCacheIndex` 是静态递增分配的，所有 ContextImpl 用同一个下标对应同一种服务。

**10 个 Activity 的缓存实际情况：**

```text
Activity1.mServiceCache[AMS_INDEX]  ──→ ActivityManager(proxy)
Activity2.mServiceCache[AMS_INDEX]  ──→ ActivityManager(proxy)
Activity3.mServiceCache[AMS_INDEX]  ──→ ActivityManager(proxy)
  ...

→ 每个 Activity 各自创建了一个 Manager 实例
→ 但 Manager 内部的 IActivityManager.Stub.Proxy 是同一个 Binder 代理对象
→ Binder 代理在进程内是单例（同一个 ServiceManager 查询结果）
```

**缓存存的是什么：**

```text
mServiceCache 数组中的每个元素：
  → 一个 Manager 对象的引用（几十字节）
  → Manager 内部持有一个 Binder 代理引用
  → Binder 代理本身在进程内是共享的

所以 10 个 Activity 的缓存：
  → 10 个 Manager 对象（几十字节 × 10 = 几百字节）
  → 1 个 Binder 代理（进程内共享）
  → 总共：几百字节 ~ 几 KB
```

### 为什么不用全局 static 缓存

**方案对比：**

| | 当前方案：每个 ContextImpl 各自缓存 | 假设方案：static 全局缓存 |
|---|---|---|
| 线程安全 | 不需要，每个 Context 单线程访问 | 需要同步锁，多 Activity 并发访问 |
| 内存管理 | Activity 销毁 → ContextImpl 回收 → 缓存释放 | static 永不释放 |
| 内存泄漏 | 无风险 | Manager 可能间接持有 Activity Context → 泄漏 |
| 复杂度 | 低 | 高 |
| 内存开销 | 几 KB（可忽略） | 几乎为零 |

**关键风险：static 缓存会导致内存泄漏**

```text
假设 static Object[] sGlobalCache:

Activity1 启动
  → getSystemService() → 创建 Manager，存入 sGlobalCache
  → Manager 内部引用了 ContextImpl（通过 createService 的参数）

Activity1 销毁
  → 但 sGlobalCache 还持有 Manager → Manager 持有 ContextImpl
  → ContextImpl 持有 Activity 的引用
  → Activity 无法被 GC → 内存泄漏

用每个 ContextImpl 自己的缓存：
Activity1 销毁 → ContextImpl 被回收 → mServiceCache 被回收 → 无泄漏
```

### 实际开发中的优化建议

**用 Application Context 获取：**

```text
// 推荐：全局一份缓存
val am = applicationContext.getSystemService(ActivityManager::class.java)
  → Application 的 ContextImpl.mServiceCache
  → 进程内只缓存一次

// 不推荐：每个 Activity 各自一份缓存
val am = getSystemService(ActivityManager::class.java)
  → Activity 自己的 ContextImpl.mServiceCache
  → 每个 Activity 都会创建新的 Manager
```

两者拿到的 Manager 内部是**同一个 Binder 代理**，功能完全一样，只是缓存位置不同。

**缓存不会跨进程共享：**

```text
App 进程 A：
  ContextImpl.mServiceCache[AMS] → Manager(proxyA)

App 进程 B：
  ContextImpl.mServiceCache[AMS] → Manager(proxyB)

不同进程有独立的 SystemServiceRegistry 和 mServiceCache
```

### 实战层

**面试常见追问：**

- mServiceCache 的下标是怎么分配的？
- 为什么不用 HashMap 而用数组？
- Application Context 和 Activity Context 的缓存有什么区别？
- CachedServiceFetcher 和 StaticServiceFetcher 的区别？
- 如果用 static 全局缓存会有什么问题？

**用数组而不是 HashMap 的原因：**

- 数组 O(1) 直接定位，HashMap 需要 hash + 比较
- `getSystemService()` 是高频调用，性能敏感
- 下标在 SystemServiceRegistry 注册时静态分配，编译时确定

**排查和观察：**

```bash
# 查看 App 进程的内存分布
adb shell dumpsys meminfo <package_name>

# 查看 ContextImpl 数量（通过 heap dump）
# 用 Android Studio Profiler → Heap Dump → 搜索 ContextImpl
```

**容易答错的点：**

- 每个 ContextImpl 确实有独立缓存，10 个 Activity 就 10 份
- 但这 10 份缓存存的 Manager 内部的 Binder 代理是同一个对象
- 不用全局 static 缓存是为了避免内存泄漏，不是因为技术上做不到
- `applicationContext.getSystemService()` 可以只用 Application 的一份缓存
- mServiceCache 用数组不用 HashMap，是为了 O(1) 查找性能

### 延伸问题

- [[系统服务获取机制]]
- [[Context体系与ContextWrapper]]
- [[如何添加自定义系统服务]]
- [[内存优化-LeakCanary与内存抖动]]

## 记忆锚点

每个 ContextImpl 各自缓存（10 个 Activity = 10 份），但 Manager 内部的 Binder 代理进程内共享，实际只多几百字节。设计原因是简单无锁 + Activity 销毁时缓存自然释放不泄漏。用 `applicationContext.getSystemService()` 可以只用全局一份。
