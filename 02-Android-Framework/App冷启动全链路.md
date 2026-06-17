---
tags: [android, 面试, framework, 启动, binder, socket, S]
difficulty: S
frequency: high
---

# App 冷启动全链路

## 问题

> 从用户点击桌面图标到看到 App 第一帧画面，中间经历了哪些阶段？涉及哪些组件？跨进程通信过程是怎样的？每个阶段的耗时瓶颈在哪里？如何度量和优化？

## 核心答案

冷启动涉及 5 个进程、3 种 IPC 机制：Launcher →（Binder）→ AMS →（Socket）→ Zygote fork 新进程 →（Binder）→ AMS 回调 → Activity 创建 → View 绘制 → 首帧上屏。关键组件：Launcher、ATMS/AMS（system_server）、Zygote、ActivityThread（新 App 进程）、SurfaceFlinger。通信方式：Launcher↔AMS 用 Binder，AMS→Zygote 用 Socket（fork 要求单线程），App↔AMS 用 Binder，App→SurfaceFlinger 用 mmap（BufferQueue 零拷贝）。耗时瓶颈在 Application.onCreate（SDK 初始化）和 Activity 首帧渲染。

## 深入解析

### 原理层（源码级）

**涉及的进程与组件：**

```
进程 1：Launcher（App 进程）
  → Activity.startActivity()

进程 2：system_server
  → ATMS (ActivityTaskManagerService) — Activity 管理（Android 10+ 从 AMS 拆出）
  → AMS (ActivityManagerService) — 进程管理
  → PMS (PackageManagerService) — 解析 Intent、权限检查
  → WMS (WindowManagerService) — 窗口管理
  → InputManagerService — 输入事件分发

进程 3：Zygote
  → ZygoteServer — Socket 监听
  → ZygoteConnection — 处理 fork 请求
  → Zygote.forkAndSpecialize() — 实际 fork

进程 4：新 App 进程
  → ActivityThread — 主线程入口
  → Instrumentation — 生命周期回调
  → LoadedApk — APK 信息和 ClassLoader
  → ViewRootImpl — View 绘制起点
  → RenderThread — 硬件加速绘制

进程 5：SurfaceFlinger
  → 合成所有应用的 Surface
  → 送显到屏幕

辅助进程：servicemanager
  → Binder 服务的注册中心，所有 Binder 调用通过它发现服务
```

---

#### 阶段 1：Launcher → AMS（Binder）

```
Launcher 进程                        system_server 进程
──────────────                       ──────────────────
用户点击图标
  → onClick()
    → startActivity(intent)
      → Instrumentation.execStartActivity()
        → ATMS.startActivity() ──Binder──→ ActivityTaskManagerService
                                              │
                                        startActivityAsUser()
                                              │
                                        ActivityStarter.execute()
                                              │
                                        解析 Intent / 权限检查
                                        创建 Task / ActivityRecord
                                              │
                                        当前 Activity (Launcher) 需要暂停
                                              │
                                        通知 Launcher onPause
                                        ←──Binder── IApplicationThread.schedulePauseActivity
```

通信方式：Binder
- Launcher 通过 `IActivityTaskManager`（Binder Proxy）调用 AMS
- AMS 通过 `IApplicationThread`（Binder Proxy）回调 Launcher 暂停

关键组件：
- `Instrumentation` — App 进程中发起启动的工具类
- `ATMS (ActivityTaskManagerService)` — Activity 管理（Android 10+ 从 AMS 拆出）
- `ActivityStarter` — 解析 Intent、权限检查、启动模式处理
- `ActivityRecord` — Activity 在系统侧的数据结构

---

#### 阶段 2：AMS → Zygote（Socket）

```
system_server 进程                   Zygote 进程
──────────────────                   ────────────
AMS 发现目标进程不存在
  → Process.start()
    → ZygoteProcess.start()
      → zygoteSendArgsAndGetResult()
        → 通过 LocalSocket 发送参数
                                         ──Socket──→
                                        ZygoteServer.runSelectLoop()
                                          → ZygoteConnection.processOneCommand()
                                            → Zygote.forkAndSpecialize()
                                              │
                                            fork()
                                            ┌────┴────┐
                                          父进程      子进程
                                         (Zygote)   (新 App)
                                          返回 pid   handleChildProc()
                                          继续等待      → RuntimeInit.zygoteInit()
                                                     → ActivityThread.main()
```

通信方式：Socket（不用 Binder 的原因：fork 要求单线程，Binder 是多线程模型）

关键组件：
- `ZygoteProcess` — AMS 侧用于与 Zygote 通信的封装
- `ZygoteServer` — Zygote 侧监听 Socket 请求
- `ZygoteConnection` — 处理单次 fork 请求
- `Zygote.forkAndSpecialize()` — 实际的 fork 调用

传递的参数：

```java
String[] args = {
    "--runtime-flags", "0",
    "--uid", "10123",                    // App uid
    "--gid", "10123",
    "--gids", "3003,3007,...",           // 附加组（网络权限等）
    "--nice-name", "com.example.app",    // 进程名
    "--app-data-dir", "/data/data/com.example.app",
    "android.app.ActivityThread",        // 入口类
    "com.example.app.MainActivity"       // 入口 Activity
};
```

---

#### 阶段 3：新进程初始化 → 回调 AMS（Binder）

```
新 App 进程                         system_server 进程
───────────                         ──────────────────
子进程启动
  → handleChildProc()
    → RuntimeInit.zygoteInit()
      → ActivityThread.main()
        → Looper.prepareMainLooper()
        → ActivityThread.attach()
          → AMS.attachApplication() ──Binder──→ attachApplicationLocked()
                                                    │
                                              获取 ApplicationInfo
                                              安排 Application 初始化
                                              安排 ContentProvider 安装
                                              安排 Activity 启动
                                                    │
                                              IApplicationThread.bindApplication()
                                                    ↓
          ←──Binder── handleBindApplication(AppBindData)
```

通信方式：Binder
- App 进程通过 `IActivityManager`（Binder Proxy）向 AMS 报到
- AMS 通过 `IApplicationThread`（Binder Proxy）下发初始化数据

关键组件：
- `ActivityThread` — App 进程的主线程管理类，main() 是 App 的入口
- `IApplicationThread` — App 暴露给 AMS 的 Binder 接口（双向通信的 App 端）
- `AppBindData` — AMS 下发的初始化数据（Application 信息、ContentProvider 列表等）

---

#### 阶段 4：Application 初始化（进程内）

```
新 App 进程（内部，无跨进程）
────────────────────────────
handleBindApplication(AppBindData data)
  │
  ├─ ① 创建 Application 对象
  │    → data.info.makeApplication()
  │    → Instrumentation.newApplication()
  │    → app.attachBaseContext(context)
  │
  ├─ ② 安装 ContentProvider
  │    → installContentProviders(app, data.providers)
  │    → 遍历每个 Provider：
  │        → 反射创建实例
  │        → provider.attachInfo() → provider.onCreate()
  │        → AMS.publishContentProviders()   ← Binder，发布到 AMS
  │
  └─ ③ Application.onCreate()
       → mInstrumentation.callApplicationOnCreate(app)
```

通信方式：进程内 + 一次 Binder 调用（publishContentProviders 到 AMS）

---

#### 阶段 5：AMS 调度 Activity 生命周期（Binder）

```
system_server 进程                   新 App 进程
──────────────────                   ──────────
attachApplicationLocked() 后续：
  → realStartActivityLocked()
    → ClientTransaction
      → addCallback(LaunchActivityItem)
      → setLifecycleStateRequest(ResumeActivityItem)
    → IApplicationThread.scheduleTransaction()
                                           ──Binder──→
                                      ActivityThread.ApplicationThread
                                        → TransactionExecutor.execute()
                                          → LaunchActivityItem.execute()
                                            → handleLaunchActivity()
                                              → performLaunchActivity()
                                                → activity.attach()   ← mBase 赋值
                                                → activity.onCreate() ← setContentView
                                          → ResumeActivityItem.execute()
                                            → handleResumeActivity()
                                              → activity.onStart()
                                              → activity.onResume()
```

通信方式：Binder
- AMS 通过 `IApplicationThread.scheduleTransaction()` 下发生命周期事务
- `ClientTransaction` 封装了要执行的生命周期变化

关键组件：
- `ClientTransaction` — 生命周期事务封装
- `LaunchActivityItem` — 创建 Activity 的回调项
- `ResumeActivityItem` — 执行 onResume 的请求
- `TransactionExecutor` — 执行事务的调度器

---

#### 阶段 6：View 绘制 → 首帧上屏

```
新 App 进程                         SurfaceFlinger 进程
───────────                         ──────────────────
Activity.onResume()
  → ViewRootImpl.performTraversals()
    → performMeasure()  → 测量
    → performLayout()   → 布局
    → performDraw()     → 绘制
      → Canvas 绑定 Surface
      → 硬件加速：RenderThread 异步绘制
        → OpenGL/Vulkan 指令提交
                                          ←─ BufferQueue (mmap 共享内存) ─→
                                        SurfaceFlinger
                                          → 合成所有 Layer
                                          → 送显（Display HAL）
                                          → 用户看到第一帧 (TTID)
```

通信方式：mmap 共享内存（BufferQueue，零拷贝）

---

### 通信方式汇总

```
┌─────────────────────────────┬──────────────┬────────────────────────┐
│          路径                │  通信方式     │        说明             │
├─────────────────────────────┼──────────────┼────────────────────────┤
│ Launcher → AMS              │  Binder      │ startActivity()        │
│ AMS → Launcher              │  Binder      │ schedulePauseActivity  │
│ AMS → Zygote                │  Socket      │ fork 请求（单线程安全）  │
│ Zygote fork 子进程           │  fork()      │ 进程创建               │
│ App → AMS                   │  Binder      │ attachApplication()    │
│ AMS → App                   │  Binder      │ bindApplication()      │
│ App → AMS                   │  Binder      │ publishContentProviders│
│ AMS → App                   │  Binder      │ scheduleTransaction()  │
│ App → SurfaceFlinger        │  mmap        │ BufferQueue 零拷贝     │
│ App ↔ SystemService         │  Binder      │ getSystemService()     │
└─────────────────────────────┴──────────────┴────────────────────────┘
```

### 时间线与耗时

```
时间      组件                 通信         耗时          可优化
─────    ────                ────         ────          ──────
0ms      Launcher            Binder       ~10ms         可（Intent 精简）
10ms     AMS/ATMS            内部         ~20ms         不可（系统逻辑）
30ms     AMS → Zygote        Socket       ~10ms         不可
40ms     Zygote fork         fork()       ~100-200ms    不可（COW 已优化）
200ms    App 进程启动          Binder       ~10ms         不可
210ms    attachApplication    Binder       ~5ms          不可
220ms    ContentProvider      进程内       ~50-200ms     高（合并/App Startup）
350ms    Application.onCreate 进程内       ~100-500ms    高（懒加载/异步）
600ms    Activity.onCreate    进程内       ~50-200ms     高（布局优化）
750ms    首帧绘制              mmap         ~50-100ms     中（布局层级）
850ms    TTID（首帧可见）     -            -             -
```

各阶段耗时分布（典型值）：

| 阶段 | 耗时 | 可优化性 |
|------|------|----------|
| 进程 fork | 100-200ms | 不可优化 |
| Application 初始化 | 200-800ms | 高（主要优化点） |
| Activity 创建+渲染 | 100-500ms | 中 |
| 数据加载 | 变化大 | 高（异步/缓存） |

**Zygote 预加载机制：**
- Zygote 预加载了常用类（~4000个）和资源（Framework resources）
- fork 时通过 COW（Copy-On-Write）共享这些内存页
- 这就是为什么 App 进程创建比从零启动快得多

### 实战层（项目经验）

**系统度量方式：**

```bash
# Logcat 自动输出
ActivityTaskManager: Displayed com.example/.MainActivity: +850ms

# 对应代码位置
ActivityRecord.reportFullyDrawn()  # TTFD
ActivityMetricsLogger              # 系统级统计
```

**度量工具：**

```kotlin
// 1. reportFullyDrawn() — 标记 TTFD
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel.dataLoaded.observe(this) {
            reportFullyDrawn()
        }
    }
}

// 2. Perfetto/Systrace 分析
// 3. Firebase Performance Monitoring
// 4. 自定义埋点
object LaunchTimer {
    var startTime = 0L
    fun start() { startTime = SystemClock.elapsedRealtime() }
    fun end(tag: String) { Log.d("Launch", "$tag: ${elapsed()}ms") }
}
```

**优化手段清单：**
1. **Application 阶段**：SDK 懒加载/异步初始化/App Startup 合并 Provider
2. **Activity 阶段**：减少布局层级、ViewStub 延迟加载、异步 inflate
3. **数据阶段**：预加载/缓存/并行请求
4. **系统级**：Baseline Profile（预编译热路径）、启动画面优化

**Baseline Profile（Android 12+）：**

```kotlin
// 告诉 ART 哪些代码是启动热路径，提前 AOT 编译
@get:Rule
val rule = BaselineProfileRule()

@Test
fun generateBaselineProfile() {
    rule.collect("com.example.app") {
        startActivityAndWait()
        // 模拟启动路径操作
    }
}
```

### 延伸问题

- [[Android进程创建方式-fork-execve与Zygote]] — 为什么 fork 后不 execve
- [[Zygote进程启动与App进程孵化]] — Zygote 预加载与 fork 细节
- [[Activity启动流程]] — AMS 调度 Activity 的完整源码链路
- [[启动优化-耗时分析与优化手段]] — 各阶段优化手段详解
- [[ContentProvider启动时序]] — 为什么 Provider 先于 Application.onCreate
- [[Binder通信原理]] — Binder 一次拷贝原理
- [[Application原理与实践]] — Application 创建与初始化
- [[Activity生命周期回调详解]] — 启动过程中的生命周期时序

## 记忆锚点

冷启动五进程三通信：Launcher（起点）→ Binder → AMS（调度中心）→ Socket → Zygote（fork 孵化）→ Binder → App 进程（ActivityThread）→ mmap → SurfaceFlinger（送显）。Binder 用于正常 IPC，Socket 用于 fork 前（单线程安全），mmap 用于图形 buffer（零拷贝）。
