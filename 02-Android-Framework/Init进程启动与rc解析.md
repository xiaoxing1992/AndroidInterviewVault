---
tags: [android, 面试, framework, init, boot, S]
difficulty: S
frequency: high
---

# Init 进程启动与 rc 解析

## 问题

> Android 的 init 进程是怎么被加载起来的？它启动后做了哪些事情？init.rc 又是如何影响 Zygote 和系统服务启动的？

## 核心答案

`init` 是 Linux kernel 启动的第一个用户态进程，PID 固定是 1。设备上电后先经过 Boot ROM、Bootloader，再由 Bootloader 加载 Linux kernel 和 ramdisk；kernel 初始化完成后会执行 ramdisk 里的 `/init`。Android init 启动后分为 first stage、SELinux setup、second stage：first stage 负责早期挂载、fstab、AVB/dm-verity；SELinux setup 负责加载安全策略并切换执行上下文；second stage 负责 property service、解析 `init.rc` 和各分区 rc 文件，并按 action/service 启动 servicemanager、zygote 等系统关键进程。

## 深入解析

### 原理层

**从上电到 init 的链路：**

```text
Boot ROM
  → Bootloader
    → Linux kernel
      → 挂载 ramdisk / 初始 rootfs
        → exec /init
          → Android init，PID = 1
```

关键点：
- Boot ROM 是芯片固化代码，负责加载 Bootloader
- Bootloader 负责校验和加载 kernel、ramdisk、dtb/dtbo 等启动镜像内容
- kernel 初始化调度器、内存、驱动、文件系统后，启动第一个用户态程序
- Android 的第一个用户态程序通常是 ramdisk 中的 `/init`
- `init` 成为 PID 1 后，负责后续用户态进程的生命周期管理和孤儿进程回收

**Android init 的阶段化启动：**

```text
/init
  → first stage init
      - 创建和挂载早期目录：/dev、/proc、/sys
      - 读取 fstab
      - 挂载 system/vendor/product/odm 等分区
      - 处理 AVB / dm-verity

  → exec 自己进入 selinux_setup
      - 加载 SELinux policy
      - 设置 enforcing/permissive 状态
      - 切换到正确的 SELinux 上下文

  → exec 自己进入 second_stage
      - 初始化 property service
      - 启动 ueventd / signal handler 等基础能力
      - 解析 init.rc 和各分区 rc 文件
      - 执行 boot action
      - 启动 class main/core/late_start 等系统服务
```

**init.rc 的核心模型：**

`init.rc` 不是普通 shell 脚本，而是 Android init 自己解析的配置语言，核心由 action 和 service 组成。

```text
on boot
    chmod 0666 /dev/null
    class_start core

service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
```

| 类型 | 作用 | 例子 |
|------|------|------|
| action | 在特定触发条件下执行一组命令 | `on boot`、`on property:sys.boot_completed=1` |
| service | 定义一个可被 init 启动和管理的进程 | `service zygote ...` |
| class | 给 service 分组，便于批量启动 | `class core`、`class main`、`class late_start` |
| trigger | 触发 action 的条件 | boot、property 变化、device-added |

**init 到 Zygote 的关系：**

```text
kernel
  → /init
    → second stage init
      → 解析 init.rc / init.zygote64.rc
        → service zygote /system/bin/app_process64 ...
          → init fork/exec app_process64
            → ZygoteInit.main()
              → fork system_server
                → AMS/PMS/WMS 等系统服务启动
```

这条链路决定了 Android 系统服务启动顺序：
- kernel 只负责把 `/init` 拉起来
- init 负责解析 rc 并启动 Zygote
- Zygote 负责 fork `system_server`
- `system_server` 负责启动 AMS、PMS、WMS 等 Java Framework 服务
- AMS 再请求 Zygote fork 普通 App 进程

**service 启动机制：**

init 解析到 service 后，不一定立刻启动。service 是否启动取决于：
- 所属 class 是否被 `class_start` 启动
- 是否配置 `disabled`，disabled service 需要显式 `start`
- 是否满足 property trigger 或 boot action
- 进程退出后是否需要 restart

简化理解：

```text
解析 service 定义
  → 等待 action/trigger
    → class_start main
      → 启动 zygote
        → fork/exec /system/bin/app_process64
```

### 实战层

**面试常见追问：**

- `init` 为什么是 PID 1？
- `init.rc` 是 shell 脚本吗？
- first stage init 和 second stage init 分别做什么？
- `init`、Zygote、system_server、AMS 的启动关系是什么？
- 为什么 Zygote 是 init 启动的，而不是 AMS 启动的？
- `service`、`class`、`on boot`、`property trigger` 分别有什么作用？

**真机观察方式：**

```bash
# 查看 PID 1
adb shell ps -A | head
adb shell cat /proc/1/cmdline

# 查看 init rc 文件
adb shell cat /system/etc/init/hw/init.rc
adb shell ls /system/etc/init
adb shell ls /vendor/etc/init

# 查看 zygote 是由 init 管理的 service
adb shell cat /system/etc/init/zygote64.rc
adb shell ps -A | grep -E 'init|zygote|system_server'
```

**源码入口：**

官方 Android Code Search：
- `init` main 入口：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/main.cpp
- first stage init：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/first_stage_init.cpp
- second stage init：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/init.cpp
- init rc 解析器：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/parser.cpp
- service 管理：https://cs.android.com/android/platform/superproject/main/+/main:system/core/init/service.cpp
- AOSP 通用 `init.rc`：https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.rc
- Zygote rc：https://cs.android.com/android/platform/superproject/main/+/main:system/core/rootdir/init.zygote64.rc

Git 镜像入口：
- https://android.googlesource.com/platform/system/core/+/main/init/
- https://android.googlesource.com/platform/system/core/+/main/rootdir/

推荐阅读顺序：
1. `system/core/init/main.cpp`：看 init 程序如何区分 first stage、SELinux setup、second stage
2. `first_stage_init.cpp`：看早期挂载、fstab、AVB/dm-verity
3. `init.cpp`：看 second stage init 如何初始化 property service、解析 rc、进入事件循环
4. `parser.cpp`：看 `init.rc` 的 action/service 语法如何被解析
5. `service.cpp`：看 service 如何 fork/exec、restart、归属 class
6. `system/core/rootdir/init.rc`：看系统默认 action 和 service 分组
7. `init.zygote64.rc`：看 Zygote service 如何定义

**容易答错的点：**

- `init` 不是 Android Framework 服务，它是 kernel 拉起的第一个用户态进程
- `init.rc` 不是 shell 脚本，而是 Android init 的配置语言
- Zygote 不是 kernel 直接启动的，而是 init 解析 rc 后启动的 service
- `system_server` 不是 init 直接启动的，它由 Zygote fork 出来
- 真机上的 rc 文件会叠加 system、vendor、odm、product 等分区配置，不能只看 AOSP 通用 `init.rc`

### 延伸问题

- [[Android进程创建方式-fork-execve与Zygote]]
- [[ServiceManager与system_server区别]]
- [[Zygote进程启动与App进程孵化]]
- [[Activity启动流程]]
- [[App冷启动全链路]]
- [[Binder通信原理]]

## 记忆锚点

Android 启动主线：Bootloader 拉 kernel，kernel 拉 `/init`，init 解析 rc 拉 Zygote，Zygote fork `system_server`，AMS 再让 Zygote fork App。
