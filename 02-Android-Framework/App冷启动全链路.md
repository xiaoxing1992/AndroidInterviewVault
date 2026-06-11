---
tags: [android, 面试, framework, 启动, S]
difficulty: S
frequency: high
---

# App 冷启动全链路

## 问题

> 从用户点击桌面图标到看到 App 第一帧画面，中间经历了哪些阶段？每个阶段的耗时瓶颈在哪里？如何度量和优化？

## 核心答案

冷启动全链路：Launcher 进程 startActivity → AMS 处理 → Zygote fork 新进程 → Application 初始化 → Activity 创建 → View 绘制 → 首帧上屏。耗时主要集中在三个阶段：进程创建（~200ms，不可优化）、Application.onCreate（SDK 初始化）、Activity 首帧渲染（布局复杂度）。度量指标是 TTID（Time To Initial Display）和 TTFD（Time To Full Display）。

## 深入解析

### 原理层

**完整时间线：**
```
T0: 用户点击图标
    │
    ├── Launcher.startActivity() → AMS
    │   AMS: 暂停当前 Activity、准备 Task
    │
T1: AMS → Zygote (Socket)
    │   Zygote: fork 子进程
    │   耗时: ~100-200ms（不可优化）
    │
T2: 新进程启动
    │   ActivityThread.main()
    │   → Looper.prepareMainLooper()
    │   → ActivityThread.attach() → AMS.attachApplication()
    │
T3: Application 初始化
    │   → Application.attachBaseContext()
    │   → ContentProvider.onCreate() (所有 Provider)
    │   → Application.onCreate()
    │   耗时: 取决于 SDK 初始化（常见瓶颈）
    │
T4: Activity 创建
    │   → Activity.onCreate()
    │   → Activity.onStart()
    │   → Activity.onResume()
    │
T5: 首帧绘制
    │   → ViewRootImpl.performTraversals()
    │   → measure → layout → draw
    │   → SurfaceFlinger 合成上屏
    │   耗时: 取决于布局复杂度
    │
T6: 用户看到第一帧 (TTID)
    │
T7: 数据加载完成，完整内容展示 (TTFD)
```

**系统度量方式：**
```
// Logcat 自动输出
ActivityTaskManager: Displayed com.example/.MainActivity: +850ms

// 对应代码位置
ActivityRecord.reportFullyDrawn()  // TTFD
ActivityMetricsLogger             // 系统级统计
```

**各阶段耗时分布（典型值）：**
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

### 实战层

**度量工具：**
```kotlin
// 1. reportFullyDrawn() — 标记 TTFD
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // 数据加载完成后调用
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

- [[Android进程创建方式-fork-execve与Zygote]]
- [[Zygote进程启动与App进程孵化]]
- [[Activity启动流程]]
- [[启动优化-耗时分析与优化手段]]
- [[ContentProvider启动时序]]

## 记忆锚点

冷启动四大阶段：fork 进程（改不了）→ Application 初始化（重点优化）→ Activity 创建 → 首帧绘制。TTID 是用户感知的启动速度。
