---
tags: [android, 面试, framework, zygote, S]
difficulty: S
frequency: high
---

# Zygote 进程启动与 App 进程孵化

## 问题

> Zygote 是什么？Android 系统启动时 Zygote 是怎么启动的？普通 App 进程又是如何从 Zygote fork 出来的？

## 核心答案

Zygote 是 Android 应用进程的孵化器，由 `init` 进程根据 `init.zygote*.rc` 启动，本质上运行的是 `/system/bin/app_process`。它启动后进入 `ZygoteInit.main()`，注册 zygote socket、预加载 Framework 类和资源、fork 出 `system_server`，然后进入循环等待 AMS 的进程创建请求。AMS 需要启动新 App 进程时，通过 socket 把进程参数发给 Zygote，Zygote fork 子进程，子进程最终进入 `ActivityThread.main()`，成为目标 App 进程。

## 深入解析

### 原理层

**Zygote 的定位：**

- Zygote 是所有普通 Android App 进程的父进程
- 它在系统启动早期创建，先加载大量 Framework 类、资源和常用 native 库
- 后续 App 进程通过 `fork()` 复制 Zygote 的进程空间
- 依靠 COW（Copy-On-Write），预加载内容的内存页可以在多个 App 之间共享

**系统启动到 Zygote 的链路：**

```text
Linux kernel
  → init 进程
    → 解析 init.rc / init.zygote64.rc / init.zygote64_32.rc
      → 启动 /system/bin/app_process
        → AndroidRuntime.start()
          → ZygoteInit.main()
            → registerZygoteSocket()
            → preload()
            → forkSystemServer()
            → runSelectLoop()
```

**init 配置示例（简化）：**

```text
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
```

关键点：
- `service zygote` 定义了 zygote 服务
- `app_process64` 是实际可执行文件
- `--zygote` 表示进入 Zygote 模式
- `--start-system-server` 表示启动过程中要 fork `system_server`
- `socket zygote` 让 `init` 创建 Unix domain socket，并把 fd 传给 Zygote 使用

**ZygoteInit.main() 做的事：**

```java
public static void main(String[] argv) {
    ZygoteServer zygoteServer = new ZygoteServer();

    // 1. 注册 zygote socket，等待 system_server 中的 AMS/ProcessList 后续连接
    zygoteServer.registerServerSocketFromEnv(socketName);

    // 2. 预加载 Framework 类、资源、OpenGL、文字资源等
    preload(bootTimingsTraceLog);

    // 3. fork system_server
    if (startSystemServer) {
        forkSystemServer(abiList, socketName, zygoteServer);
    }

    // 4. 进入循环，等待 AMS 请求创建 App 进程
    zygoteServer.runSelectLoop(abiList);
}
```

**App 进程创建链路：**

```text
system_server 进程                         Zygote 进程                      App 子进程
────────────────                         ───────────                      ─────────
AMS/ATMS 发现目标进程不存在
  → Process.start()
    → ZygoteProcess.start()
      → openZygoteSocketIfNeeded()
      → zygoteSendArgsAndGetResult()
         通过 socket 发送参数 ───────────→ runSelectLoop()
                                           → processCommand()
                                           → ZygoteConnection.runOnce()  // [[ZygoteConnection-runOnce详解]]
                                           → Zygote.forkAndSpecialize()
                                                                        子进程返回 pid = 0
                                                                        → handleChildProc()
                                                                        → RuntimeInit.zygoteInit()
                                                                        → ActivityThread.main()
                                           父进程返回子进程 pid
      ← 得到新进程 pid
  → 记录 ProcessRecord
```

**fork 后父子进程分工：**

| 进程 | fork 返回值 | 后续动作 |
|------|-------------|----------|
| Zygote 父进程 | 子进程 pid | 继续留在 `runSelectLoop()`，等待下一个创建请求 |
| App 子进程 | 0 | 关闭无关 socket，设置进程名/uid/gid/capabilities，进入 `ActivityThread.main()` |

**为什么 Zygote 通信用 socket，而不是 Binder？**

核心原因是 fork 安全性。Zygote 必须尽量保持单线程、状态简单，避免 fork 多线程进程后出现锁状态不一致。Binder 驱动和 Binder 线程池会引入多线程模型，不适合承担 Zygote 的进程孵化通信。Unix domain socket 更简单，便于 Zygote 在 select loop 中串行处理创建请求。

**为什么 App 进程从 Zygote fork 会更快？**

- 不需要每个 App 都从零加载 Java 虚拟机和 Framework 基础类
- 预加载类和资源通过 COW 共享，只有写入时才复制内存页
- App 进程 fork 后只需要做差异化初始化：uid/gid、进程名、运行时参数、入口类等

**system_server 和普通 App 的区别：**

- `system_server` 也是由 Zygote fork 出来的，但它在系统启动阶段由 `forkSystemServer()` 创建
- `system_server` 会启动 AMS、PMS、WMS 等核心系统服务
- 普通 App 进程由 AMS 后续按需请求 Zygote 创建

**zygote64、zygote32、zygote64_32 的区别：**

- `zygote64`：只支持 64 位 App 进程
- `zygote32`：只支持 32 位 App 进程
- `zygote64_32`：主 Zygote 是 64 位，同时启动 secondary zygote 支持 32 位
- AMS 会根据目标 App 的 ABI 选择合适的 Zygote socket

### 实战层

**面试常见追问：**

- `init`、Zygote、system_server、AMS 的关系是什么？
- 为什么 App 进程不是由 AMS 直接 fork？
- 为什么 Zygote fork 后能共享内存？
- 为什么 fork 之前要预加载类和资源？
- 为什么 Zygote 不能随便开线程？

**排查和观察方式：**

```bash
# 查看 zygote/system_server/app 进程关系
adb shell ps -A | grep -E 'zygote|system_server|com.example'

# 查看 zygote 启动配置
adb shell ls /system/etc/init | grep zygote
adb shell cat /system/etc/init/zygote64.rc
```

**源码入口：**

官方 Android Code Search：
- `init.rc`：https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.rc
- `init.zygote64.rc`：https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.zygote64.rc
- `init.zygote64_32.rc`：https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.zygote64_32.rc
- `app_process` 入口：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/cmds/app_process/app_main.cpp
- `ZygoteInit.java`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
- `ZygoteServer.java`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
- `ZygoteProcess.java`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/ZygoteProcess.java

Git 镜像入口：
- https://android.googlesource.com/platform/system/core/+/main/rootdir/
- https://android.googlesource.com/platform/frameworks/base/+/main/cmds/app_process/
- https://android.googlesource.com/platform/frameworks/base/+/main/core/java/com/android/internal/os/

真机上的 rc 文件会叠加设备和厂商配置，不一定只等于 AOSP 通用版本：

```bash
adb shell cat /system/etc/init/hw/init.rc
adb shell cat /system/etc/init/zygote64.rc
adb shell ls /vendor/etc/init
```

推荐阅读顺序：
1. `init.zygote64.rc`：看 `service zygote` 如何定义 `app_process64`、socket 和启动参数
2. `app_main.cpp`：看 `--zygote` 参数如何进入 `AndroidRuntime.start()`
3. `AndroidRuntime.start()`：看 native 层如何反射进入 `ZygoteInit.main()`
4. `ZygoteInit.java`：看 socket 注册、预加载、fork `system_server`
5. `ZygoteServer.java`：看 `runSelectLoop()` 如何接收 fork 请求
6. `ZygoteProcess.java`：看 system_server 侧如何连接 Zygote 并发送创建进程参数

**容易答错的点：**

- Zygote 不是系统服务，它是由 `init` 拉起的 native 进程
- AMS 不直接 fork App，AMS 通过 zygote socket 请求 Zygote fork
- App 子进程不是从 `main()` 从零启动，而是 fork 后进入 `ActivityThread.main()`
- Zygote 预加载提升的不只是速度，还有跨进程共享内存带来的内存收益

### 延伸问题

- [[SystemServer启动流程]]
- [[Zygote为什么用Socket不用Binder]]
- [[为什么App由Zygote-fork而非system_server]]
- [[ZygoteConnection-runOnce详解]]
- [[Android进程创建方式-fork-execve与Zygote]]
- [[ServiceManager与system_server区别]]
- [[Init进程启动与rc解析]]
- [[Activity启动流程]]
- [[App冷启动全链路]]
- [[Binder通信原理]]
- [[启动优化-耗时分析与优化手段]]

## 记忆锚点

Zygote 四步记：`init` 拉起 `app_process`，Zygote 预加载，先 fork `system_server`，再等 AMS 通过 socket 请求 fork App。
