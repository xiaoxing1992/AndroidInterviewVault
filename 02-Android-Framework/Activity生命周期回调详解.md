---
tags: [android, 面试, framework, activity, lifecycle, B]
difficulty: B
frequency: high
---

# Activity 生命周期回调详解

## 问题

> 启动一个 Activity 会经历哪些生命周期回调？A 启动 B 的回调顺序是什么？按返回键回来呢？onSaveInstanceState 什么时候调用？onPause 里做耗时操作会怎样？

## 核心答案

正常生命周期：onCreate → onStart → onResume → 运行中 → onPause → onStop → onDestroy。A 启动 B 的关键顺序：A.onPause → B.onCreate → B.onStart → B.onResume → A.onStop。系统优先保证新页面尽快显示，所以 A.onStop 延后执行。按返回键回来：B.onPause → A.onRestart → A.onStart → A.onResume → B.onStop → B.onDestroy。onSaveInstanceState 在系统认为有销毁风险时调用（主动 finish 不调），API 28+ 在 onPause 之后、onStop 之前。

## 深入解析

### 原理层（源码级）

**各生命周期回调的触发源：**

```
每个生命周期回调都由 ActivityThread 中的 TransactionExecutor 调度

ActivityThread.handleResumeActivity()
  → performResumeActivity()
    → Activity.performResume()
      → mInstrumentation.callActivityOnResume(activity)
        → activity.onResume()

ActivityThread.handlePauseActivity()
  → performPauseActivity()
    → Activity.performPause()
      → mInstrumentation.callActivityOnPause(activity)
        → activity.onPause()

ActivityThread.handleStopActivity()
  → performStopActivity()
    → Activity.performStop()
      → activity.onStop()

ActivityThread.handleDestroyActivity()
  → performDestroyActivity()
    → Activity.performDestroy()
      → activity.onDestroy()
```

**从零启动一个 Activity（冷启动完整链路）：**

```
Zygote fork → ActivityThread.main()
  → handleBindApplication()
    → Application.attachBaseContext()
    → ContentProvider.onCreate()
    → Application.onCreate()
        ↓
  AMS 调度启动 Activity
        ↓
  ActivityThread.performLaunchActivity()
    → Activity.attach()            ← 绑定 Context/Window
    → Instrumentation.callActivityOnCreate()
      → Activity.onCreate()        ← setContentView、数据初始化
    → Activity.onStart()           ← 变为可见
    → Activity.onResume()          ← 获取焦点，可交互
```

### 各场景下的回调顺序

**场景 1：A 启动 B（标准模式）**

```
A 运行中...
    ↓ startActivity(B)
A.onPause()              ← A 先失去焦点
    ↓
B.attach()               ← B 开始创建
B.onCreate()             ← B 页面创建
B.onStart()              ← B 变为可见
B.onResume()             ← B 获取焦点，可交互
    ↓
A.onStop()               ← A 变为不可见（在 B.onResume 之后）
```

为什么 A.onStop 在 B.onResume 之后？
→ 系统优先保证新页面尽快显示，A 的 onStop 延后执行

源码依据（ActivityThread）：

```java
// ActivityThread.handlePauseActivity()
// A.onPause 先执行

// 然后 AMS 调度 B 的启动：
// ActivityThread.performLaunchActivity() → B.onCreate
// ActivityThread.handleResumeActivity()  → B.onResume

// 最后 AMS 回调 A 的 stop：
// ActivityThread.handleStopActivity()    → A.onStop
// （此时 A 已经不可见了）
```

**场景 2：B 按返回键回 A**

```
B 运行中...
    ↓ onBackPressed()
B.onPause()              ← B 失去焦点
    ↓
A.onRestart()            ← A 从停止状态恢复（不是 onCreate）
A.onStart()              ← A 变为可见
A.onResume()             ← A 重新获取焦点
    ↓
B.onStop()               ← B 不可见
B.onDestroy()            ← B 销毁
```

**场景 3：A 跳转到 B（B 是透明/Dialog 主题）**

```
A.onPause()
B.onCreate()
B.onStart()
B.onResume()
A 不调 onStop             ← A 仍然可见（被透明 B 遮挡但未完全不可见）
```

**场景 4：配置变更（屏幕旋转，默认行为）**

```
onPause()
onSaveInstanceState()    ← 保存状态，Bundle 非空
onStop()
onDestroy()
onCreate(savedInstanceState)  ← 重建，Bundle 非空
onStart()
onRestoreInstanceState(savedInstanceState)  ← 恢复状态
onResume()
```

声明 `android:configChanges="orientation|screenSize"` 则不重建：

```
onConfigurationChanged()  ← 只回调这个，不走生命周期
```

**场景 5：进程被杀后恢复**

```
onCreate(savedInstanceState)       ← savedInstanceState 非空
onStart()
onRestoreInstanceState(savedInstanceState)  ← 在 onStart 和 onResume 之间
onResume()
```

**场景 6：A 跳 B，B 跳 C，按两次返回**

```
跳转过程：
  A.onPause → B.onCreate → B.onStart → B.onResume → A.onStop
  B.onPause → C.onCreate → C.onStart → C.onResume → B.onStop

第一次返回（C → B）：
  C.onPause → B.onRestart → B.onStart → B.onResume → C.onStop → C.onDestroy

第二次返回（B → A）：
  B.onPause → A.onRestart → A.onStart → A.onResume → B.onStop → B.onDestroy
```

### onSaveInstanceState 与 onRestoreInstanceState

**调用时机：**

```
onSaveInstanceState:
  → API 28+：onPause 之后、onStop 之前
  → API 28 之前：onStop 之后
  → 系统认为 Activity 有被销毁风险时才调用
  → 主动 finish() 不会调用
  → 按 Home 键、屏幕旋转、内存不足时会调用

onRestoreInstanceState:
  → 在 onStart 之后、onResume 之前
  → 只有 Bundle 非空时才调用
  → 和 onCreate 的区别：onCreate 需要自己判空，onRestoreInstanceState 一定非空
```

```kotlin
// onCreate 中恢复（需要判空）
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val name = savedInstanceState?.getString("name")  // 需要判空，首次创建为 null
}

// onRestoreInstanceState 中恢复（无需判空，一定有值）
override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    super.onRestoreInstanceState(savedInstanceState)
    val name = savedInstanceState.getString("name")   // 一定非空，只有恢复时才调用
}
```

**哪些 View 的状态会被自动保存？**

```
自动保存的条件：
  → View 有 android:id
  → View 实现了 onSaveInstanceState() / onRestoreInstanceState()
  → 常见：EditText（文本）、CheckBox（勾选状态）、ScrollView（滚动位置）

不会自动保存：
  → 没有 id 的 View
  → 自定义 View 没有覆写 onSaveInstanceState
```

**不保存状态的情况：**

```
以下情况 onSaveInstanceState 不会被调用：
  → 主动调用 finish()
  → 用户主动按返回键（系统认为是用户意图，不需要保存）
  → Activity 首次创建（还没有状态可保存）

以下情况会调用：
  → 按 Home 键回到桌面
  → 从最近任务切到其他 App
  → 屏幕旋转
  → 系统内存不足回收后台 Activity
  → 切换到其他 Activity（A.onPause 之后系统可能调用）
```

### 各状态与回调的对应关系

```
┌─────────────┬─────────────────────────────────────────────────┐
│    状态      │          对应回调                                │
├─────────────┼─────────────────────────────────────────────────┤
│  Created    │ onCreate() 已调用，onDestroy() 还没调用           │
│  Started    │ onStart() 已调用，可见但不可交互                  │
│  Resumed    │ onResume() 已调用，前台可交互                     │
│  Paused     │ onPause() 已调用，部分遮挡（如 Dialog 遮挡）      │
│  Stopped    │ onStop() 已调用，完全不可见                       │
│  Destroyed  │ onDestroy() 已调用，即将被回收                    │
└─────────────┴─────────────────────────────────────────────────┘

特殊状态：
  onRestart() → 从 Stopped 回到 Started 时调用，不经过 onCreate
  onSaveInstanceState() → 系统认为有销毁风险时，在 onPause~onStop 之间调用
  onRestoreInstanceState() → 恢复时在 onStart~onResume 之间调用
```

### 实战层（项目经验）

**onPause 里做耗时操作的后果：**

```kotlin
// ❌ 错误：onPause 里做耗时操作
override fun onPause() {
    super.onPause()
    saveDataToDatabase()  // 假设耗时 500ms
    // 后果：B 的 onCreate/onResume 必须等 A.onPause 完成
    // → B 显示卡顿 500ms
    // → 严重时触发 ANR
}

// ✅ 正确：onPause 只做轻量操作
override fun onPause() {
    super.onPause()
    // 只保存关键状态，大数据写入交给后台线程
    pauseMediaPlayer()
}
```

**生命周期回调的最佳实践：**

```
onCreate:
  → setContentView、数据初始化、绑定观察者
  → 不要做耗时操作（影响启动速度）

onStart:
  → 注册广播接收器、开始定位

onResume:
  → 恢复动画、刷新数据
  → 可交互状态的准备

onPause:
  → 暂停动画、暂停定位
  → 只做轻量操作，不要做耗时操作（阻塞 B 的启动）

onStop:
  → 释放资源、取消网络请求
  → 保存数据到数据库（此时 B 已显示，可以做耗时操作）

onDestroy:
  → 最终清理（但不保证一定被调用，不要依赖它做关键操作）
```

**不同版本的 API 差异：**

```
API < 23 (Android 6.0 之前):
  onSaveInstanceState 在 onStop 之后调用

API 23 ~ 27:
  onSaveInstanceState 在 onPause 之后、onStop 之前调用

API 28+ (Android 9.0):
  onSaveInstanceState 在 onPause 之后、onStop 之前
  （与 23-27 相同，但官方文档更明确了这个时序）
```

### 延伸问题

- [[Activity启动流程]] — Activity 启动的完整源码链路
- [[App冷启动全链路]] — 冷启动时 Application 和 Activity 的初始化顺序
- [[View绘制三大流程]] — onCreate 中 setContentView 触发的绘制流程
- [[性能优化]] — 生命周期回调中的性能优化切入点

## 记忆锚点

生命周期口诀：**"创建可见可交互，部分遮挡全不见，最终销毁"** → onCreate → onStart → onResume → onPause → onStop → onDestroy。A 启动 B 的关键：A 先让出（onPause）→ B 创建并显示 → A 再消失（onStop）。就像两人换座位：先让的人先站起来（onPause），新人坐下后（onResume），让的人才走开（onStop）。
