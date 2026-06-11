---
tags: [android, 面试, 存储, mmkv, S]
difficulty: S
frequency: high
---

# MMKV 原理 - mmap 与 protobuf

## 问题

> MMKV 为什么比 SharedPreferences 快？mmap 的原理是什么？MMKV 如何保证数据一致性和跨进程安全？

## 核心答案

MMKV 使用 mmap 内存映射文件，写入时直接操作映射的内存区域（由 OS 异步刷盘），避免了传统文件 IO 的系统调用开销。数据格式使用 protobuf 编码（紧凑高效）。增量写入（append）而非全量重写。跨进程通过文件锁（flock）+ CRC 校验保证一致性。

## 深入解析

### 原理层

**mmap 原理：**
```
传统文件 IO:
  用户空间 Buffer → copy_to_user → 内核 Page Cache → 磁盘
  (2次拷贝 + 系统调用)

mmap:
  用户空间直接映射到内核 Page Cache → 磁盘
  (0次拷贝，直接读写内存，OS 负责刷盘)

void* ptr = mmap(NULL, fileSize, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
// ptr 指向的内存区域直接对应文件内容
// 写 ptr 就是写文件（由 OS 在合适时机刷盘）
```

**MMKV 数据格式（protobuf 编码）：**
```
文件结构:
┌──────────────────────────────────┐
│ 4 bytes: 有效数据长度              │
├──────────────────────────────────┤
│ Key1(varint长度+内容) + Value1    │
│ Key2(varint长度+内容) + Value2    │
│ ...                              │
│ Key_n + Value_n (append 追加)    │
├──────────────────────────────────┤
│ 空闲空间（预分配）                 │
└──────────────────────────────────┘
```

**写入策略 — 增量 append：**
```cpp
// 写入新 KV 对：直接 append 到末尾
void MMKV::setString(const string& key, const string& value) {
    auto data = encodeProtobuf(key, value);
    append(data);  // 追加到 mmap 区域末尾
    m_dic[key] = value;  // 更新内存字典
}

// 空间不足时：触发 full write back
// 1. 用内存字典（去重后）重新编码
// 2. 扩容文件（按页对齐）
// 3. 重新 mmap
```

**跨进程安全：**
```cpp
// 1. 文件锁（flock）保证互斥写入
flock(fd, LOCK_EX);  // 写锁
// ... 写入操作 ...
flock(fd, LOCK_UN);  // 解锁

// 2. CRC 校验保证数据完整性
// 每次写入后更新 CRC，读取时校验
// CRC 不匹配 → 数据损坏 → 从文件重新加载

// 3. 进程间状态同步
// 通过文件的 actualSize 和 sequence 判断其他进程是否修改了数据
// 如果 sequence 变化 → 重新加载文件内容到内存字典
```

**与 SharedPreferences 性能对比：**

| 操作 | SharedPreferences | MMKV |
|------|-------------------|------|
| 写入 1 个 key | 全量重写 XML | append 几十字节 |
| 读取 | 内存 HashMap | 内存 HashMap |
| 首次加载 | 解析 XML（慢） | mmap + protobuf 解码（快） |
| 崩溃安全 | apply 可能丢数据 | OS 保证 mmap 刷盘 |
| 跨进程 | 不支持 | 支持（文件锁） |

### 实战层

**基本使用：**
```kotlin
// 初始化
MMKV.initialize(context)

// 默认实例
val kv = MMKV.defaultMMKV()
kv.encode("name", "张三")
val name = kv.decodeString("name")

// 多进程实例
val multiProcessKV = MMKV.mmkvWithID("mydata", MMKV.MULTI_PROCESS_MODE)
```

**从 SP 迁移：**
```kotlin
val oldSP = getSharedPreferences("old_data", MODE_PRIVATE)
val kv = MMKV.mmkvWithID("old_data")
kv.importFromSharedPreferences(oldSP)
oldSP.edit().clear().apply()  // 清理旧数据
```

**注意事项：**
- MMKV 不支持 getAll()（protobuf 格式不保留类型信息）
- 文件大小会持续增长（append 模式），需要定期 trim
- 加密模式：`MMKV.mmkvWithID("secret", MMKV.SINGLE_PROCESS_MODE, "encryption_key")`
- 不适合存储大 value（>100KB），大数据用文件存储

### 延伸问题

- [[SharedPreferences问题与DataStore迁移]]
- [[Binder通信原理]]
- [[存储优化-IO调度与MMKV原理]]

## 记忆锚点

MMKV 快的三个原因：mmap 省掉系统调用（直接写内存）、protobuf 编码紧凑、append 增量写入（不全量重写）。跨进程靠文件锁。
