---
tags: [android, 面试, 存储, sqlite, S]
difficulty: S
frequency: medium
---

# SQLite WAL 模式与并发读写

## 问题

> SQLite 的 WAL 模式是什么？它如何实现读写并发？与默认的 Journal 模式有什么区别？Room 默认使用哪种模式？

## 核心答案

WAL（Write-Ahead Logging）模式将写操作追加到 WAL 文件而非直接修改数据库文件，读操作读取数据库文件+WAL 中的最新数据。这实现了读写并发：写不阻塞读，多个读可以并行。默认 Journal 模式写时锁定整个数据库，读写互斥。Room 从 2.0 开始默认启用 WAL 模式。

## 深入解析

### 原理层

**Journal 模式（默认）：**
```
写入流程:
1. 将原始页面拷贝到 .db-journal 文件（回滚日志）
2. 修改数据库文件中的页面
3. 删除 journal 文件（提交）
崩溃恢复: 用 journal 恢复原始数据

问题: 写入时需要独占锁，读操作被阻塞
```

**WAL 模式：**
```
写入流程:
1. 将修改追加到 .db-wal 文件（不修改主数据库）
2. 记录 WAL 索引（wal-index，共享内存 .db-shm）
3. Checkpoint 时将 WAL 内容合并回主数据库

读取流程:
1. 读取主数据库文件
2. 检查 WAL 中是否有更新的版本
3. 返回最新数据

并发模型:
- 读操作看到的是开始读取时的快照（MVCC 思想）
- 写操作追加到 WAL 末尾
- 读写互不阻塞 ✅
- 多个写仍然串行（WAL 只有一个写入点）
```

**WAL Checkpoint：**
```
WAL 文件不断增长 → 需要定期 checkpoint 合并回主数据库
触发条件:
- WAL 文件达到 1000 页（默认阈值）
- 数据库关闭时
- 手动调用 PRAGMA wal_checkpoint

Checkpoint 模式:
- PASSIVE: 不阻塞读写，尽量合并
- FULL: 等待所有读完成后合并
- RESTART: FULL + 重置 WAL 文件
- TRUNCATE: RESTART + 截断 WAL 文件为零
```

**对比表：**

| 特性 | Journal | WAL |
|------|---------|-----|
| 读写并发 | 互斥 | 读写并发 |
| 多读并发 | 支持 | 支持 |
| 多写并发 | 不支持 | 不支持 |
| 写入性能 | 中（随机 IO） | 高（顺序追加） |
| 读取性能 | 高 | 中（需查 WAL） |
| 崩溃恢复 | 回滚 journal | 重放 WAL |
| 文件数 | .db + .db-journal | .db + .db-wal + .db-shm |

### 实战层

**Room 中的 WAL：**
```kotlin
// Room 2.0+ 默认启用 WAL（API 16+）
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)  // 默认
    .build()

// 如果需要关闭 WAL（极少数场景）
.setJournalMode(RoomDatabase.JournalMode.TRUNCATE)
```

**并发最佳实践：**
```kotlin
// Room 内部使用 beginTransaction/endTransaction 保证原子性
// 多线程写入通过 SQLite 内部锁串行化

// 推荐：使用 withTransaction 进行批量操作
db.withTransaction {
    dao.deleteAll()
    dao.insertAll(newItems)
}
```

**坑点：**
- WAL 文件可能很大（checkpoint 不及时），影响备份和传输
- 不支持跨文件系统的数据库（WAL 需要共享内存 .shm）
- 多进程访问同一数据库时 WAL 可能有问题（建议用 ContentProvider 封装）

### 延伸问题

- [[Room数据库迁移策略]]
- [[数据库索引优化]]
- [[大数据量分页查询优化]]

## 记忆锚点

WAL = 写操作追加到日志文件，不动主数据库。读看快照，写追加末尾，所以读写不冲突。代价是多了 WAL 文件和 checkpoint 开销。
