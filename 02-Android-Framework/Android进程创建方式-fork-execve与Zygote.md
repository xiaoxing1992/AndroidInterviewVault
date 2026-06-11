---
tags: [android, 面试, framework, process, fork, execve, zygote, S]
difficulty: S
frequency: high
---

# Android 进程创建方式 - fork/execve 与 Zygote

## 问题

> Android 启动进程有几种方式？`fork + execve` 和 Zygote 的 `fork + handleChildProc` 有什么区别？为什么 App 进程不是 fork 后再 execve？

## 核心答案

Android 常见进程创建可以分三类：第一类是 `init` 启动 native daemon，走 `fork + execve`，子进程 fork 出来后用 `execve()` 替换成目标可执行文件；第二类是 Zygote 创建 `system_server` 和普通 App 进程，走 `fork + handle`，子进程不 `execve`，而是在 Zygote 预加载好的运行时里继续执行 `handleChildProc()` / `zygoteInit()`，最后进入 `SystemServer.main()` 或 `ActivityThread.main()`；第三类是目标进程已存在时不创建新进程，AMS 直接通过 Binder 通知现有进程启动组件。

## 深入解析

### 原理层

**三种常见场景：**

| 场景 | 创建方式 | 典型发起方 | 子进程后续 |
|------|----------|------------|------------|
| native 系统服务 | `fork + execve` | `init` | 替换成目标 ELF 程序，如 `servicemanager`、`surfaceflinger` |
| App / `system_server` | `fork + handle` | Zygote | 不替换进程镜像，继续执行 Java/ART 初始化 |
| 进程已存在 | 不 fork | AMS/ATMS | Binder 通知现有进程执行组件生命周期 |

**方式一：fork + execve**

这是传统 Linux 创建新程序的方式。

```text
父进程
  → fork()
    ├─ 父进程：得到子进程 pid，继续管理
    └─ 子进程：execve("/system/bin/servicemanager", argv, envp)
         → 当前进程镜像被替换
         → 从 servicemanager 的入口函数重新开始执行
```

`execve()` 的关键效果：
- 不会创建新 pid，而是在当前子进程里替换代码段、数据段、堆、栈等用户态进程镜像
- 原来 fork 继承来的大部分用户态内存会被新程序替换
- 保留 pid、部分文件描述符、进程属性等内核态身份
- 适合启动完全独立的 native 可执行文件

在 Android 里，`init` 解析 rc 里的 service 后，典型就是这种模型：

```text
service servicemanager /system/bin/servicemanager
service surfaceflinger /system/bin/surfaceflinger

init
  → fork()
    → child execve("/system/bin/servicemanager")
```

**方式二：Zygote fork + handle**

Zygote 创建 App 不是传统的 `fork + execve`，而是：

```text
system_server
  → Process.start()
    → ZygoteProcess 通过 socket 请求 Zygote

Zygote
  → forkAndSpecialize()
    ├─ 父进程：返回子进程 pid，继续等待下一次请求
    └─ 子进程：handleChildProc()
         → RuntimeInit.zygoteInit()
         → ActivityThread.main()
```

`system_server` 也是类似路径，只是入口不同：

```text
ZygoteInit.main()
  → forkSystemServer()
    └─ 子进程：handleSystemServerProcess()
         → SystemServer.main()
```

这里的“handle”不是 Linux 标准术语，而是面试里常用来描述“fork 后不 execve，由子进程继续处理后续初始化逻辑”。AOSP 里能看到的关键名字包括 `handleChildProc()`、`handleSystemServerProcess()`、`RuntimeInit.zygoteInit()`。

**为什么 App 不走 fork + execve？**

因为 Zygote 的价值就在于“先预加载，再 fork 共享”：

- Zygote 已经启动 ART runtime
- Zygote 已经预加载 Framework 常用类和资源
- fork 后通过 COW（Copy-On-Write）共享这些内存页
- 子进程只做 uid/gid、进程名、seinfo、capabilities、入口类等差异化初始化

如果 fork 后立刻 `execve()`：
- 子进程镜像会被目标程序替换
- Zygote 预加载的 Java/Framework/ART 状态基本失效
- App 进程又要重新初始化运行时和 Framework 基础环境
- Zygote 的启动速度和共享内存收益会被破坏

所以普通 App 进程必须保留 Zygote 的运行时现场，fork 后继续走 `ActivityThread.main()`，而不是 exec 一个新的 app 可执行文件。

**方式三：进程已存在，不创建新进程**

很多 `startActivity()` 并不会创建进程。AMS 会先判断目标进程是否已经存在：

```text
目标进程存在
  → AMS 通过 ApplicationThread Binder
  → scheduleTransaction()
  → ActivityThread 在主线程执行 Activity 生命周期

目标进程不存在
  → AMS 请求 Zygote fork 新进程
```

因此面试里问“启动 Activity 是否一定 fork 进程”，答案是否定的。只有目标进程不存在时，才会走 Zygote 创建进程。

**fork、vfork、clone、posix_spawn 的关系：**

这些是更底层或更通用的进程/线程创建接口：
- `fork()`：复制当前进程，父子进程从 fork 返回点继续执行
- `execve()`：替换当前进程镜像
- `clone()`：Linux 更底层接口，可精细控制共享哪些资源，线程本质上也基于 clone
- `vfork()`：特殊优化版本，通常用于马上 exec 的场景
- `posix_spawn()`：高级封装，语义上接近 fork + exec

Android 面试讨论系统启动和 App 进程时，重点不是 API 数量，而是区分：
- `init` 启动 native service：`fork + execve`
- Zygote 启动 App：`fork + handle`，不 `execve`

### 实战层

**面试常见追问：**

- `execve()` 会不会改变 pid？
- 为什么 Zygote fork App 后不能 execve？
- `system_server` 是 `fork + execve` 还是 `fork + handle`？
- `servicemanager` 是怎么启动的？
- `startActivity()` 是否一定会创建新进程？
- fork 后父进程和子进程分别做什么？

**排查和观察方式：**

```bash
# 看 init 启动的 native service 和 zygote/system_server
adb shell ps -A | grep -E 'init|servicemanager|zygote|system_server|surfaceflinger'

# 看 zygote rc 定义
adb shell cat /system/etc/init/zygote64.rc

# 看 native service rc 定义
adb shell grep -R "service servicemanager" /system/etc/init /vendor/etc/init 2>/dev/null
adb shell grep -R "service surfaceflinger" /system/etc/init /vendor/etc/init 2>/dev/null
```

**源码入口：**

官方 Android Code Search：
- `init` service 启动与 fork/exec：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/service.cpp
- `ZygoteConnection` 处理 fork 请求：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
- `Zygote` forkAndSpecialize：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/Zygote.java
- `ZygoteInit` fork system_server：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
- `RuntimeInit.zygoteInit()`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
- `ActivityThread.main()`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/ActivityThread.java
- `SystemServer.main()`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/java/com/android/server/SystemServer.java

推荐阅读顺序：
1. `system/core/init/service.cpp`：看 native service 如何 fork 后 exec
2. `init.zygote64.rc`：看 Zygote 本身如何作为 service 被 init 启动
3. `ZygoteConnection.java`：看 Zygote 如何接收创建 App 进程请求
4. `Zygote.java`：看 `forkAndSpecialize()` 如何 fork 并设置进程属性
5. `RuntimeInit.java`：看 App 子进程如何进入 Java main
6. `ActivityThread.java` / `SystemServer.java`：看 App 和 system_server 的最终入口

**容易答错的点：**

- App 进程不是 `fork + execve`，而是 Zygote `fork + handle`
- `execve()` 不创建新 pid，它替换当前进程镜像
- `system_server` 不是 init 直接 exec 出来的，它由 Zygote fork 后进入 `SystemServer.main()`
- `servicemanager` 这类 native daemon 通常由 init 按 service 配置启动，走 fork/exec 模型
- `startActivity()` 不一定创建进程，目标进程已存在时只走 Binder 调度生命周期

### 延伸问题

- [[Init进程启动与rc解析]]
- [[为什么App由Zygote-fork而非system_server]]
- [[ZygoteConnection-runOnce详解]]
- [[Zygote进程启动与App进程孵化]]
- [[ServiceManager与system_server区别]]
- [[Activity启动流程]]
- [[App冷启动全链路]]

## 记忆锚点

`fork + execve` 是“生个子进程再换脑子”，Zygote 的 `fork + handle` 是“复制 Zygote 后沿用预加载现场继续跑”。
