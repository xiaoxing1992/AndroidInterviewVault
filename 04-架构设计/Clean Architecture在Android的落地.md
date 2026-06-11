---
tags: [android, 面试, 架构, clean, A]
difficulty: A
frequency: high
---

# Clean Architecture 在 Android 的落地

## 问题

> Clean Architecture 的核心原则是什么？在 Android 项目中如何落地？Domain 层为什么不能依赖 Android Framework？UseCase 的价值是什么？

## 核心答案

Clean Architecture 核心是依赖规则：外层依赖内层，内层不知道外层的存在。Android 落地为三层：Presentation（UI+ViewModel）→ Domain（UseCase+Entity+Repository 接口）→ Data（Repository 实现+数据源）。Domain 层纯 Kotlin 无 Android 依赖，保证业务逻辑可测试、可复用。UseCase 封装单一业务操作，是 ViewModel 和 Data 层之间的边界。

## 深入解析

### 原理层

**依赖方向（由外向内）：**
```
┌─────────────────────────────────────────┐
│  Presentation (UI/ViewModel)            │  ← Android Framework
│  ┌─────────────────────────────────┐    │
│  │  Domain (UseCase/Entity)        │    │  ← 纯 Kotlin
│  │  ┌─────────────────────────┐    │    │
│  │  │  Entity (业务模型)       │    │    │
│  │  └─────────────────────────┘    │    │
│  └─────────────────────────────────┘    │
│  Data (Repository 实现/DataSource)      │  ← Android Framework
└─────────────────────────────────────────┘

依赖方向: Presentation → Domain ← Data
         (Domain 不依赖任何外层)
```

**各层职责：**
```kotlin
// Domain 层 — 纯 Kotlin 模块
// Entity
data class User(val id: String, val name: String, val email: String)

// Repository 接口（Domain 定义，Data 实现）
interface UserRepository {
    suspend fun getUser(id: String): User
    suspend fun updateUser(user: User)
}

// UseCase（单一职责）
class GetUserUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(userId: String): Result<User> {
        return runCatching { repository.getUser(userId) }
    }
}

// Data 层 — 实现 Repository
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao,
    private val mapper: UserMapper
) : UserRepository {
    override suspend fun getUser(id: String): User {
        return try {
            val dto = api.getUser(id)
            dao.insert(mapper.toEntity(dto))
            mapper.toDomain(dto)
        } catch (e: IOException) {
            mapper.toDomain(dao.getUser(id))  // 离线降级
        }
    }
}

// Presentation 层
class UserViewModel(private val getUserUseCase: GetUserUseCase) : ViewModel() {
    fun loadUser(id: String) {
        viewModelScope.launch {
            _state.value = getUserUseCase(id).fold(
                onSuccess = { UiState.Success(it) },
                onFailure = { UiState.Error(it.message) }
            )
        }
    }
}
```

**依赖反转实现：**
```
Domain 定义接口: interface UserRepository
Data 实现接口:   class UserRepositoryImpl : UserRepository
DI 绑定:        @Binds fun bindUserRepo(impl: UserRepositoryImpl): UserRepository

结果: Domain 不知道 Data 层的存在，但运行时用的是 Data 层的实现
```

### 实战层

**UseCase 的价值：**
- 封装业务规则（不只是转发 Repository 调用）
- ViewModel 不直接依赖 Repository，方便替换数据源
- 可组合：复杂业务由多个 UseCase 组合
- 可测试：纯函数，mock Repository 即可

**何时可以省略 UseCase：**
- 只是简单的 CRUD 透传（无业务逻辑）
- 小型项目，ViewModel 直接调用 Repository 也可接受
- 但保持一致性比省代码更重要

**数据模型映射：**
```
API DTO (data layer) → Domain Entity (domain layer) → UI Model (presentation layer)
UserResponse         → User                         → UserUiState

每层有自己的模型，通过 Mapper 转换
避免 API 字段变化直接影响 UI
```

**模块划分：**
```
:core:domain     — UseCase、Entity、Repository 接口（纯 Kotlin library）
:core:data       — Repository 实现、DataSource、DTO
:feature:user    — UserViewModel、UserScreen（依赖 :core:domain）
```

### 延伸问题

- [[MVI vs MVVM vs MVC对比与选型]]
- [[模块化依赖管理策略]]
- [[Hilt依赖注入原理]]

## 记忆锚点

Clean Architecture = 洋葱圈，越内层越稳定。Domain 是核心（纯业务逻辑，不碰 Android），外层通过接口反转依赖内层。UseCase 是 ViewModel 和 Data 之间的防腐层。
