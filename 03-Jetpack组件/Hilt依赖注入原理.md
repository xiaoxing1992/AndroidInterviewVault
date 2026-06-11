---
tags: [android, 面试, jetpack, hilt, S]
difficulty: S
frequency: high
---

# Hilt 依赖注入原理

## 问题

> Hilt 的底层原理是什么？它是如何在编译时生成代码的？Component 层级结构是怎样的？与 Dagger 的关系是什么？

## 核心答案

Hilt 是 Dagger 的 Android 专用封装，通过编译时注解处理（KSP/KAPT）生成 Dagger Component 和 Module 代码。它预定义了与 Android 生命周期对应的 Component 层级（SingletonComponent → ActivityComponent → FragmentComponent），通过字节码转换（Gradle Plugin）将生成的注入代码织入 Application/Activity 基类，实现无感注入。

## 深入解析

### 原理层

**Component 层级：**
```
SingletonComponent (Application 生命周期)
├── ServiceComponent
├── ActivityRetainedComponent (ViewModel 生命周期)
│   └── ActivityComponent
│       ├── FragmentComponent
│       └── ViewComponent
│           └── ViewWithFragmentComponent
└── ViewModelComponent
```

**编译时代码生成流程：**
```
@HiltAndroidApp class MyApp : Application()
        ↓ (注解处理器)
生成 Hilt_MyApp.java (基类，包含 Component 初始化)
生成 DaggerMyApp_HiltComponents_SingletonC.java (Dagger Component 实现)
        ↓ (Gradle Transform / ASM 字节码修改)
MyApp 的父类被替换为 Hilt_MyApp
```

**@HiltAndroidApp 生成的代码：**
```java
// 生成的 Hilt_MyApp
public abstract class Hilt_MyApp extends Application implements GeneratedComponentManagerHolder {
    private final ApplicationComponentManager componentManager = 
        new ApplicationComponentManager(/* component builder */);
    
    @Override
    public void onCreate() {
        // 初始化 Dagger Component
        ((ApplicationComponentManager) componentManager).inject(this);
        super.onCreate();
    }
}
```

**@Inject 构造函数注入原理：**
```kotlin
// 源码
class UserRepository @Inject constructor(
    private val api: UserApi,
    private val db: UserDao
)

// 生成的 Factory
class UserRepository_Factory(
    private val apiProvider: Provider<UserApi>,
    private val dbProvider: Provider<UserDao>
) : Factory<UserRepository> {
    override fun get(): UserRepository {
        return UserRepository(apiProvider.get(), dbProvider.get())
    }
}
```

**@Module + @Provides 原理：**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides @Singleton
    fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder().build()
}

// 生成的 Module 方法被 Component 调用
// Component 内部维护 Provider 的单例缓存（DoubleCheck）
```

**Scoping 实现：**
- `@Singleton` → DoubleCheck 包装 Provider，保证单例
- 无 Scope → 每次注入创建新实例
- `@ActivityScoped` → 绑定到 ActivityComponent 生命周期

### 实战层

**ViewModel 注入：**
```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel()

// Activity 中使用
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    private val viewModel: MainViewModel by viewModels()
}
```

**常见 Scope 选择：**
- 全局单例（网络客户端、数据库）→ `@Singleton` + `SingletonComponent`
- Activity 级别（Presenter）→ `@ActivityScoped` + `ActivityComponent`
- ViewModel 级别 → `@ViewModelScoped` + `ViewModelComponent`

**测试替换：**
```kotlin
@HiltAndroidTest
class MainActivityTest {
    @get:Rule val hiltRule = HiltAndroidRule(this)
    
    @BindValue  // 替换真实实现
    val fakeRepo: UserRepository = FakeUserRepository()
}
```

**坑点：**
- `@AndroidEntryPoint` 的 Activity/Fragment 必须继承自 ComponentActivity/Fragment
- Hilt 不支持注入到非 Hilt 管理的类（需要 `@EntryPoint`）
- 多模块项目中 `@InstallIn` 的 Component 必须在 app 模块可见

### 延伸问题

- [[KSP与KAPT对比]]
- [[ViewModel存活原理]]
- [[Clean Architecture在Android的落地]]

## 记忆锚点

Hilt = Dagger 的 Android 自动挡。编译时生成 Factory 和 Component，Gradle 插件改字节码让 Application/Activity 自动持有 Component。你只管 @Inject，它帮你接线。
