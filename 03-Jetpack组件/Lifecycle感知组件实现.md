---
tags: [android, 面试, jetpack, lifecycle, A]
difficulty: A
frequency: high
---

# Lifecycle 感知组件实现

## 问题

> Jetpack Lifecycle 是如何感知 Activity/Fragment 生命周期的？LifecycleObserver 的事件分发机制是什么？为什么不再推荐 @OnLifecycleEvent 注解？

## 核心答案

Lifecycle 通过在 Activity/Fragment 中注入一个无 UI 的 ReportFragment 来监听生命周期回调，将回调转化为 Lifecycle.Event 分发给所有注册的 LifecycleObserver。ComponentActivity 直接实现了 LifecycleOwner，内部维护 LifecycleRegistry 管理状态机和观察者列表。不推荐 @OnLifecycleEvent 因为它依赖反射或 KAPT，新方案 DefaultLifecycleObserver/LifecycleEventObserver 是直接接口调用，性能更好。

## 深入解析

### 原理层

**核心类关系：**
```
LifecycleOwner (接口，提供 Lifecycle)
  └── ComponentActivity / Fragment

Lifecycle (抽象类，管理状态和观察者)
  └── LifecycleRegistry (实现类)
        ├── State: INITIALIZED → CREATED → STARTED → RESUMED
        ├── Event: ON_CREATE, ON_START, ON_RESUME, ON_PAUSE, ON_STOP, ON_DESTROY
        └── ObserverWithState[] (观察者列表)

LifecycleObserver (标记接口)
  ├── DefaultLifecycleObserver (推荐，有默认方法)
  └── LifecycleEventObserver (单方法回调)
```

**生命周期感知机制（Activity）：**
```java
// ComponentActivity.onCreate()
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // 注入 ReportFragment 监听生命周期
    ReportFragment.injectIfNeededIn(this);
}

// ReportFragment（无 UI Fragment）
public class ReportFragment extends Fragment {
    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }
    
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleOwner) {
            ((LifecycleOwner) activity).getLifecycle().handleLifecycleEvent(event);
        }
    }
}
// 注：API 29+ 直接通过 Activity.registerActivityLifecycleCallbacks 实现，不再需要 ReportFragment
```

**LifecycleRegistry 状态机：**
```
State:    INITIALIZED ←→ CREATED ←→ STARTED ←→ RESUMED
Event:         ON_CREATE↗   ON_START↗   ON_RESUME↗
               ON_DESTROY↙  ON_STOP↙    ON_PAUSE↙
```

**事件分发流程：**
```java
// LifecycleRegistry.handleLifecycleEvent()
public void handleLifecycleEvent(Event event) {
    State next = event.getTargetState();
    moveToState(next);
}

private void moveToState(State next) {
    mState = next;
    // 遍历所有观察者，同步到新状态
    sync();  // 按注册顺序正向/逆向分发事件
}

// 保证观察者不会错过中间状态
// 例如：观察者在 STARTED 时注册，会先收到 ON_CREATE 再收到 ON_START
```

**为什么弃用 @OnLifecycleEvent：**
```kotlin
// 旧方式（反射或 KAPT 生成适配器）
class MyObserver : LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onStart() { ... }  // 需要反射调用或生成 GeneratedAdapter
}

// 新方式（直接接口调用，零反射）
class MyObserver : DefaultLifecycleObserver {
    override fun onStart(owner: LifecycleOwner) { ... }
}
```

### 实战层

- **自定义 LifecycleOwner**：实现 LifecycleOwner 接口 + LifecycleRegistry，适用于自定义 View 或 Service
- **ProcessLifecycleOwner**：监听整个 App 的前后台切换（ON_START = 前台，ON_STOP = 后台）
- **LiveData 内部使用**：LiveData.observe() 内部注册 LifecycleObserver，STARTED 以上才分发数据
- **协程集成**：`lifecycleScope`、`repeatOnLifecycle` 都基于 Lifecycle 状态控制协程启停

```kotlin
// 监听 App 前后台
ProcessLifecycleOwner.get().lifecycle.addObserver(object : DefaultLifecycleObserver {
    override fun onStart(owner: LifecycleOwner) { /* App 进入前台 */ }
    override fun onStop(owner: LifecycleOwner) { /* App 进入后台 */ }
})
```

### 延伸问题

- [[ViewModel存活原理]]
- [[LiveData vs Flow选型]]
- [[协程作用域与结构化并发]]

## 记忆锚点

Lifecycle = 观察者模式 + 状态机。Activity 通过 ReportFragment（或 API 29+ 的回调）把生命周期事件喂给 LifecycleRegistry，Registry 再通知所有观察者。
