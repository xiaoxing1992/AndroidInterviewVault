---
tags: [android, 面试, jetpack, viewmodel, S]
difficulty: S
frequency: high
---

# ViewModel 存活原理

## 问题

> ViewModel 为什么能在 Activity 配置变更（如屏幕旋转）后存活？它的生命周期是如何管理的？ViewModelStore 和 ViewModelProvider 的关系是什么？

## 核心答案

ViewModel 存活的关键是 ViewModelStore 被保存在 Activity 的 NonConfigurationInstances 中，配置变更时 Activity 销毁重建但 NonConfigurationInstances 会被系统保留并传递给新 Activity。ViewModelProvider 负责从 ViewModelStore 中获取或创建 ViewModel，ViewModelStore 本质是一个 HashMap<String, ViewModel>。

## 深入解析

### 原理层

**存活机制核心链路：**
```
Activity 配置变更销毁时：
  ActivityThread.performDestroyActivity()
    → Activity.retainNonConfigurationInstances()
      → onRetainNonConfigurationInstance()
        → ViewModelStore 被保存到 NonConfigurationInstances

Activity 重建时：
  ActivityThread.performLaunchActivity()
    → Activity.attach(... lastNonConfigurationInstances ...)
      → 新 Activity 拿到旧的 NonConfigurationInstances
        → getViewModelStore() 返回之前保存的 ViewModelStore
          → ViewModel 还在里面，未被销毁
```

**关键源码：**
```java
// ComponentActivity
public ViewModelStore getViewModelStore() {
    if (mViewModelStore == null) {
        // 尝试从 NonConfigurationInstances 恢复
        NonConfigurationInstances nc = 
            (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            mViewModelStore = nc.viewModelStore;  // 恢复！
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();  // 首次创建
        }
    }
    return mViewModelStore;
}

// ViewModelStore 本质
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    
    final void put(String key, ViewModel viewModel) { mMap.put(key, viewModel); }
    final ViewModel get(String key) { return mMap.get(key); }
    
    public final void clear() {
        for (ViewModel vm : mMap.values()) { vm.clear(); }  // 调用 onCleared()
        mMap.clear();
    }
}
```

**ViewModelProvider 工作流程：**
```kotlin
// viewModels() 委托最终调用
ViewModelProvider(viewModelStore, factory).get(MyViewModel::class.java)

// ViewModelProvider.get()
fun <T : ViewModel> get(modelClass: Class<T>): T {
    val key = "androidx.lifecycle.ViewModelProvider.DefaultKey:${modelClass.canonicalName}"
    var viewModel = store.get(key)
    if (modelClass.isInstance(viewModel)) {
        return viewModel as T  // 已存在，直接返回
    }
    viewModel = factory.create(modelClass)  // 不存在，创建新的
    store.put(key, viewModel)
    return viewModel as T
}
```

**ViewModel 真正销毁的时机：**
```java
// ComponentActivity 构造函数中注册
getLifecycle().addObserver(new LifecycleEventObserver() {
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            if (!isChangingConfigurations()) {
                // 不是配置变更导致的销毁 → 真正清理
                getViewModelStore().clear();  // 调用所有 ViewModel.onCleared()
            }
        }
    }
});
```

### 实战层

- **Fragment 的 ViewModel**：Fragment 也有自己的 ViewModelStore，通过 `by viewModels()` 获取；`by activityViewModels()` 共享 Activity 级别的 ViewModel
- **SavedStateHandle**：ViewModel 配合 SavedStateHandle 可以在进程被杀后恢复数据（存入 Bundle）
- **自定义 Factory**：需要构造函数参数时，实现 `ViewModelProvider.Factory`
- **坑点**：ViewModel 不能持有 View/Activity 引用（生命周期不同步会泄漏）
- **Hilt 集成**：`@HiltViewModel` + `@Inject constructor` 自动注入依赖

```kotlin
// SavedStateHandle 用法
@HiltViewModel
class MyViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: Repository
) : ViewModel() {
    val query = savedStateHandle.getStateFlow("query", "")
}
```

### 延伸问题

- [[Lifecycle感知组件实现]]
- [[LiveData vs Flow选型]]
- [[Hilt依赖注入原理]]

## 记忆锚点

ViewModel 存活靠 NonConfigurationInstances — 配置变更时系统帮你把 ViewModelStore 这个"保险箱"从旧 Activity 搬到新 Activity。不是配置变更的销毁才真正 clear。
