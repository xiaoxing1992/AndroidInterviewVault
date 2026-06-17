---
tags: [android, 面试, framework, service, lifecycle, A]
difficulty: A
frequency: high
---

# Service 启动方式与原理

## 问题

> Android 中 Service 有几种启动方式？startService 和 bindService 的区别是什么？它们的底层原理是怎样的？前台 Service 和普通 Service 有什么区别？生命周期是如何管理的？

## 核心答案

Service 有三种启动方式：startService（独立运行，通过 stopService/stopSelf 停止）、bindService（绑定通信，所有客户端解绑后自动销毁）、startForegroundService（Android 8.0 后台限制下的前台服务，5 秒内必须显示通知）。底层都是通过 AMS（ActiveServices）调度，startService 走 scheduleCreateService + scheduleServiceArgs，bindService 额外走 onBind 返回 IBinder 建立双向通信通道。混用时，两种引用计数独立，必须都释放才会触发 onDestroy。

## 深入解析

### 原理层（源码级）

**三种启动方式总览：**

| 方式                         | 生命周期特点                      | 通信方式           | 典型场景           |
| -------------------------- | --------------------------- | -------------- | -------------- |
| `startService()`           | 独立运行，调用方停了它还在               | Intent 传参（单向）  | 上传文件、播放音乐      |
| `bindService()`            | 绑定者全解绑后自动销毁                 | IBinder 接口（双向） | 需要与 Service 交互 |
| `startForegroundService()` | 同 startService，但必须 5s 内显示通知 | Intent 传参      | 音乐播放、导航、录音     |

---

**startService 原理：**

```
App 调用 startService(intent)
        ↓
ContextImpl.startService()
        ↓
AMS.startService()                ← 跨进程 Binder 调用
        ↓
ActiveServices.startServiceLocked()
        ↓
  Service 是否存在？
  ├─ 不存在 → RealStartServiceLocked()
  │           ├─ ApplicationThread.scheduleCreateService()  → onCreate()
  │           └─ ApplicationThread.scheduleServiceAr4gs()    → onStartCommand()
  └─ 已存在 → scheduleServiceArgs()                         → onStartCommand()
```

`onStartCommand` 的返回值决定系统杀掉 Service 后的行为：

```java
// 系统杀掉后，自动重建，但 Intent 为 null
public static final int START_STICKY = 1;

// 系统杀掉后，自动重建，并重传最后的 Intent
public static final int START_REDELIVER_INTENT = 3;

// 系统杀掉后，不重建（默认）
public static final int START_NOT_STICKY = 2;
```

```kotlin
class UploadService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val filePath = intent?.getStringExtra("file") ?: return START_NOT_STICKY
        // 异步上传...
        return START_REDELIVER_INTENT  // 崩溃后系统会重传这个 Intent
    }

    override fun onBind(intent: Intent?) = null  // startService 模式不需要
}
```

---

**bindService 原理：**

```
App 调用 bindService(intent, conn, flags)
        ↓
ContextImpl.bindServiceCommon()
        ↓
  创建 ServiceConnection 的匿名内部类 IServiceConnection.Stub
  （这就是客户端的 Binder 代理通道）
        ↓
AMS.bindService()
        ↓
ActiveServices.bindServiceLocked()
        ↓
  Service 是否存在？
  ├─ 不存在 → RealStartServiceLocked() → onCreate()
  └─ 已存在
        ↓
  ApplicationThread.scheduleBindService()
        ↓
  Service.onBind() 返回 IBinder
        ↓
  AMS.publishService()
        ↓
  IServiceConnection.connected(IBinder)    ← 回调到客户端
        ↓
  ServiceConnection.onServiceConnected(ComponentName, IBinder)
```

核心：`onBind()` 返回的 IBinder 就是客户端与 Service 的通信桥梁。

```kotlin
// ---- Service 端 ----
class MusicService : Service() {
    private val binder = MusicBinder()

    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }

    override fun onBind(intent: Intent): IBinder = binder

    fun play() { /* 播放逻辑 */ }
    fun pause() { /* 暂停逻辑 */ }
}

// ---- Client 端 ----
class MusicActivity : AppCompatActivity() {
    private var musicService: MusicService? = null

    private val conn = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, binder: IBinder) {
            musicService = (binder as MusicService.MusicBinder).getService()
            // 现在可以直接调用 musicService.play()
        }

        override fun onServiceDisconnected(name: ComponentName) {
            musicService = null  // 意外断开（进程被杀）
        }
    }

    override fun onStart() {
        super.onStart()
        bindService(Intent(this, MusicService::class.java), conn, BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        unbindService(conn)
    }
}
```

---

**startForegroundService 原理：**

Android 8.0 后台限制：后台应用不能 startService，只能 startForegroundService。

```
startForegroundService()
        ↓
  与 startService 流程基本一致
        ↓
  但在 onCreate / onStartCommand 中 5 秒内必须调用：
  startForeground(notificationId, notification)
        ↓
  否则 → ANR → 系统杀掉应用
```

```kotlin
class MusicForegroundService : Service() {
    override fun onCreate() {
        super.onCreate()
        val notification = NotificationCompat.Builder(this, "music_channel")
            .setContentTitle("正在播放")
            .setSmallIcon(R.drawable.ic_music)
            .build()
        startForeground(1, notification)
    }
}
```

---

**Service 启动时的进程创建机制：**

当目标进程不存在时，`startService` / `bindService` 会触发进程创建。**发起方是 system_server 中的 ActiveServices**。

```
App 进程 (调用方)                system_server 进程               Zygote 进程
───────────────                ──────────────────               ────────────
startService(intent)
  → ContextImpl
    → AMS.startService() ──Binder──→ ActiveServices.startServiceLocked()
                                          │
                                     查找 ServiceRecord
                                          │
                                    目标进程存在吗？
                                     ┌────┴────┐
                                     ↓         ↓
                                   存在      不存在
                                     │         │
                            scheduleServiceArgs   startServiceInnerLocked()
                            (直接走生命周期)         → retrieveServiceLocked()
                                                  → bringUpServiceLocked()
                                                  → realStartServiceLocked()
                                                        │
                                                  判断进程不存在
                                                        │
                                                  Process.start()
                                                        │
                                               ──Socket──→ Zygote
                                                            │
                                                        fork()
                                                        ┌───┴───┐
                                                      父(Zygote)  子(App)
                                                      继续等待    handleChildProc()
                                                                    │
                                                              RuntimeInit.zygoteInit()
                                                                    │
                                                              ActivityThread.main()
                                                                    │
                                                              attachApplication()
                                                          ──Binder──→ AMS
                                                                      │
                                                              realStartServiceLocked()
                                                              scheduleCreateService → onCreate
                                                              scheduleServiceArgs → onStartCommand
```

核心源码路径（ActiveServices.bringUpServiceLocked）：

```java
private String bringUpServiceLocked(ServiceRecord r, ...) {
    // 1. 进程是否已存在？
    ProcessRecord app = mAm.getProcessRecordLocked(r.processName, r.appInfo.uid);

    if (app != null && app.thread != null) {
        // 进程存在，直接在这个进程里启动 Service
        realStartServiceLocked(r, app, ...);
        return null;
    }

    // 2. 进程不存在 → 请求 AMS 创建新进程
    app = mAm.startProcessLocked(r.processName, r.appInfo, ...);
    // ↓ 内部调用 Process.start()
    // ↓ 通过 socket 请求 Zygote fork
    // ↓ fork 成功后子进程回调 attachApplication
    // ↓ AMS 再次调用 realStartServiceLocked
}
```

关键节点：

| 问题 | 答案 |
|------|------|
| 谁发起进程创建？ | system_server 中的 ActiveServices |
| 什么时候触发？ | `bringUpServiceLocked()` 发现目标进程不存在时 |
| 通过什么创建？ | `Process.start()` → Socket → Zygote `fork()` |
| 为什么用 Socket 不用 Binder？ | Zygote fork 要求单线程环境，Binder 是多线程模型，不适合 |
| 子进程怎么回到 Service？ | `ActivityThread.main()` → `attachApplication()` → Binder 回调 AMS → AMS 调度 `realStartServiceLocked()` |
| 进程已存在时会怎样？ | 跳过 fork，AMS 直接通过 Binder（ApplicationThread）调度 Service 生命周期 |

**为什么 Zygote 用 Socket 不用 Binder？**

核心矛盾：**fork 要求单线程，Binder 是多线程模型。**

```
Binder 通信模型（多线程）：
  Binder 线程池
    ├── BinderThread-1  处理请求
    ├── BinderThread-2  处理请求
    └── BinderThread-3  处理请求

fork() 的限制：
  fork 只复制调用它的那个线程，其他线程直接消失
  → 如果其他线程正持有锁（如 malloc 内部锁）
  → 子进程继承了锁，但释放锁的线程没了
  → 子进程死锁
```

具体场景：

```
假设 Zygote 用 Binder 接收 fork 请求：
  1. Binder 线程池多线程运行中
  2. BinderThread-2 收到请求，调用 fork()
  3. 子进程里只有 BinderThread-2 存活
     → BinderThread-1/3 消失
     → 如果 BinderThread-1 正持有锁 → 子进程死锁

Socket 模型（单线程）：
  Zygote 主线程监听 socket
    → 收到请求 → fork()
    → 没有多线程竞争
    → 子进程不会继承"孤儿锁"
    → 安全
```

源码实现：

```java
// ZygoteServer.java — 主线程单线程监听
while (true) {
    ZygoteConnection connection = peers.get(pollIndex);  // 阻塞等待连接
    connection.processOneCommand(this);                   // 主线程处理 fork 请求
}
```

Binder 服务的注册时机：

```
fork 之前 → Socket（单线程安全，避免死锁）
fork 之后 → Binder（子进程启动后注册 Binder 服务，正常 IPC）

子进程启动路径：
  ActivityThread.main()
    → ActivityThread.attach()
      → AMS.attachApplication()    ← 这里开始用 Binder 与 AMS 通信
```

记忆口诀：**fork 前用 Socket（单线程安全），fork 后用 Binder（多线程高效）。**

---

**生命周期对比：**

```
startService:                    bindService:

  onCreate ←─── 首次               onCreate ←─── 首次
  onStartCommand                  onBind ←─── 首次
    ↓ (可多次调用)                  ↓ (只调一次)
  ... 运行中 ...                  ... 运行中 ...
    ↓                              ↓ (所有客户端都 unbind)
  stopService / stopSelf          onUnbind → onDestroy
  onDestroy

────── 混合使用 ──────

先 startService 再 bindService：
  onCreate → onStartCommand → onBind
  解绑后不会销毁（因为 startService 引用计数 > 0）
  必须 stopService 才销毁

先 bindService 再 startService：
  onCreate → onBind → onStartCommand
  同样，需要 stopService + 全部解绑才销毁
```

核心规则：`onDestroy` 必须满足两个条件同时成立——没有 startService 的活跃调用 + 没有任何绑定者。

### 实战层（项目经验）

**bindService flags 含义：**

| Flag | 含义 |
|------|------|
| `BIND_AUTO_CREATE` | Service 不存在时自动创建，最常用 |
| `BIND_NOT_FOREGROUND` | 不允许 Service 提升到前台进程优先级 |
| `BIND_IMPORTANT` | 标记为重要，Service 进程被杀优先级提高 |
| `BIND_WAIVE_PRIORITY` | 放弃调用方的优先级传递 |

**Android 8.0+ 后台 Service 限制：**

```
后台应用限制：
1. 不能 startService → 必须用 startForegroundService
2. 后台执行时间限制 1 分钟
3. Doze 模式下 CPU/网络受限

应对方案：
├─ 用户感知任务 → startForegroundService + 通知
├─ 可延迟任务 → WorkManager
└─ 周期任务 → WorkManager.PeriodicWorkRequest
```

**Service 线程模型：**

```
Service 运行在主线程！
├─ 耗时操作必须自己开线程
├─ 推荐用协程：lifecycleScope / 自建 CoroutineScope
├─ 或用 HandlerThread 处理串行任务
└─ IntentService（已废弃）本质就是 HandlerThread + Service
```

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| startService 和 bindService 能混用吗？ | 能。常见模式：bindService 拿交互接口 + startService 保活。两个都释放才销毁 |
| bindService 时 Service 还没创建完怎么办？ | `onServiceConnected` 会在 Service 创建好后回调，不会丢失 |
| Service 在哪个线程运行？ | 主线程。耗时操作必须自己开线程或用协程 |
| Service 里能弹 Toast 吗？ | 能，因为 Service 在主线程，有 Looper |
| unbindService 忘记调用会怎样？ | ServiceConnection 泄漏，Service 不会销毁 |
| onServiceDisconnected 什么时候调用？ | 进程被杀或 crash 时调用，正常 unbindService 不会触发 |

### 延伸问题

- [[Handler-Looper-MessageQueue机制]] — Service 内部线程通信基础
- [[Binder通信原理]] — startService/bindService 底层依赖 Binder IPC
- [[IntentService为什么被废弃]] — HandlerThread + Service 的封装与局限
- [[App冷启动全链路]] — Service 启动与进程创建的关联
- [[协程作用域与结构化并发]] — Service 中使用协程管理异步任务

## 记忆锚点

startService 是"打电话叫外卖"——点了就不管了，外卖自己送（独立运行）；bindService 是"去餐厅面对面点菜"——走的时候结账（解绑），服务员才下班；startForegroundService 是"带喇叭的外卖"——不仅要送，还得让所有人知道你在点外卖（显示通知）。混用时就像同时电话点餐又在餐厅坐着——必须挂电话+离开餐厅，服务才算结束。
