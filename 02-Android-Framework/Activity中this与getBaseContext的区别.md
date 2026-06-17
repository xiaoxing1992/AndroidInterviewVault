---
tags: [android, 面试, framework, context, activity, A]
difficulty: A
frequency: high
---

# Activity 中 this 与 getBaseContext 的区别

## 问题

> Activity 里的 `this` 和 `getBaseContext()` 有什么区别？`baseContext` 是什么时候赋值的？为什么它们不一致？

## 核心答案

`this` 是 Activity 实例本身（ContextThemeWrapper），拥有 Window 和 Theme，是完整的 Context；`getBaseContext()` 返回的是 ContextWrapper 内部持有的 `mBase`，即 ContextImpl，是没有 Window/Theme 的裸 Context。两者不一致的原因是 Android 采用装饰器模式：ContextImpl 是真正干活的实现，ContextWrapper 是壳，Activity 继承了 ContextWrapper 壳，内部持有 ContextImpl 实例。`mBase` 的赋值发生在 `Activity.attach()` → `attachBaseContext(context)`，由 ActivityThread 在创建 Activity 时调用，开发者不应手动修改。

## 深入解析

### 原理层（源码级）

**继承体系与 this/baseContext 的关系：**

```
this (Activity 实例)
  │
  │  Activity extends ContextThemeWrapper
  │                extends ContextWrapper
  │                      extends Context
  │
  └── ContextWrapper 内部：
        Context mBase;   ← getBaseContext() 返回的就是这个

        mBase 是什么？
          → ContextImpl 实例
          → ContextImpl extends Context
          → 是真正实现资源访问、系统服务获取的类
```

```
Activity this                    getBaseContext() / baseContext
────────────                     ─────────────────────────────
Activity 实例                     ContextImpl 实例
├── Window (PhoneWindow)         ├── 无 Window
├── Theme                        ├── 无 Theme
├── setContentView()             ├── 无 UI 能力
├── startActivity() (直接调)      ├── startActivity() (需要 NEW_TASK)
└── AlertDialog ✅               └── AlertDialog ❌ (崩溃)

两者是不同的对象，但 this 内部委托 mBase 做实际工作
```

**mBase 是什么时候赋值的？**

`mBase` 的赋值链路要分三个组件来看：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ① Application 的 mBase 赋值
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ActivityThread.handleBindApplication()
  → data.info.makeApplication()
    → Instrumentation.newApplication()
      → app.attach(baseContext)                      ← Application 也是 ContextWrapper
        → Application.attachBaseContext(base)
          → super.attachBaseContext(base)             ← ContextWrapper.attachBaseContext
            → mBase = base;                          ← 赋值在这里！
                                                     ← base 是 ContextImpl

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ② Activity 的 mBase 赋值
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ActivityThread.performLaunchActivity()
  → Instrumentation.newActivity()                    ← 反射创建 Activity
  → activity.attach(context, ...)                    ← 系统调用，不是开发者
    → Activity.attachBaseContext(context)
      → super.attachBaseContext(context)              ← ContextWrapper.attachBaseContext
        → mBase = context;                           ← 赋值在这里！
                                                     ← context 是 ContextImpl（带 Theme 的包装）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 ③ Service 的 mBase 赋值
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ActivityThread.handleCreateService()
  → service.attach(context, ...)
    → Service.attachBaseContext(context)
      → super.attachBaseContext(context)              ← ContextWrapper.attachBaseContext
        → mBase = context;                           ← 赋值在这里！
```

**Activity.attach() 源码（关键）：**

```java
// Activity.java
final void attach(Context context, ActivityThread aThread, Instrumentation instr,
        IBinder token, int ident, Application application, Intent intent,
        ActivityInfo info, CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback) {

    // ① 调用 ContextWrapper.attachBaseContext → mBase = context (ContextImpl)
    attachBaseContext(context);

    // ② 创建 Window（这就是为什么 Activity 能弹 Dialog）
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);  // Activity 作为 Window 的回调

    // ③ 设置 WindowManager
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);

    // ④ 设置主题
    // ContextImpl 在创建时就已经为 Activity 包装了 Theme 信息
}
```

**ContextWrapper.attachBaseContext 源码：**

```java
// ContextWrapper.java
protected void attachBaseContext(Context base) {
    if (mBase != null) {
        throw new IllegalStateException("Base context already set");
        // 只能赋值一次！重复调用会崩溃
    }
    mBase = base;  // ← mBase 在这里赋值
}
```

**ContextImpl 是怎么创建并传入的？**

```java
// ActivityThread.performLaunchActivity()
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ① 创建 Activity 的 ContextImpl
    ContextImpl appContext = createBaseContextForActivity(r);
    // 内部：
    //   → ContextImpl.createActivityContext(...)
    //   → 创建时携带了 Theme、Display、ClassLoader 等信息

    // ② 反射创建 Activity 实例
    Activity activity = mInstrumentation.newActivity(
            component.getClassLoader(), component.getClassName(), r.intent);

    // ③ 将 ContextImpl 传入 attach → attachBaseContext → mBase = context
    activity.attach(appContext, this, getInstrumentation, r.token,
            r.ident, application, r.intent, r.activityInfo, title, r.parent,
            r.embeddedID, r.lastNonConfigurationInstances, config,
            r.referrer, r.voiceInteractor, window, r.configCallback);

    return activity;
}
```

### 为什么 this 和 getBaseContext() 不一致？

```
根本原因：装饰器模式

1. ContextImpl 是真正的实现类
   → getResources()、getSystemService()、startActivity() 都在这里实现

2. ContextWrapper 是装饰器壳
   → 内部持有 mBase（ContextImpl）
   → 所有方法委托给 mBase

3. Activity 继承了 ContextWrapper
   → Activity 本身是"壳"
   → mBase 是壳里面包着的 ContextImpl

4. this = Activity 实例（壳 + 附加能力）
   → 附加了 Window、Theme、UI 能力

5. getBaseContext() = mBase = ContextImpl（裸实现）
   → 没有 Window、没有 Theme

为什么不直接继承 ContextImpl？
  → 因为 ContextImpl 太庞大，直接继承会导致子类耦合过重
  → 装饰器模式让 Activity/Service/Application 可以各自扩展不同能力
  → Activity 加 Window + Theme
  → Service 不需要 UI
  → Application 不需要 UI 也不需要临时生命周期
```

**方法调用对比：**

```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // ========== 能力相同的调用 ==========
        // this 和 baseContext 都委托给同一个 ContextImpl 实现
        this.getResources()           // → mBase.getResources()
        baseContext.getResources()    // → 直接就是 ContextImpl.getResources()
        // 结果相同

        this.getSystemService(...)    // → mBase.getSystemService(...)
        baseContext.getSystemService(...) // → ContextImpl.getSystemService(...)
        // 结果相同

        // ========== 能力不同的调用 ==========
        // 弹 Dialog — 需要 Window Token
        AlertDialog.Builder(this).show()         // ✅ Activity 有 Window
        AlertDialog.Builder(baseContext).show()   // ❌ 崩溃：没有 Window Token

        // 设置主题 — 需要 Theme
        this.setTheme(R.style.AppTheme)          // ✅ ContextThemeWrapper 覆写了
        baseContext.setTheme(R.style.AppTheme)   // ⚠️ 能调但不完整

        // inflate 带主题的布局
        LayoutInflater.from(this).inflate(...)           // ✅ 有主题
        LayoutInflater.from(baseContext).inflate(...)    // ⚠️ 无主题，样式可能丢失

        // startActivity
        this.startActivity(intent)                 // ✅ 直接调用
        baseContext.startActivity(intent)           // ❌ 需要 FLAG_ACTIVITY_NEW_TASK
    }
}
```

**AlertDialog 为什么用 baseContext 会崩溃？**

```java
// AlertDialog.show() 内部
public void show() {
    // 需要获取 Window Token 来添加窗口
    mWindow.getDecorView()  // ...
    // 内部通过 Context 获取 token:
    //   如果是 Activity → 拿到 Activity 的 Window Token → 成功
    //   如果是 ContextImpl → 没有 Window Token → token 为 null → 崩溃
}

// WindowManagerImpl.addView 内部
public void addView(View view, ViewGroup.LayoutParams params) {
    ...
    // 需要 parentWindow 的 token
    // ContextImpl 没有 parentWindow → token null → BadTokenException
}
```

### 实战层（项目经验）

**各 Context 来源的使用场景：**

| 场景 | 推荐用什么 | 原因 |
|------|-----------|------|
| 弹 Dialog | `this` (Activity) | 需要 Window Token |
| inflate 带主题布局 | `this` (Activity) | 需要正确的 Theme |
| 创建 View | `this` (Activity) | 需要正确的 Context |
| startActivity | `this` (Activity) | 不需要 NEW_TASK flag |
| 注册系统服务回调 | `applicationContext` | 避免 Activity 泄漏 |
| 单例持有 Context | `applicationContext` | 生命周期最长 |
| getBaseContext() | **几乎不用** | 给子类覆写用的，外部不直接使用 |

**常见错误：**

```kotlin
// ❌ 单例持有 Activity this → 内存泄漏
object ImageLoader {
    lateinit var context: Context
    fun init(context: Context) {
        this.context = context  // 如果传入 Activity，Activity 销毁后无法回收
    }
}

// ✅ 使用 applicationContext
object ImageLoader {
    lateinit var context: Context
    fun init(context: Context) {
        this.context = context.applicationContext  // 生命周期与进程一致
    }
}
```

### 延伸问题

- [[Context体系与ContextWrapper]] — 完整的 Context 继承体系与装饰器模式
- [[Application原理与实践]] — Application 的 attachBaseContext 调用时机
- [[Activity启动流程]] — Activity.attach() 在启动流程中的位置
- [[内存优化-LeakCanary与内存抖动]] — Activity Context 泄漏问题

## 记忆锚点

`this` 是"穿了衣服的人"（Activity + Window + Theme），`getBaseContext()` 是"剥了壳的花生"（裸 ContextImpl）。壳和内容是两个对象——装饰器模式决定的。`mBase` 在 `attachBaseContext()` 里赋值，只能赋一次，由系统在创建组件时调用，开发者别碰。
