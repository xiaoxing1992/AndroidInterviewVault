---
tags: [android, 面试, 存储, room, A]
difficulty: A
frequency: high
---

# Room 数据库迁移策略

## 问题

> Room 数据库版本升级时如何处理 schema 变更？Migration 和 AutoMigration 的区别是什么？如何测试迁移的正确性？

## 核心答案

Room 通过 Migration 对象定义版本间的 SQL 变更（ALTER TABLE、CREATE TABLE 等）。AutoMigration（Room 2.4+）可以自动处理简单变更（加列、加表、改列名），复杂变更仍需手动 Migration。测试通过 MigrationTestHelper 验证迁移后 schema 正确性和数据完整性。如果没有提供 Migration，Room 默认会销毁重建数据库（丢失数据）。

## 深入解析

### 原理层

**手动 Migration：**
```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE users ADD COLUMN age INTEGER NOT NULL DEFAULT 0")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // SQLite 不支持 ALTER COLUMN，需要重建表
        db.execSQL("CREATE TABLE users_new (id INTEGER PRIMARY KEY, name TEXT NOT NULL, age INTEGER NOT NULL)")
        db.execSQL("INSERT INTO users_new (id, name, age) SELECT id, name, age FROM users")
        db.execSQL("DROP TABLE users")
        db.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}

Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

**AutoMigration（Room 2.4+）：**
```kotlin
@Database(
    version = 3,
    entities = [User::class],
    autoMigrations = [
        AutoMigration(from = 1, to = 2),  // 简单加列
        AutoMigration(from = 2, to = 3, spec = Migration2To3::class)  // 需要额外信息
    ]
)
abstract class AppDatabase : RoomDatabase()

// 当 AutoMigration 需要额外信息时
@RenameColumn(tableName = "users", fromColumnName = "user_name", toColumnName = "name")
class Migration2To3 : AutoMigrationSpec
```

**AutoMigration 支持的操作：**
- 新增列（带默认值）
- 新增表
- 删除列（@DeleteColumn）
- 重命名列（@RenameColumn）
- 重命名表（@RenameTable）
- 删除表（@DeleteTable）

**不支持（需要手动 Migration）：**
- 修改列类型
- 修改主键
- 复杂数据转换
- 索引变更

**Room 迁移校验原理：**
```
1. 打开数据库，读取当前版本号
2. 查找从当前版本到目标版本的 Migration 路径
3. 依次执行 Migration
4. 执行后校验 schema 是否与 @Entity 定义一致
   → 不一致则抛出 IllegalStateException
5. 校验通过，更新版本号
```

### 实战层

**迁移测试：**
```kotlin
@RunWith(AndroidJUnit4::class)
class MigrationTest {
    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java
    )
    
    @Test
    fun migrate1To2() {
        // 创建 v1 数据库并插入数据
        helper.createDatabase("app.db", 1).apply {
            execSQL("INSERT INTO users (id, name) VALUES (1, '张三')")
            close()
        }
        
        // 执行迁移
        val db = helper.runMigrationsAndValidate("app.db", 2, true, MIGRATION_1_2)
        
        // 验证数据
        val cursor = db.query("SELECT * FROM users WHERE id = 1")
        assertTrue(cursor.moveToFirst())
        assertEquals(0, cursor.getInt(cursor.getColumnIndex("age")))  // 默认值
    }
}
```

**fallbackToDestructiveMigration：**
```kotlin
// 找不到 Migration 时销毁重建（开发阶段可用，生产慎用）
Room.databaseBuilder(...)
    .fallbackToDestructiveMigration()
    .build()

// 只对特定版本降级时销毁
.fallbackToDestructiveMigrationOnDowngrade()

// 只对特定起始版本销毁
.fallbackToDestructiveMigrationFrom(1, 2)
```

**最佳实践：**
- 导出 schema JSON（`exportSchema = true`），用于 CI 校验
- 每次版本升级都写迁移测试
- 复杂迁移分步执行，每步可验证
- 生产环境永远不要用 fallbackToDestructiveMigration

### 延伸问题

- [[Room编译时生成原理]]
- [[SQLite WAL模式与并发读写]]
- [[数据库索引优化]]

## 记忆锚点

Room 迁移 = 版本号 + Migration SQL。AutoMigration 处理简单变更（加列/加表），复杂的（改类型/改主键）手动写。必须测试，否则用户升级就崩。
