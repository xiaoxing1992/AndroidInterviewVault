---
tags: [android, 面试, framework, activity, S]
difficulty: S
frequency: high
---

# Activity 启动流程

## 问题

> 请详细描述从调用 startActivity() 到目标 Activity 的 onCreate() 被调用，中间经历了哪些关键步骤？涉及哪些系统服务和进程间通信？

## 核心答案

startActivity 经过 Instrumentation 发起 Binder 调用到 system_server 进程的 AMS（ActivityManagerService），AMS 进行权限检查、栈管理后，如果目标进程不存在则通过 Zygote fork 新进程。目标进程启动后通过 attachApplication 回调 AMS，AMS 再通过 ApplicationThread（Binder）通知目标进程创建 Activity，最终在主线程通过 Handler 调度执行 Activity 的生命周期。

## 深入解析

### 原理层

**完整调用链（简化）：**

```
App 进程                          system_server 进程              Zygote 进程
─────────                        ──────────────────              ────────────
startActivity()
  → Instrumentation.execStartActivity()
    → ATMS.startActivity() ──Binder──→ ActivityTaskManagerService
                                         → startActivityAsUser()
                                         → ActivityStarter.execute()
                                         → 权限检查/Intent 解析
                                         → Task 栈管理
                                         → 目标进程是否存在？
                                           ├─ 存在 → realStartActivityLocked()
                                           └─ 不存在 → Process.start() ──Socket──→ fork()
                                                                                    ↓
新进程启动 ←──────────────────────────────────────────────────────────── 子进程
  ActivityThread.main()
    → Looper.prepareMainLooper()
    → ActivityThread.attach()
      → AMS.attachApplication() ──Binder──→ attachApplicationLocked()
                                              → realStartActivityLocked()
    ← ClientTransaction ──Binder──────────── scheduleTransaction()
  TransactionExecutor.execute()
    → LaunchActivityItem.execute()
      → ActivityThread.handleLaunchActivity()
        → performLaunchActivity()
          → Instrumentation.newActivity() // 反射创建 Activity
          → Activity.attach()             // 绑定 Window/Context
          → Instrumentation.callActivityOnCreate()
            → Activity.onCreate()
```

**关键类：**
- `ActivityTaskManagerService (ATMS)` — Android 10+ 从 AMS 拆分出来，管理 Activity 和 Task
- `ActivityStarter` — 处理 Intent 解析、权限检查、启动模式
- `ActivityRecord` — Activity 在 AMS 中的数据结构
- `TaskRecord / Task` — 任务栈
- `ClientTransaction` — 封装生命周期变化的事务
- `LaunchActivityItem` — 具体的启动回调项
- `ActivityThread` — App 进程的主线程管理类
- `ApplicationThread` — App 进程暴露给 AMS 的 Binder 接口

**启动模式影响：**
- standard → 新建 ActivityRecord 入栈
- singleTop → 栈顶复用，调用 onNewIntent
- singleTask → 栈内复用，清除其上的 Activity
- singleInstance → 独占一个 Task

### 实战层

- **性能影响**：整个流程涉及 2 次跨进程 Binder 调用（App→AMS、AMS→App），如果需要 fork 进程还有 Socket 通信
- **启动优化切入点**：Application.onCreate()、Activity.onCreate() 中的耗时操作是优化重点
- **面试追问**：为什么用 Socket 通信 Zygote 而不用 Binder？因为 fork 要求单线程，Binder 是多线程模型
- **Android 12+ 变化**：引入 Splash Screen API，在 Activity 创建前显示启动画面

### 延伸问题

- [[Android进程创建方式-fork-execve与Zygote]]
- [[Zygote进程启动与App进程孵化]]
- [[App冷启动全链路]]
- [[Binder通信原理]]
- [[启动优化-耗时分析与优化手段]]

## 记忆锚点

startActivity 三步走：App→AMS（我要启动）、AMS 检查+调度（行不行/进程在不在）、AMS→App（你可以创建了）。两次 Binder 跨进程。
