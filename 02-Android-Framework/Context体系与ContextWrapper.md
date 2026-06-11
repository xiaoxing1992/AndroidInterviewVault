---
tags: [android, 面试, framework, context, A]
difficulty: A
frequency: high
---

# Context 体系与 ContextWrapper

## 问题

> Android 中有多少种 Context？Application Context 和 Activity Context 有什么区别？为什么有些场景必须用 Activity Context？ContextWrapper 的设计模式是什么？

## 核心答案

Context 是 Android 的上帝对象，提供访问系统资源和服务的能力。Context 体系采用装饰器模式：ContextImpl 是真正的实现，ContextWrapper 是装饰器基类，Application/Activity/Service 都继承自 ContextWrapper。Activity Context 关联了 Window 和主题，能启动 Activity 和弹 Dialog；Application Context 生命周期长但缺少 UI 能力。

## 深入解析

### 原理层

**继承体系：**
```
Context (抽象类)
├── ContextImpl (真正的实现类)
└── ContextWrapper (装饰器)
    ├── Application
    ├── ContextThemeWrapper
    │   └── Activity
    └── Service
```

**Context 数量公式：**
```
Context 总数 = Activity 数 + Service 数 + 1 (Application)
```

**ContextImpl 核心能力：**
```java
class ContextImpl extends Context {
    private Resources mResources;
    private ContentResolver mContentResolver;
    private PackageManager mPackageManager;
    
    @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
    
    @Override
    public SharedPreferences getSharedPreferences(String name, int mode) {
        // 实际的文件操作实现
    }
    
    @Override
    public void startActivity(Intent intent) {
        // 非 Activity Context 启动 Activity 需要 FLAG_ACTIVITY_NEW_TASK
        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException("...");
        }
    }
}
```

**装饰器模式实现：**
```java
public class ContextWrapper extends Context {
    Context mBase;  // 被装饰的 ContextImpl
    
    protected void attachBaseContext(Context base) {
        mBase = base;
    }
    
    @Override
    public Resources getResources() {
        return mBase.getResources();  // 委托给 ContextImpl
    }
}
```

**Activity Context 的特殊性：**
```java
public class Activity extends ContextThemeWrapper {
    private Window mWindow;  // PhoneWindow
    private WindowManager mWindowManager;
    
    // ContextThemeWrapper 额外持有 Theme
    // 所以 Activity 能正确应用主题样式
}
```

**各 Context 能力对比：**

| 操作 | Application | Activity | Service |
|------|-------------|----------|---------|
| startActivity | 需要 NEW_TASK | 正常 | 需要 NEW_TASK |
| 弹 Dialog | 不行 | 正常 | 不行 |
| inflate Layout | 无主题 | 有主题 | 无主题 |
| getResources | 正常 | 正常 | 正常 |
| getSystemService | 正常 | 正常 | 正常 |
| 绑定 Service | 正常 | 正常 | 正常 |

### 实战层

**选择 Context 的原则：**
- 需要 UI 操作（Dialog、Theme、inflate）→ Activity Context
- 生命周期长于 Activity（单例、全局缓存）→ Application Context
- 普通系统服务调用 → 两者皆可，优先 Application 避免泄漏

**内存泄漏场景：**
```kotlin
// ❌ 单例持有 Activity Context → 内存泄漏
object ImageLoader {
    lateinit var context: Context  // 如果传入 Activity 就泄漏了
}

// ✅ 使用 Application Context
object ImageLoader {
    lateinit var context: Context
    fun init(context: Context) {
        this.context = context.applicationContext
    }
}
```

**ContextWrapper 的实际应用：**
- `ContextThemeWrapper` — 给 View inflate 时注入不同主题
- `MutableContextWrapper` — 动态切换 base Context（WebView 预加载场景）
- 自定义 ContextWrapper 可以拦截/修改 Context 行为（如多语言切换）

### 延伸问题

- [[Activity启动流程]]
- [[内存优化-LeakCanary与内存抖动]]
- [[Hilt依赖注入原理]]

## 记忆锚点

Context = 装饰器模式。ContextImpl 干活，ContextWrapper 套壳。Activity 多了 Window 和 Theme 所以能弹窗，Application 活得久但没 UI 能力。
