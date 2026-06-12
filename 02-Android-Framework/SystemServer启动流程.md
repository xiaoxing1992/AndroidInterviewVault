---
tags: [android, 面试, framework, system_server, zygote, S]
difficulty: S
frequency: high
---

# SystemServer 启动流程

## 问题

> SystemServer 是怎么启动的？它和 Zygote 是什么关系？它内部启动了哪些系统服务？

## 核心答案

SystemServer 由 Zygote 在启动阶段通过 `forkSystemServer()` fork 出来，是 Zygote 的第一个子进程。fork 后子进程进入 `handleSystemServerProcess()`，通过反射调用 `SystemServer.main()`，最终执行 `SystemServer.run()` 方法，按顺序启动三类系统服务：引导服务（AMS、PMS、PKMS 等）、核心服务（BatteryService 等）、其他服务（WMS、InputManagerService 等）。SystemServer 启动完成后，Zygote 才进入 `runSelectLoop()` 等待 AMS 的进程创建请求。

## 深入解析

### 原理层

**SystemServer 在启动链路中的位置：**

```text
Linux kernel
  → init 进程
    → 启动 /system/bin/app_process --zygote --start-system-server
      → ZygoteInit.main()
        → registerZygoteSocket()
        → preload()                        // 预加载（SystemServer 也共享这些）
        → forkSystemServer()               // ★ fork SystemServer
          ├─ 子进程：SystemServer 进程
          └─ 父进程：Zygote，继续 runSelectLoop()
```

**forkSystemServer() 的关键参数：**

```java
// ZygoteInit.java
private static boolean forkSystemServer(String abiList, String socketName,
        ZygoteServer zygoteServer) {

    // 准备 SystemServer 的参数
    String args[] = {
        "--setuid=1000",              // uid = 1000 (SYSTEM)
        "--setgid=1000",              // gid = 1000 (SYSTEM)
        "--setgroups=1001,1002,...",   // 附加组（net_bt 等）
        "--capabilities=" + capabilities,
        "--nice-name=system_server",  // 进程名
        "com.android.server.SystemServer"  // 入口类
    };

    // fork 子进程
    int pid = Zygote.forkSystemServer(
        parsedArgs.mUid, parsedArgs.mGid,
        parsedArgs.mGids, parsedArgs.mRuntimeFlags, ...);

    if (pid == 0) {
        // 子进程：处理 SystemServer 的初始化
        handleSystemServerProcess(parsedArgs);
    }
    return true;
}
```

关键点：
- uid/gid 都是 1000（SYSTEM 用户），这是系统级权限
- 入口类是 `com.android.server.SystemServer`
- fork 和普通 App 的 `forkAndSpecialize` 类似，但走的是 `forkSystemServer` 专用路径

**子进程 fork 后的链路：**

```text
handleSystemServerProcess(parsedArgs)
  → 关闭从 Zygote 继承的 socket（SystemServer 不需要）
  → ZygoteInit.zygoteInit()
    → RuntimeInit.commonInit()         // 设置默认 UncaughtExceptionHandler
    → ZygoteInit.nativeZygoteInit()    // 启动 Binder 线程池
    → RuntimeInit.applicationInit()
      → invokeStaticMain()             // 反射调用入口类的 main()
        → Class.forName("com.android.server.SystemServer")
        → SystemServer.main(args)
```

`invokeStaticMain()` 的实现原理：
- 通过反射拿到 `SystemServer.main()` 的 Method 对象
- 包装成 `Zygote.MethodAndArgsCaller`（Runnable）
- 抛出给 `ZygoteInit.main()` 捕获并执行
- 这个 trick 的目的是清理栈帧，让 SystemServer 的 main 看起来像真正的入口函数

**SystemServer.main() → run() 的核心流程：**

```java
// SystemServer.java
private void run() {
    // 1. 设置系统属性和时区
    SystemProperties.set("persist.sys.dalvik.vm.lib.2",
                         VMRuntime.getRuntime().vmLibrary());

    // 2. 创建主线程 Looper
    Looper.prepareMainLooper();

    // 3. 加载 android_servers 系统库
    System.loadLibrary("android_servers");

    // 4. ★ 启动引导服务（Bootstrap Services）
    startBootstrapServices(t);
    //    → ActivityManagerService
    //    → PowerManagerService
    //    → PackageManagerService (PKMS)
    //    → RecoverySystemService
    //    → ...

    // 5. ★ 启动核心服务（Core Services）
    startCoreServices(t);
    //    → BatteryService
    //    → UsageStatsService
    //    → WebViewUpdateService
    //    → ...

    // 6. ★ 启动其他服务（Other Services）
    startOtherServices(t);
    //    → WindowManagerService
    //    → InputManagerService
    //    → NetworkManagementService
    //    → ConnectivityService
    //    → AlarmManagerService
    //    → ...

    // 7. 服务启动完毕，开启 Looper 消息循环
    Looper.loop();
    // 永远不会返回（如果返回说明系统出错）
}
```

**三类服务的启动顺序和原因：**

| 阶段 | 方法 | 典型服务 | 为什么排在这个顺序 |
|------|------|----------|-------------------|
| 引导服务 | `startBootstrapServices()` | AMS、PMS、PKMS | 其他服务依赖这些基础服务，必须先启动 |
| 核心服务 | `startCoreServices()` | BatteryService、UsageStats | 非必须但重要的系统功能 |
| 其他服务 | `startOtherServices()` | WMS、InputManager | 依赖引导服务已就绪 |

启动顺序有严格的依赖关系：
- AMS 必须在 WMS 之前（WMS 依赖 AMS 管理 Activity 栈）
- PMS 必须在其他服务之前（其他服务需要查询包信息）
- PowerManagerService 必须在 WMS 之前（WMS 依赖电源状态）

**SystemServer 和 Zygote 的关系：**

```text
Zygote
  ├── fork → SystemServer（第一个子进程，启动时创建）
  ├── fork → App 进程 A（AMS 按需请求创建）
  ├── fork → App 进程 B
  └── ...
```

- SystemServer 是 Zygote fork 出来的，共享预加载的 Framework 类（COW）
- SystemServer 启动完所有服务后，才标志着系统"基本就绪"
- AMS 在 SystemServer 中运行，后续 App 进程由 AMS 通过 socket 请求 Zygote fork
- SystemServer 崩溃会触发 `watchdog`，导致整个系统重启（zygote + 所有 App 一起重启）

**SystemServer 崩溃后的重启流程：**

```text
SystemServer 崩溃
  → Zygote 的 SIGCHLD 处理
  → init 检测到 zygote 子进程退出
  → init 重启 zygote（rc 配置的 onrestart 行为）
  → Zygote 重启
    → 重新 preload
    → 重新 forkSystemServer
    → SystemServer 重新启动所有服务
    → 所有 App 进程需要重新创建
```

### 实战层

**面试常见追问：**

- SystemServer 和普通 App 进程有什么区别？
- SystemServer 为什么是 Zygote fork 而不是 init 启动的？
- AMS 和 SystemServer 是什么关系？
- `startBootstrapServices()` 里的启动顺序是固定的吗？为什么？
- SystemServer 崩溃后系统会怎样？
- `invokeStaticMain()` 为什么用反射而不是直接调用？
- SystemServer 的 uid/gid 是多少？为什么是 1000？

**排查和观察：**

```bash
# 查看 SystemServer 进程信息
adb shell ps -A | grep system_server

# 查看 SystemServer 线程数（通常 200+）
adb shell ps -T -p $(adb shell pidof system_server) | wc -l

# 查看 SystemServer 日志
adb logcat -s SystemServer

# 查看系统服务启动耗时
adb logcat | grep "SystemServer.*took"

# 查看 SystemServer 的 SELinux context
adb shell cat /proc/$(adb shell pidof system_server)/attr/current
```

**源码入口：**

- `ZygoteInit.forkSystemServer()`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
- `SystemServer.java`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/java/com/android/server/SystemServer.java
- `RuntimeInit.applicationInit()`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/RuntimeInit.java

推荐阅读顺序：
1. `ZygoteInit.java` `forkSystemServer()` —— 看 fork 参数和子进程处理
2. `SystemServer.java` `main()` → `run()` —— 看三类服务的启动流程
3. `SystemServer.java` `startBootstrapServices()` —— 看 AMS/PMS 的创建顺序
4. `ActivityManagerService.java` 构造函数 —— 看 AMS 如何初始化

**容易答错的点：**

- SystemServer 不是 init 直接启动的，它由 Zygote fork 出来
- SystemServer 是 Zygote 的**第一个**子进程，在 `runSelectLoop()` 之前创建
- SystemServer 不是系统服务本身，它是承载所有系统服务的**进程**
- AMS 是 SystemServer 内部启动的一个服务，不是独立进程
- SystemServer 崩溃会导致整个 Android 系统重启，不只是服务重启

### 延伸问题

- [[系统服务依赖解决与发布机制与线程模型]]
- [[系统服务启动流程]]
- [[Zygote进程启动与App进程孵化]]
- [[ZygoteConnection-runOnce详解]]
- [[ServiceManager与system_server区别]]
- [[Activity启动流程]]
- [[App冷启动全链路]]
- [[为什么App由Zygote-fork而非system_server]]

## 记忆锚点

SystemServer 是 Zygote fork 的**第一个子进程**，uid=1000，入口 `SystemServer.main()` 里按"引导→核心→其他"三阶段启动所有系统服务。它是 Zygote 和 App 进程之间的桥梁——没有 SystemServer 里的 AMS，就没有后续的 App 进程创建请求。
