---
tags: [android, 面试, jetpack, room, A]
difficulty: A
frequency: medium
---

# Room 编译时生成原理

## 问题

> Room 是如何在编译时生成 DAO 实现的？它与 SQLite 的关系是什么？编译时做了哪些校验？

## 核心答案

Room 通过注解处理器（KSP/KAPT）在编译时解析 @Dao、@Entity、@Database 注解，生成 DAO 接口的实现类（包含完整的 SQLite 操作代码）和 Database 的实现类。编译时会校验 SQL 语法正确性、返回类型与查询列的匹配、外键关系等，将运行时错误前移到编译期。

## 深入解析

### 原理层

**编译时生成的代码结构：**
```
@Dao → UserDao_Impl.java (DAO 实现)
@Database → AppDatabase_Impl.java (Database 实现)
@Entity → 建表 SQL 语句
```

**DAO 实现生成示例：**
```kotlin
// 源码
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Long): User?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
}

// 生成的实现（简化）
class UserDao_Impl(private val __db: RoomDatabase) : UserDao {
    override suspend fun getUserById(userId: Long): User? {
        val _sql = "SELECT * FROM users WHERE id = ?"
        val _statement = RoomSQLiteQuery.acquire(_sql, 1)
        _statement.bindLong(1, userId)
        
        return withContext(__db.queryCoroutineContext) {
            val _cursor = DBUtil.query(__db, _statement, false, null)
            try {
                val _cursorIndexOfId = CursorUtil.getColumnIndexOrThrow(_cursor, "id")
                val _cursorIndexOfName = CursorUtil.getColumnIndexOrThrow(_cursor, "name")
                if (_cursor.moveToFirst()) {
                    User(
                        id = _cursor.getLong(_cursorIndexOfId),
                        name = _cursor.getString(_cursorIndexOfName)
                    )
                } else null
            } finally {
                _cursor.close()
                _statement.release()
            }
        }
    }
}
```

**编译时校验：**
1. **SQL 语法校验** — 解析 SQL 语句，检查语法错误
2. **表/列名校验** — 确认 SQL 中引用的表和列在 @Entity 中存在
3. **类型匹配** — 返回类型的字段与 SELECT 列对应
4. **参数绑定** — `:param` 与方法参数名匹配
5. **外键约束** — @ForeignKey 引用的表和列存在
6. **Migration 校验** — 检查 schema 变更是否有对应 Migration

**Database 实现生成：**
```java
class AppDatabase_Impl extends AppDatabase {
    @Override
    protected SupportSQLiteOpenHelper createOpenHelper(DatabaseConfiguration config) {
        return config.sqliteOpenHelperFactory.create(
            SupportSQLiteOpenHelper.Configuration.builder(config.context)
                .name(config.name)
                .callback(new RoomOpenHelper(config, new RoomOpenHelper.Delegate() {
                    public void createAllTables(SupportSQLiteDatabase db) {
                        db.execSQL("CREATE TABLE IF NOT EXISTS `users` (`id` INTEGER PRIMARY KEY, `name` TEXT)");
                    }
                }))
                .build()
        );
    }
    
    @Override
    public UserDao userDao() {
        if (_userDao == null) _userDao = new UserDao_Impl(this);
        return _userDao;
    }
}
```

### 实战层

- **TypeConverter**：自定义类型转换（Date↔Long、List↔JSON），编译时校验转换器覆盖所有需要的类型
- **Flow/LiveData 返回**：Room 自动生成 InvalidationTracker 监听表变化，数据更新时重新查询并发射
- **关系查询**：@Relation + @Embedded 处理一对多/多对多，生成多次查询代码
- **KSP 迁移**：Room 2.4+ 支持 KSP，编译速度提升显著
- **schema 导出**：`exportSchema = true` 导出 JSON schema 文件，用于 Migration 测试

```kotlin
// 自动观察数据变化
@Query("SELECT * FROM users")
fun getAllUsers(): Flow<List<User>>
// 生成的代码注册 InvalidationTracker，users 表变化时重新查询
```

### 延伸问题

- [[KSP与KAPT对比]]
- [[Room数据库迁移策略]]
- [[SQLite WAL模式与并发读写]]

## 记忆锚点

Room = 编译时把你的 @Query SQL 变成完整的 Cursor 操作代码。好处是 SQL 写错编译就报错，不用等运行时崩溃。
