---
tags: [android, 面试, 存储, contentprovider, A]
difficulty: A
frequency: medium
---

# ContentProvider 跨进程数据共享

## 问题

> ContentProvider 如何实现跨进程数据共享？URI 的结构是什么？如何保证数据安全？批量操作如何优化性能？

## 核心答案

ContentProvider 通过 Binder 机制暴露 CRUD 接口给其他进程，使用 URI 标识数据资源。URI 结构为 `content://authority/path/id`。安全性通过权限声明（readPermission/writePermission）、URI 临时权限（grantUriPermission）和路径权限（path-permission）控制。批量操作使用 `applyBatch` + `ContentProviderOperation` 在单次 Binder 调用中执行多个操作。

## 深入解析

### 原理层

**URI 结构：**
```
content://com.example.app.provider/users/123
│         │                        │     │
scheme    authority                path   id

- scheme: 固定为 content://
- authority: Provider 的唯一标识（通常是包名+provider）
- path: 数据表/集合名
- id: 具体记录（可选）
```

**ContentProvider 方法线程模型：**
```java
// query/insert/update/delete 在 Binder 线程池中执行
// 多个客户端并发调用时，方法可能被多线程同时调用
// 需要自己保证线程安全

// ContentProvider.onCreate() 在主线程执行
// 且在 Application.onCreate() 之前
```

**权限控制：**
```xml
<provider
    android:name=".UserProvider"
    android:authorities="com.example.provider"
    android:exported="true"
    android:readPermission="com.example.READ_DATA"
    android:writePermission="com.example.WRITE_DATA">
    
    <!-- 路径级别权限 -->
    <path-permission
        android:pathPrefix="/public"
        android:readPermission="com.example.READ_PUBLIC" />
    
    <!-- URI 临时授权 -->
    <grant-uri-permission android:pathPattern="/share/.*" />
</provider>
```

**批量操作优化：**
```kotlin
// 单次 Binder 调用执行多个操作
val operations = ArrayList<ContentProviderOperation>()
for (user in users) {
    operations.add(
        ContentProviderOperation.newInsert(CONTENT_URI)
            .withValue("name", user.name)
            .withValue("email", user.email)
            .build()
    )
}
// 一次 IPC 调用完成所有操作
contentResolver.applyBatch(AUTHORITY, operations)

// 对比：逐条 insert 需要 N 次 Binder 调用
```

**ContentResolver 查询优化：**
```kotlin
// CursorLoader（已废弃）→ 现在用协程 + Flow
fun observeUsers(): Flow<List<User>> = callbackFlow {
    val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
        override fun onChange(selfChange: Boolean) {
            trySend(queryUsers())
        }
    }
    contentResolver.registerContentObserver(CONTENT_URI, true, observer)
    send(queryUsers())  // 初始数据
    awaitClose { contentResolver.unregisterContentObserver(observer) }
}
```

### 实战层

- **FileProvider**：分享文件给其他 App 的标准方式，通过临时 URI 权限授权
- **ContactsProvider**：系统联系人数据的 ContentProvider，展示了复杂 Provider 的设计
- **通知数据变化**：`contentResolver.notifyChange(uri, null)` 通知观察者
- **分页查询**：`LIMIT` 和 `OFFSET` 通过 URI 参数或 Bundle 传递
- **坑点**：Cursor 必须 close，否则内存泄漏；跨进程传输大量数据考虑用 ContentProvider + 文件 URI

### 延伸问题

- [[ContentProvider启动时序]]
- [[Binder通信原理]]
- [[文件存储-Scoped Storage适配]]

## 记忆锚点

ContentProvider = 数据库的 REST API。URI 是地址，CRUD 是操作，权限是门禁。批量操作用 applyBatch 减少 Binder 调用次数。
