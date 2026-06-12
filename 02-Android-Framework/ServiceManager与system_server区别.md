---
tags: [android, 面试, framework, binder, servicemanager, system_server, S]
difficulty: S
frequency: high
---

# ServiceManager 与 system_server 区别

## 问题

> servicemanager 和 system_server 有什么区别？为什么系统里既需要 servicemanager，又需要 system_server？

## 核心答案

`servicemanager` 是 Binder 服务注册表，本质是一个 native 进程，由 `init` 直接启动，负责维护“服务名 → Binder 引用”的映射。`system_server` 是承载系统服务的 Java 进程，由 Zygote fork 出来，里面运行 AMS、PMS、WMS、PowerManagerService 等真正处理业务的系统服务。App 查找 AMS 时，会先通过 `ServiceManager.getService("activity")` 向 servicemanager 查询 AMS 的 Binder 引用，拿到引用后再通过 Binder 直接调用 system_server 里的 AMS。

## 深入解析

### 原理层

**启动关系：**

```text
init
  → 启动 servicemanager
  → 启动 zygote
      → zygote fork system_server
          → system_server 启动 AMS/PMS/WMS 等系统服务
              → 这些服务注册到 servicemanager
```

**核心区别：**

| 对象 | 本质 | 谁启动 | 主要职责 |
|------|------|--------|----------|
| `servicemanager` | native 进程 | `init` | 管理 Binder 服务名和 Binder 引用 |
| `ServiceManager` | Java 工具类 | App/framework 代码调用 | 封装 `getService/addService` 等查询注册 API |
| `system_server` | Java 进程 | Zygote fork | 承载 AMS、PMS、WMS、PowerManagerService 等系统服务 |

**一次系统服务查找过程：**

```text
App 进程
  → ServiceManager.getService("activity")
    → Binder 驱动
      → servicemanager 查询 "activity"
        → 返回 AMS 的 Binder 引用
  → ActivityManager.getService()
    → 使用 AMS Binder Proxy
      → 直接 Binder 调用 system_server 里的 AMS
```

关键点：
- `servicemanager` 只负责“找服务”，不处理 `startActivity()` 这类业务
- 真正处理 `startActivity()` 的是 `system_server` 里的 AMS
- 拿到服务 Binder 引用后，后续调用通常不再经过 servicemanager，而是直接走 Binder 到目标服务

**系统服务注册过程：**

```text
system_server 启动
  → SystemServer.startBootstrapServices()
    → 创建 ActivityManagerService / PackageManagerService 等
      → ServiceManager.addService("activity", ams)
        → servicemanager 记录 "activity" 到 AMS Binder 的映射
```

常见服务名：

| 服务名 | 实际服务 | 所在进程 |
|--------|----------|----------|
| `activity` | ActivityManagerService / ActivityTaskManagerService 相关入口 | `system_server` |
| `package` | PackageManagerService | `system_server` |
| `window` | WindowManagerService | `system_server` |
| `power` | PowerManagerService | `system_server` |

**为什么不能只有 system_server？**

Client 进程需要一个稳定入口来发现系统服务。系统服务很多，而且 Binder 引用是运行时对象，不能靠硬编码地址访问。`servicemanager` 就是 Binder 世界的命名服务，提供统一的服务发现能力。

**为什么不能只有 servicemanager？**

`servicemanager` 只是注册表，不承载 AMS、PMS、WMS 的复杂业务逻辑。系统服务需要运行在一个长期存活、能加载 Java Framework、能统一调度生命周期的进程里，这个进程就是 `system_server`。

**和 Binder 驱动的关系：**

- Binder 驱动提供内核 IPC 能力
- `servicemanager` 是 Binder 上下文管理者，负责服务注册和查询
- `system_server` 中的系统服务是 Binder Server
- App、framework 组件通常是 Binder Client

### 实战层

**面试常见追问：**

- `servicemanager` 和 Java 的 `ServiceManager` 是一回事吗？
- `ServiceManager.getService("activity")` 返回的是什么？
- `startActivity()` 是 servicemanager 处理的吗？
- AMS、PMS、WMS 为什么大多在 `system_server` 里？
- 系统服务注册后，后续每次调用都要经过 servicemanager 吗？

**排查和观察方式：**

```bash
# 查看关键进程
adb shell ps -A | grep -E 'servicemanager|system_server|zygote'

# 查看 Binder 服务列表
adb shell service list

# 查询某个服务
adb shell service check activity
adb shell service check package
adb shell service check window
```

**源码入口：**

官方 Android Code Search：
- `servicemanager` native 进程：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/native/cmds/servicemanager/
- Java `ServiceManager`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/os/ServiceManager.java
- `SystemServer.java`：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/java/com/android/server/SystemServer.java
- AMS 注册入口：https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

推荐阅读顺序：
1. `init.rc`：看 `servicemanager` 和 `zygote` 分别由 init 启动
2. `SystemServer.java`：看系统服务如何被创建
3. `ServiceManager.addService()`：看服务如何注册
4. `ServiceManager.getService()`：看 Client 如何查服务
5. `servicemanager` native 源码：看服务名到 Binder 引用的注册表实现

**容易答错的点：**

- `servicemanager` 不是 `system_server` 的一个 Java 对象，它是独立 native 进程
- Java `ServiceManager` 不是服务本身，只是访问 servicemanager 的工具类
- `servicemanager` 不执行 AMS/PMS/WMS 的业务逻辑
- `system_server` 不是 Binder 注册表，它只是大量系统服务所在的进程
- `getService()` 只负责拿 Binder 引用，拿到后业务调用走目标服务的 Binder

### 延伸问题

- [[Binder通信原理]]
- [[Init进程启动与rc解析]]
- [[Zygote进程启动与App进程孵化]]
- [[系统服务启动流程]]
- [[SystemServer启动流程]]
- [[为什么App由Zygote-fork而非system_server]]
- [[Activity启动流程]]

## 记忆锚点

`servicemanager` 管“服务名到 Binder 的映射”，`system_server` 跑“真正干活的系统服务”。
