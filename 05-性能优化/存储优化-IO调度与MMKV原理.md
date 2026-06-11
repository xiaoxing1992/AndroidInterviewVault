---
tags:
  - 性能优化
  - 存储优化
  - MMKV
  - mmap
  - IO
difficulty: A
frequency: medium
---

# 存储优化-IO调度与MMKV原理

## 问题

> MMKV 为什么比 SharedPreferences 快？mmap 的原理是什么？文件 IO 有哪些优化手段？

## 核心答案（30秒版本）

MMKV 基于 mmap 内存映射文件，写入时直接操作内存（由 OS 异步刷盘），避免了 SharedPreferences 的 `apply()` 异步写盘延迟和 `commit()` 同步阻塞问题。mmap 将文件映射到进程虚拟地址空间，读写文件等同于读写内存，省去了用户态到内核态的数据拷贝（零拷贝）。MMKV 使用 Protobuf 编码 + append-only 写入 + 定期 full-write 回收空间的策略，兼顾性能和空间效率。

## 深入解析

### 原理层（源码级）

**SharedPreferences 的问题：**

```java
// frameworks/base/core/java/android/app/SharedPreferencesImpl.java

// commit() - 同步写入，阻塞调用线程
public boolean commit() {
    MemoryCommitResult mcr = commitToMemory();
    // 同步等待写入完成
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null);
    mcr.writtenToDiskLatch.await(); // 阻塞！
    return mcr.writeToDiskResult;
}

// apply() - 异步写入，但有隐患
public void apply() {
    MemoryCommitResult mcr = commitToMemory();
    // 提交到 QueuedWork
    QueuedWork.addFinisher(mcr.writtenToDiskLatch::await);
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
}

// 隐患：Activity.onStop() 中会等待所有 apply 完成！
// frameworks/base/core/java/android/app/ActivityThread.java
private void handleStopActivity(IBinder token) {
    // 等待所有 pending 的 SharedPreferences 写入完成
    QueuedWork.waitToFinish(); // 可能导致 ANR！
}
```

**mmap 原理：**

```
传统文件 IO（read/write）：
用户空间 Buffer ←→ 内核 Page Cache ←→ 磁盘
                 ↑ 两次数据拷贝 ↑

mmap 内存映射：
用户空间虚拟地址 → 直接映射到 → 内核 Page Cache → 磁盘
                 ↑ 零拷贝，直接操作 ↑

┌─────────────────────────────────────────┐
│ 进程虚拟地址空间                          │
│  ┌──────────┐                           │
│  │ mmap 区域 │ ← 虚拟地址直接映射         │
│  └──────────┘                           │
│       ↓ (页表映射)                        │
│  ┌──────────┐                           │
│  │Page Cache│ ← 内核管理，异步刷盘        │
│  └──────────┘                           │
│       ↓ (OS 异步 writeback)              │
│  ┌──────────┐                           │
│  │   磁盘    │                           │
│  └──────────┘                           │
└─────────────────────────────────────────┘

关键优势：
1. 零拷贝：省去用户态↔内核态的数据复制
2. 异步刷盘：OS 负责将脏页写回磁盘
3. 进程崩溃安全：mmap 的数据在内核 Page Cache 中，
   即使进程崩溃，OS 仍会将脏页刷盘
```

**MMKV 核心实现：**

```cpp
// MMKV/Core/MMKV.cpp (简化)
class MMKV {
    // 内存映射文件
    MemoryFile *m_file;
    // 数据编码（Protobuf 格式）
    MMBuffer m_output;
    // 内存中的 key-value 字典
    unordered_map<string, MMBuffer> m_dic;

    // 写入操作
    bool setDataForKey(MMBuffer &&data, const string &key) {
        // 1. 更新内存字典
        m_dic[key] = data;

        // 2. Protobuf 编码
        auto encodedData = MiniPBCoder::encodeDataWithObject(data, key);

        // 3. Append 到 mmap 文件末尾
        if (m_output.length() + encodedData.length() <= m_file->getFileSize()) {
            // 空间足够，直接 append
            m_file->append(encodedData);
        } else {
            // 空间不足，full write back（去重 + 重写）
            fullWriteBack();
        }
        return true;
    }

    // 空间回收：将内存字典重新序列化写入
    void fullWriteBack() {
        auto allData = MiniPBCoder::encodeDataWithObject(m_dic);
        m_file->rewrite(allData);
    }
};
```

**MMKV 数据格式（Protobuf 编码）：**

```
文件结构：
┌────────────────────────────────────────┐
│ [4 bytes] 有效数据长度                   │
├────────────────────────────────────────┤
│ [key1_len][key1][value1_len][value1]   │ ← Protobuf 编码
│ [key2_len][key2][value2_len][value2]   │
│ ...                                    │
│ [key1_len][key1][new_value1]           │ ← append 的更新（覆盖旧值）
├────────────────────────────────────────┤
│ [未使用空间]                             │
└────────────────────────────────────────┘

读取时：顺序解析，后出现的 key 覆盖先出现的（最终一致性）
空间回收：当 append 空间不足时，触发 full write back
```

### 实战层（项目经验）

**1. MMKV 使用最佳实践：**

```kotlin
// 初始化
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        MMKV.initialize(this)
    }
}

// 基本使用
val kv = MMKV.defaultMMKV()
kv.encode("username", "张三")
kv.encode("login_count", 42)
kv.encode("is_vip", true)

val name = kv.decodeString("username", "")
val count = kv.decodeInt("login_count", 0)

// 多进程支持
val multiProcessKV = MMKV.mmkvWithID("shared_data", MMKV.MULTI_PROCESS_MODE)

// 从 SharedPreferences 迁移
val oldSp = getSharedPreferences("old_sp", MODE_PRIVATE)
kv.importFromSharedPreferences(oldSp)
oldSp.edit().clear().apply() // 清理旧数据
```

**2. 数据库优化（SQLite/Room）：**

```kotlin
// Room 数据库优化配置
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    companion object {
        fun create(context: Context): AppDatabase {
            return Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
                // WAL 模式（Write-Ahead Logging）
                .setJournalMode(JournalMode.WRITE_AHEAD)
                // 设置连接池大小
                .setQueryExecutor(Executors.newFixedThreadPool(4))
                .build()
        }
    }
}

// 批量插入优化
@Dao
interface UserDao {
    // 单条插入 - 慢
    @Insert
    suspend fun insert(user: User)

    // 批量插入 - 快（单个事务）
    @Insert
    suspend fun insertAll(users: List<User>)

    // 手动事务控制
    @Transaction
    suspend fun replaceAll(users: List<User>) {
        deleteAll()
        insertAll(users)
    }
}

// 索引优化
@Entity(
    tableName = "orders",
    indices = [
        Index(value = ["user_id"]),           // 单列索引
        Index(value = ["status", "created_at"]) // 复合索引
    ]
)
data class Order(
    @PrimaryKey val id: Long,
    @ColumnInfo(name = "user_id") val userId: Long,
    val status: String,
    @ColumnInfo(name = "created_at") val createdAt: Long
)
```

**3. 文件 IO 优化策略：**

```kotlin
// 1. 使用 BufferedInputStream/BufferedOutputStream
fun copyFileBuffered(src: File, dst: File) {
    src.inputStream().buffered(8192).use { input ->
        dst.outputStream().buffered(8192).use { output ->
            input.copyTo(output, bufferSize = 8192)
        }
    }
}

// 2. 大文件使用 FileChannel（零拷贝）
fun copyFileChannel(src: File, dst: File) {
    FileInputStream(src).channel.use { srcChannel ->
        FileOutputStream(dst).channel.use { dstChannel ->
            srcChannel.transferTo(0, srcChannel.size(), dstChannel)
        }
    }
}

// 3. 随机读写使用 RandomAccessFile + MappedByteBuffer
fun mmapRead(file: File, offset: Long, length: Int): ByteArray {
    RandomAccessFile(file, "r").use { raf ->
        val channel = raf.channel
        val buffer = channel.map(
            FileChannel.MapMode.READ_ONLY, offset, length.toLong())
        val bytes = ByteArray(length)
        buffer.get(bytes)
        return bytes
    }
}
```

**4. 序列化性能对比：**

```kotlin
// 性能排序（快→慢）：
// FlatBuffers > Protobuf > MMKV 内置 > JSON(Moshi) > JSON(Gson) > Serializable

// Protobuf 使用示例
// user.proto
// message User {
//     int64 id = 1;
//     string name = 2;
//     int32 age = 3;
// }

// 生成的代码直接使用
val user = User.newBuilder()
    .setId(1L)
    .setName("张三")
    .setAge(25)
    .build()

// 序列化（比 JSON 小 30-50%，快 5-10 倍）
val bytes = user.toByteArray()
// 反序列化
val parsed = User.parseFrom(bytes)
```

**5. 磁盘空间监控：**

```kotlin
class StorageMonitor(private val context: Context) {
    fun checkStorageHealth(): StorageState {
        val stat = StatFs(context.filesDir.path)
        val availableBytes = stat.availableBytes
        val totalBytes = stat.totalBytes
        val usageRatio = 1 - (availableBytes.toFloat() / totalBytes)

        return when {
            availableBytes < 100 * 1024 * 1024 -> StorageState.CRITICAL // < 100MB
            usageRatio > 0.9 -> StorageState.WARNING
            else -> StorageState.NORMAL
        }
    }

    // 清理策略
    fun cleanupIfNeeded() {
        if (checkStorageHealth() != StorageState.NORMAL) {
            // 按优先级清理
            clearExpiredCache()      // 1. 过期缓存
            clearImageDiskCache()    // 2. 图片磁盘缓存
            clearOldLogs()           // 3. 旧日志
            compactDatabase()        // 4. 数据库 VACUUM
        }
    }
}
```

### 延伸问题

- [[内存优化-LeakCanary与内存抖动]] — mmap 与内存管理的关系？
- [[ANR分析与治理]] — SharedPreferences apply 导致的 ANR？
- [[启动优化-耗时分析与优化手段]] — 启动阶段的 IO 优化？
- [[电量优化-WakeLock与JobScheduler]] — 频繁 IO 对电量的影响？

## 记忆锚点

MMKV 快在 mmap（零拷贝 + OS 异步刷盘）+ Protobuf 编码 + append-only 写入；SP 慢在 XML 全量写入 + apply 在 onStop 时同步等待；IO 优化 = Buffer + Channel 零拷贝 + mmap 内存映射。
