---
tags: [android, 面试, framework, activity, process, zygote, binder, S]
difficulty: S
frequency: high
---

# Activity 所在进程的启动过程

## 问题

> Activity 所在的进程是怎么启动的？AMS 发现进程不存在后做了什么？子进程 fork 出来后第一件事做什么？进程已存在时还 fork 吗？

## 核心答案

Activity 进程启动分两种场景：进程不存在时，AMS 通过 ProcessList → ZygoteProcess → Socket 请求 Zygote fork 新进程，子进程进入 ActivityThread.main() → attach() 向 AMS 报到 → AMS 通过 bindApplication 下发初始化数据 → scheduleTransaction 调度 Activity 生命周期。进程已存在时，跳过整个 fork 链路，AMS 直接通过 Binder（IApplicationThread.scheduleTransaction）调度 Activity 创建。关键区别：fork 前用 Socket（单线程安全），fork 后用 Binder（多线程高效）。

## 深入解析

### 原理层（源码级）

#### 场景 A：进程不存在 — 完整 fork 链路

**Step 1：AMS 发现进程不存在**

```java
// ActivityStarter.java
int startActivity(...) {
    // 解析 Intent、权限检查、创建 ActivityRecord
    return startActivityUnchecked(...);
}

// ActivityStackSupervisor.realStartActivityLocked()
boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app, ...) {
    if (app == null || app.thread == null) {
        // 进程不存在，需要创建
    }
}

// ProcessList.java
ProcessRecord startProcessLocked(String processName, ApplicationInfo info, ...) {
    ProcessRecord app = getProcessRecordLocked(processName, info.uid);

    if (app == null) {
        // 不存在，创建新的 ProcessRecord
        app = newProcessRecordLocked(info, processName, isolated, ...);
    }

    // 启动进程
    startProcessLocked(app, hostingType, ...);
}

private Process.ProcessStartResult startProcessLocked(ProcessRecord app, ...) {
    // 通过 ZygoteProcess 创建进程
    Process.ProcessStartResult startResult = Process.start(...);
    // ↓ 内部调用 ZygoteProcess.start()
}
```

---

**Step 2：ZygoteProcess → Zygote（Socket）**

```java
// ZygoteProcess.java
public Process.ProcessStartResult start(...) {
    return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), ...);
}

private ZygoteState openZygoteSocketIfNeeded(String abi) {
    // 根据 ABI 选择 zygote32 或 zygote64 的 socket
    // 建立 LocalSocket 连接
}

private Process.ProcessStartResult zygoteSendArgsAndGetResult(
        ZygoteState zygoteState, List<String> args) {
    // 把参数写入 socket
    // 参数内容：
    //   --runtime-flags 0
    //   --uid 10123
    //   --gid 10123
    //   --nice-name com.example.app
    //   android.app.ActivityThread        ← 入口类
    //   com.example.app.MainActivity      ← 入口 Activity

    final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
    zygoteWriter.write(Integer.toString(args.size()));
    for (String arg : args) {
        zygoteWriter.newLine();
        zygoteWriter.write(arg);
    }
    zygoteWriter.flush();

    // 读取结果（pid 或错误码）
    Process.ProcessStartResult result = new Process.ProcessStartResult();
    result.pid = zygoteState.mZygoteInputStream.readInt();
    result.usingWrapper = zygoteState.mZygoteInputStream.readBoolean();
    return result;
}
```

---

**Step 3：Zygote 接收请求并 fork**

```java
// ZygoteServer.java
void runSelectLoop(String abiList) {
    while (true) {
        // select 等待 socket 连接
        ZygoteConnection connection = peers.get(pollIndex);
        final Runnable command = connection.processOneCommand(this);
        // command 在子进程中执行
    }
}

// ZygoteConnection.java
Runnable processOneCommand(ZygoteServer zygoteServer) {
    Zygote.Arguments args = new Zygote.Arguments(mArgs);

    // fork 子进程
    pid = Zygote.forkAndSpecialize(
        parsedArgs.mUid, parsedArgs.mGid,
        parsedArgs.mGids, parsedArgs.mRuntimeFlags,
        rlimits, parsedArgs.mMountExternal,
        parsedArgs.mSeInfo, parsedArgs.mNiceName,
        fdsToClose, fdsToIgnore, parsedArgs.mStartChildZygote,
        parsedArgs.mInstructionSet, parsedArgs.mAppDataDir,
        parsedArgs.mInvokeWith, parsedArgs.mZygoteSessionId);

    if (pid == 0) {
        // 子进程
        zygoteServer.setForkChild();
        return handleChildProc(args, ...);
    } else {
        // 父进程（Zygote），继续等待下一个请求
        return handleParentProc(pid, ...);
    }
}
```

---

**Step 4：子进程进入 ActivityThread.main()**

```java
// ZygoteConnection.java
private Runnable handleChildProc(Zygote.Arguments args, ...) {
    // 关闭从 Zygote 继承的 socket
    // 设置进程名、uid、gid、capabilities
    return ZygoteInit.zygoteInit(...);
}

// ZygoteInit.java
public static Runnable zygoteInit(int targetSdkVersion, ...) {
    RuntimeInit.commonInit();           // 设置默认 UncaughtExceptionHandler
    ZygoteInit.nativeZygoteInit();      // 启动 Binder 线程池（关键！）
    return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
}

// RuntimeInit.java
protected static Runnable applicationInit(int targetSdkVersion, ...) {
    // 反射调用 ActivityThread.main()
    return findStaticMain(args.startClass, args.startArgs, classLoader);
}
```

子进程启动顺序：

```
fork()
  → 关闭继承的 Socket
  → 设置 uid/gid/capabilities/进程名
  → RuntimeInit.commonInit()           ← 设置默认异常处理
  → nativeZygoteInit()                 ← 启动 Binder 线程池
  → ActivityThread.main()              ← 进入主线程
  → Looper.prepareMainLooper()         ← 准备主线程 Looper
  → ActivityThread.attach()            ← 向 AMS 报到
  → Looper.loop()                      ← 消息循环，永不返回
```

---

**Step 5：attachApplication — 子进程向 AMS 报到**

```java
// ActivityThread.java
private void attach(boolean system, long startSeq) {
    if (!system) {
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        final IActivityManager mgr = ActivityManager.getService();
        // 向 AMS 报到，传入 ApplicationThread（Binder 接口）
        mgr.attachApplication(mAppThread, startSeq);
        // ↑ Binder 跨进程调用到 system_server
    }
}

// AMS.java (system_server 进程)
private boolean attachApplicationLocked(IApplicationThread thread, int pid, ...) {
    // 1. 找到对应的 ProcessRecord
    ProcessRecord app = mPidsSelfLocked.get(pid);

    // 2. 绑定 ApplicationThread（建立双向 Binder 通道）
    app.makeActive(thread, mProcessStats);

    // 3. 发送 bindApplication 数据
    thread.bindApplication(processName, appInfo, ...);
    // → App 进程收到 → handleBindApplication()

    // 4. 检查待启动的组件
    if (mStackSupervisor.attachApplicationLocked(app)) {
        // → realStartActivityLocked() → 启动 Activity
    }
    // 同时检查 Service、BroadcastReceiver
}
```

---

**Step 6：AMS 调度 Activity 生命周期**

```java
// ActivityStackSupervisor.java
boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app, ...) {
    // 创建 ClientTransaction
    final ClientTransaction clientTransaction = ClientTransaction.obtain(
            app.getThread(), r.token);

    // 添加 LaunchActivityItem（创建 Activity）
    clientTransaction.addCallback(LaunchActivityItem.obtain(...));

    // 设置目标生命周期状态为 RESUMED
    clientTransaction.setLifecycleStateRequest(
            ResumeActivityItem.obtain(...));

    // 通过 Binder 发送到 App 进程
    mService.getLifecycleManager().scheduleTransaction(clientTransaction);
    // → IApplicationThread.scheduleTransaction()
}
```

```java
// ActivityThread.java (App 进程收到)
class ApplicationThread extends IApplicationThread.Stub {
    @Override
    public void scheduleTransaction(ClientTransaction transaction) {
        // 通过 Handler 投递到主线程
        ActivityThread.this.scheduleTransaction(transaction);
        // → TransactionExecutor.execute()
        //   → LaunchActivityItem.execute()
        //     → handleLaunchActivity()
        //       → performLaunchActivity()
        //         → activity.attach()       ← mBase = ContextImpl
        //         → activity.onCreate()     ← setContentView
        //   → ResumeActivityItem.execute()
        //     → handleResumeActivity()
        //       → activity.onStart()
        //       → activity.onResume()
    }
}
```

---

#### 场景 B：进程已存在 — 直接调度

```
App 进程已在运行中
    ↓
startActivity(B)
    ↓
AMS.resolveActivity()
    ↓
找到目标进程 ProcessRecord（已存在，app.thread != null）
    ↓
realStartActivityLocked(r, app, ...)
    ↓
不需要 fork，直接：
    → ClientTransaction
    → LaunchActivityItem + ResumeActivityItem
    → IApplicationThread.scheduleTransaction()  ← Binder
    ↓
App 进程主线程收到
    → B.attach()
    → B.onCreate()
    → B.onStart()
    → B.onResume()
```

与场景 A 的区别：跳过了 fork、ActivityThread.main()、attachApplication、handleBindApplication 整个流程。

---

### 完整流程图（场景 A）

```
Launcher 进程        system_server         Zygote              新 App 进程
─────────────       ──────────────        ───────              ──────────
startActivity()
  → AMS ──Binder──→ ATMS
                    ActivityStarter
                    解析/权限/Task
                      ↓
                    进程不存在
                      ↓
                    ProcessList
                    → ZygoteProcess
                      → socket ──→ ZygoteServer
                                    runSelectLoop
                                    processOneCommand
                                    forkAndSpecialize()
                                      fork()
                                    ┌────────┴────────┐
                                  Zygote 等待          子进程
                                                    handleChildProc
                                                    zygoteInit
                                                    nativeZygoteInit
                                                      (Binder 线程池)
                                                    ActivityThread.main()
                                                      ↓
                                                    attach()
                                                    → AMS ──Binder──→ attachApplicationLocked
                                                                       ↓
                                                  ←──Binder── bindApplication()
                                                    handleBindApplication
                                                      attachBaseContext
                                                      ContentProvider.onCreate
                                                      Application.onCreate
                                                                       ↓
                                                  ←──Binder── scheduleTransaction
                                                              (LaunchActivityItem)
                                                    handleLaunchActivity
                                                      activity.attach()
                                                      activity.onCreate()
                                                    handleResumeActivity
                                                      activity.onStart()
                                                      activity.onResume()
                                                    ↓
                                                    ViewRootImpl
                                                    measure → layout → draw
                                                    ↓
                                                  SurfaceFlinger → 屏幕
```

### 通信方式汇总

| 阶段 | 发起方 | 接收方 | 通信方式 | 说明 |
|------|--------|--------|----------|------|
| startActivity | Launcher | AMS | Binder | IActivityTaskManager |
| pauseActivity | AMS | Launcher | Binder | IApplicationThread |
| 创建进程请求 | AMS | Zygote | Socket | 单线程安全，避免 fork 死锁 |
| fork | Zygote | - | fork() | COW 共享内存 |
| attachApplication | App | AMS | Binder | 子进程启动 Binder 线程池后发起 |
| bindApplication | AMS | App | Binder | 下发 Application 初始化数据 |
| publishContentProviders | App | AMS | Binder | 发布 ContentProvider |
| scheduleTransaction | AMS | App | Binder | 下发 Activity 生命周期事务 |

### 实战层（项目经验）

**面试高频追问：**

| 问题 | 答案 |
|------|------|
| Activity 的进程是 AMS 直接创建的吗？ | 不是，AMS 通过 Socket 请求 Zygote fork |
| fork 后子进程第一件事做什么？ | 关闭继承的 Socket → 设置 uid/gid → 启动 Binder 线程池 → ActivityThread.main() |
| 子进程怎么知道要启动哪个 Activity？ | AMS 通过 bindApplication 传入 AppBindData，然后通过 scheduleTransaction 传入 LaunchActivityItem |
| ActivityThread.main() 为什么不会退出？ | Looper.loop() 进入消息循环，永不返回 |
| 进程已存在时会重复 fork 吗？ | 不会，AMS 先查 ProcessRecord，存在则直接 Binder 调度 |
| attachApplication 用什么通信？ | Binder（此时 Binder 线程池已通过 nativeZygoteInit 启动） |
| 为什么 fork 前用 Socket，fork 后用 Binder？ | fork 要求单线程（Binder 多线程会导致死锁），fork 后 Binder 线程池已启动，用 Binder 更高效 |
| 子进程的 Binder 线程池什么时候启动？ | nativeZygoteInit() 中，在 ActivityThread.main() 之前 |

**排查和观察方式：**

```bash
# 查看进程关系（Zygote 是所有 App 进程的父进程）
adb shell ps -A | grep -E 'zygote|system_server|com.example'

# 查看进程启动参数
adb shell cat /proc/<pid>/cmdline

# 查看 Activity 启动耗时
adb logcat -s ActivityTaskManager
# 输出: Displayed com.example/.MainActivity: +850ms

# Systrace/Perfetto 分析 fork 和启动耗时
# 抓 trace 时关注 Zygote、ActivityThread、bindApplication 相关标签
```

### 延伸问题

- [[Zygote进程启动与App进程孵化]] — Zygote 预加载与 fork 机制
- [[Android进程创建方式-fork-execve与Zygote]] — 为什么 fork 后不 execve
- [[Zygote为什么用Socket不用Binder]] — Socket vs Binder 的深层原因
- [[Activity启动流程]] — AMS 调度 Activity 生命周期的完整源码
- [[Application原理与实践]] — handleBindApplication 中的初始化顺序
- [[ContentProvider启动时序]] — 为什么 Provider 先于 Application.onCreate
- [[App冷启动全链路]] — 完整的时间线与耗时分析
- [[Binder通信原理]] — attachApplication 背后的 Binder 机制

## 记忆锚点

Activity 进程启动三步走：AMS 发现进程不在 → Socket 请 Zygote fork → 子进程 ActivityThread.main() 向 AMS 报到（Binder）→ AMS 下发 Activity 生命周期（Binder）。进程在就直接调度，不在才 fork。Socket 用于 fork 前（安全），Binder 用于 fork 后（高效）。
