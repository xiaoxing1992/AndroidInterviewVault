---
tags: [android, 面试, 存储, sharedpreferences, A]
difficulty: A
frequency: high
---

# SharedPreferences 问题与 DataStore 迁移

## 问题

> SharedPreferences 有哪些已知问题？为什么 Google 推荐迁移到 DataStore？Preferences DataStore 和 Proto DataStore 的区别是什么？

## 核心答案

SharedPreferences 的问题：全量读写（修改一个值要重写整个文件）、ANR 风险（apply 在 Activity onStop 时同步等待写入）、不支持跨进程、类型不安全。DataStore 基于协程和 Flow，异步非阻塞、支持类型安全（Proto）、有事务保证。Preferences DataStore 是 KV 存储（类似 SP），Proto DataStore 使用 Protocol Buffers 定义 schema，有类型安全和 schema 演进能力。

## 深入解析

### 原理层

**SharedPreferences 的问题：**

```java
// 问题1: 全量加载到内存
// SharedPreferencesImpl.startLoadFromDisk()
// 首次 getSharedPreferences 时，整个 XML 文件解析到 HashMap
// 大文件会阻塞主线程

// 问题2: apply() 的 ANR 风险
// apply() 看似异步，但在 Activity.onStop() 时会同步等待
// QueuedWork.waitToFinish() → 主线程阻塞等待所有 apply 完成
void handleStopActivity(IBinder token, ...) {
    // ...
    QueuedWork.waitToFinish();  // 这里会 ANR！
}

// 问题3: 全量写入
// 即使只改一个 key，commit/apply 都会重写整个 XML 文件
// 频繁写入 = 频繁全量 IO

// 问题4: 不支持跨进程
// MODE_MULTI_PROCESS 已废弃，多进程读写会数据丢失
```

**DataStore 架构：**
```kotlin
// Preferences DataStore（KV 存储）
val Context.dataStore by preferencesDataStore(name = "settings")

// 读取
val darkMode: Flow<Boolean> = context.dataStore.data.map { prefs ->
    prefs[booleanPreferencesKey("dark_mode")] ?: false
}

// 写入（事务性）
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { prefs ->
        prefs[booleanPreferencesKey("dark_mode")] = enabled
    }
}
```

```kotlin
// Proto DataStore（类型安全）
// 1. 定义 proto schema
// settings.proto
message Settings {
    bool dark_mode = 1;
    int32 font_size = 2;
    string language = 3;
}

// 2. 创建 Serializer
object SettingsSerializer : Serializer<Settings> {
    override val defaultValue: Settings = Settings.getDefaultInstance()
    override suspend fun readFrom(input: InputStream): Settings = Settings.parseFrom(input)
    override suspend fun writeTo(t: Settings, output: OutputStream) = t.writeTo(output)
}

// 3. 使用
val settingsStore: DataStore<Settings> = context.dataStore
val darkMode: Flow<Boolean> = settingsStore.data.map { it.darkMode }
```

**DataStore 底层实现：**
- 使用 `SingleProcessDataStore` 管理文件读写
- 写入通过 `actor` 模式串行化（避免并发冲突）
- 文件格式：Preferences 用 protobuf 编码的 PreferencesProto，Proto 用用户定义的 proto
- 原子写入：先写临时文件，再 rename（保证不会写坏）

### 实战层

**迁移方案：**
```kotlin
// 从 SharedPreferences 迁移到 DataStore
val dataStore = context.createDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(SharedPreferencesMigration(context, "old_sp_name"))
    }
)
// 迁移后自动删除旧 SP 文件
```

**选型建议：**
| 场景 | 推荐 |
|------|------|
| 简单 KV（开关、配置） | Preferences DataStore |
| 复杂结构化数据 | Proto DataStore |
| 需要跨进程 | MMKV 或 ContentProvider |
| 高频写入 | MMKV（mmap 性能更好） |

**坑点：**
- DataStore 不能在主线程同步读取（设计如此，强制异步）
- 启动时需要数据怎么办？用 `runBlocking { dataStore.data.first() }` 或提前预加载
- 文件损坏处理：`CorruptionHandler` 返回默认值

### 延伸问题

- [[MMKV原理-mmap与protobuf]]
- [[ANR分析与治理]]
- [[委托属性原理与自定义委托]]

## 记忆锚点

SP 三宗罪：全量 IO、apply 会 ANR、不支持跨进程。DataStore = 协程 + Flow + 原子写入，彻底异步化。Proto 版本还有类型安全。
