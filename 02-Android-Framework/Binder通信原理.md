---
tags: [android, 面试, framework, binder, S]
difficulty: S
frequency: high
---

# Binder 通信原理

## 问题

> Android 为什么选择 Binder 作为主要的 IPC 机制？Binder 的一次拷贝是如何实现的？请描述一次完整的 Binder 通信过程。

## 核心答案

Binder 基于 mmap 内存映射实现一次拷贝（传统 IPC 需要两次），兼具性能好、安全性高（内核校验 UID/PID）、支持 C/S 架构的优点。通信过程：Client 通过 Proxy 将数据序列化写入内核的 Binder 驱动，驱动通过 mmap 映射将数据直接拷贝到 Server 的用户空间，Server 端的 Stub 反序列化后调用实际方法。

## 深入解析

### 原理层

**为什么选 Binder 而非其他 IPC：**

| IPC 方式 | 拷贝次数 | 安全性 | 特点 |
|----------|----------|--------|------|
| 共享内存 | 0 | 低（无身份校验） | 需要自己同步 |
| Binder | 1 | 高（内核校验 PID/UID） | C/S 架构，面向对象 |
| Socket | 2 | 中 | 通用但开销大 |
| 管道/消息队列 | 2 | 低 | 不适合复杂通信 |

**一次拷贝原理（mmap）：**
```
Client 进程          内核空间              Server 进程
┌──────────┐    ┌──────────────┐    ┌──────────────┐
│ 用户空间  │    │ Binder 驱动   │    │  用户空间     │
│          │    │              │    │              │
│ data ────────→│ copy_from_user│    │              │
│          │    │      │       │    │              │
│          │    │      ↓       │    │              │
│          │    │ 内核缓冲区 ═══════════ mmap 映射区  │
│          │    │              │    │  (直接可读)   │
└──────────┘    └──────────────┘    └──────────────┘
```
- Server 进程通过 mmap 将一块内核缓冲区映射到自己的用户空间
- Client 数据通过 `copy_from_user` 拷贝到内核缓冲区（1 次拷贝）
- Server 直接读取映射区域的数据（0 次拷贝）
- 总计：1 次拷贝（传统方式需要 copy_from_user + copy_to_user = 2 次）

**完整通信过程：**
```
Client                    Binder Driver                 Server
──────                    ─────────────                 ──────
1. BpBinder.transact()
   → 序列化数据到 Parcel
   → ioctl(BINDER_WRITE_READ)
                          2. 找到目标进程的 binder_node
                          → copy_from_user 拷贝数据
                          → 将事务插入目标线程的 todo 队列
                          → 唤醒目标线程
                                                       3. binder_thread_read()
                                                       → BBinder.onTransact()
                                                       → Stub.onTransact()
                                                       → 执行实际方法
                                                       → 写入返回数据
                          4. 将返回数据传回 Client
5. 收到返回值
   → 反序列化 Parcel
```

**关键数据结构：**
- `binder_proc` — 每个使用 Binder 的进程
- `binder_node` — Server 端的 Binder 实体
- `binder_ref` — Client 端对 Server Binder 的引用
- `binder_transaction` — 一次通信事务

**AIDL 生成代码结构：**
```java
interface IMyService extends IInterface {
    // Stub（Server 端）
    abstract class Stub extends Binder implements IMyService {
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) {
            switch (code) {
                case TRANSACTION_getData:
                    String result = this.getData();
                    reply.writeString(result);
                    return true;
            }
        }
        // Proxy（Client 端）
        private static class Proxy implements IMyService {
            public String getData() {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                mRemote.transact(TRANSACTION_getData, data, reply, 0);
                return reply.readString();
            }
        }
    }
}
```

### 实战层

- **Binder 线程池**：默认最大 16 个线程（`BINDER_SET_MAX_THREADS`），高并发场景可能成为瓶颈
- **数据大小限制**：单次 Binder 传输上限约 1MB（`BINDER_VM_SIZE = 1MB - 8KB`），大数据用共享内存或 ContentProvider
- **死亡通知**：`linkToDeath` 注册 DeathRecipient，Server 进程死亡时收到回调
- **oneway 关键字**：异步调用，Client 不等待 Server 返回（如 `ActivityManager` 的很多调用）

### 延伸问题

- [[ServiceManager与system_server区别]]
- [[Activity启动流程]]
- [[ContentProvider启动时序]]
- [[跨进程通信架构设计]]

## 记忆锚点

Binder 一次拷贝的秘密：Server 用 mmap 把内核缓冲区映射到自己家，Client 的数据拷贝到内核后 Server 直接就能看到。
