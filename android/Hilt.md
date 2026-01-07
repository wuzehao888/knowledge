# Android Hilt 框架完全指南

## 📚 目录

1. [Hilt 简介](#1-hilt-简介)
2. [核心概念](#2-核心概念)
3. [项目配置](#3-项目配置)
4. [依赖注入基础](#4-依赖注入基础)
5. [常用注解详解](#5-常用注解详解)
6. [模块与组件](#6-模块与组件)
7. [作用域管理](#7-作用域管理)
8. [ViewModel 集成](#8-viewmodel-集成)
9. [测试支持](#9-测试支持)
10. [最佳实践](#10-最佳实践)
11. [实战案例](#11-实战案例)
12. [常见问题](#12-常见问题)

---

## 1. Hilt 简介

### 1.1 什么是 Hilt

Hilt 是 Google 为 Android 构建的依赖注入库，基于 Dagger 构建，专门针对 Android 平台进行了优化。

**核心特性：**
- ✅ 基于 Dagger 2，提供编译时验证
- ✅ 专门为 Android 优化
- ✅ 简化依赖注入配置
- ✅ 集成 Android Jetpack 组件
- ✅ 支持 ViewModel、WorkManager 等

### 1.2 为什么使用 Hilt

**传统依赖注入的问题：**
```kotlin
// ❌ 手动依赖管理
class UserRepository {
    private val api = ApiClient()
    private val db = Database()
}
```

**Hilt 的优势：**
```kotlin
// ✅ 自动依赖注入
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var userRepository: UserRepository
}
```

### 1.3 Hilt vs Dagger

| 特性 | Dagger | Hilt |
|------|---------|-------|
| 学习曲线 | 陡峭 | 平缓 |
| Android 集成 | 需要手动配置 | 自动集成 |
| ViewModel 支持 | 需要额外配置 | 原生支持 |
| 代码量 | 多 | 少 |
| 适用场景 | 通用 | Android 专用 |

---

## 2. 核心概念

### 2.1 依赖注入（DI）

依赖注入是一种设计模式，通过外部容器来管理对象的依赖关系。

**示例：**
```kotlin
// 不使用 DI
class Car {
    private val engine = Engine()  // 紧耦合
}

// 使用 DI
class Car @Inject constructor(
    private val engine: Engine  // 依赖注入
)
```

### 2.2 Hilt 的核心组件

1. **Application 类**：应用的入口点
2. **Module**：提供依赖的模块
3. **Component**：依赖注入的容器
4. **EntryPoint**：注入依赖的入口点
5. **Inject**：标记需要注入的依赖

### 2.3 Hilt 工作流程

```
Application
    ↓
Hilt_Application_Component
    ↓
Modules (提供依赖)
    ↓
EntryPoints (注入依赖)
    ↓
Activities, Fragments, Services 等
```

---

## 3. 项目配置

### 3.1 添加依赖

**在 `build.gradle` (Module level) 中添加：**

```gradle
plugins {
    id('kotlin-kapt')
    id('dagger.hilt.android.plugin')
}

dependencies {
    implementation 'com.google.dagger:hilt-android:2.48'
    kapt 'com.google.dagger:hilt-compiler:2.48'
    
    // ViewModel 支持
    implementation 'androidx.hilt:hilt-lifecycle-viewmodel:1.1.0'
    kapt 'androidx.hilt:hilt-compiler:1.1.0'
}
```

**在 `build.gradle` (Project level) 中添加：**

```gradle
buildscript {
    dependencies {
        classpath 'com.google.dagger:hilt-android-gradle-plugin:2.48'
    }
}
```

### 3.2 创建 Application 类

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt 会自动生成依赖注入代码
}
```

**在 `AndroidManifest.xml` 中注册：**

```xml
<application
    android:name=".MyApplication"
    ...>
</application>
```

---

## 4. 依赖注入基础

### 4.1 构造函数注入

```kotlin
class UserRepository @Inject constructor(
    private val apiService: ApiService,
    private val database: AppDatabase
) {
    // 自动注入 apiService 和 database
}
```

### 4.2 字段注入

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    @Inject lateinit var userRepository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // userRepository 已自动注入
    }
}
```

### 4.3 方法注入

```kotlin
class AnalyticsHelper @Inject constructor() {
    
    @Inject
    fun logEvent(event: String) {
        // 方法注入
    }
}
```

---

## 5. 常用注解详解

### 5.1 @Inject

**用途：** 标记需要注入的依赖

```kotlin
// 构造函数注入
class MyRepository @Inject constructor(
    private val apiService: ApiService
)

// 字段注入
@AndroidEntryPoint
class MyActivity : AppCompatActivity() {
    @Inject lateinit var repository: MyRepository
}
```

### 5.2 @Module

**用途：** 定义提供依赖的模块

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
}
```

### 5.3 @Provides

**用途：** 在 Module 中提供依赖实例

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
}
```

### 5.4 @Binds

**用途：** 接口到实现的绑定

```kotlin
interface UserRepository {
    fun getUser(): User
}

@Module
@InstallIn(SingletonComponent::class)
abstract class UserModule {
    
    @Binds
    abstract fun bindUserRepository(
        userRepositoryImpl: UserRepositoryImpl
    ): UserRepository
}
```

### 5.5 @Qualifier

**用途：** 区分相同类型的依赖

```kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptor

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class LoggingInterceptor

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @AuthInterceptor
    fun provideAuthInterceptor(): Interceptor {
        return AuthInterceptor()
    }
    
    @Provides
    @LoggingInterceptor
    fun provideLoggingInterceptor(): Interceptor {
        return LoggingInterceptor()
    }
}

// 使用
class MyRepository @Inject constructor(
    @AuthInterceptor private val authInterceptor: Interceptor,
    @LoggingInterceptor private val loggingInterceptor: Interceptor
)
```

### 5.6 @Named

**用途：** 使用字符串区分依赖

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ConfigModule {
    
    @Provides
    @Named("api_key")
    fun provideApiKey(): String {
        return "your_api_key"
    }
    
    @Provides
    @Named("base_url")
    fun provideBaseUrl(): String {
        return "https://api.example.com"
    }
}

// 使用
class MyRepository @Inject constructor(
    @Named("api_key") private val apiKey: String,
    @Named("base_url") private val baseUrl: String
)
```

---

## 6. 模块与组件

### 6.1 Hilt 组件层次

```
ApplicationComponent (SingletonComponent)
    ↓
ActivityRetainedComponent
    ↓
ActivityComponent
    ↓
FragmentComponent
    ↓
ViewComponent
    ↓
ViewWithFragmentComponent
    ↓
ServiceComponent
```

### 6.2 常用组件说明

| 组件 | 作用域 | 生命周期 |
|------|--------|---------|
| `SingletonComponent` | `@Singleton` | 整个应用 |
| `ActivityRetainedComponent` | `@ActivityRetainedScoped` | Activity 重建 |
| `ActivityComponent` | `@ActivityScoped` | Activity |
| `FragmentComponent` | `@FragmentScoped` | Fragment |
| `ViewModelComponent` | `@ViewModelScoped` | ViewModel |
| `ServiceComponent` | `@ServiceScoped` | Service |

### 6.3 创建自定义 Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    // 提供 Context
    @Provides
    @ApplicationContext
    fun provideApplicationContext(@ApplicationContext context: Context): Context {
        return context
    }
    
    // 提供 Application Context
    @Provides
    fun provideApplication(application: Application): Application {
        return application
    }
    
    // 提供 SharedPreferences
    @Provides
    @Singleton
    fun provideSharedPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    }
}
```

### 6.4 多 Module 组合

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit = ...
}

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    @Provides
    @Singleton
    fun provideDatabase(): AppDatabase = ...
}

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: ApiService,
        database: AppDatabase
    ): UserRepository = UserRepositoryImpl(apiService, database)
}
```

---

## 7. 作用域管理

### 7.1 @Singleton

**用途：** 整个应用生命周期内单例

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideRepository(): UserRepository {
        return UserRepositoryImpl()
    }
}

// 整个应用中只有一个实例
```

### 7.2 @ActivityScoped

**用途：** Activity 生命周期内单例

```kotlin
@Module
@InstallIn(ActivityComponent::class)
object ActivityModule {
    
    @Provides
    @ActivityScoped
    fun provideActivityData(): ActivityData {
        return ActivityData()
    }
}
```

### 7.3 @FragmentScoped

**用途：** Fragment 生命周期内单例

```kotlin
@Module
@InstallIn(FragmentComponent::class)
object FragmentModule {
    
    @Provides
    @FragmentScoped
    fun provideFragmentData(): FragmentData {
        return FragmentData()
    }
}
```

### 7.4 @ViewModelScoped

**用途：** ViewModel 生命周期内单例

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    // repository 是 @ViewModelScoped，在 ViewModel 重建时保持不变
}
```

### 7.5 作用域对比

| 作用域 | 生命周期 | 适用场景 |
|--------|---------|---------|
| `@Singleton` | 应用生命周期 | 全局单例（数据库、网络） |
| `@ActivityRetainedScoped` | Activity 重建 | 需要在配置更改时保留的数据 |
| `@ActivityScoped` | Activity | Activity 级别的数据 |
| `@FragmentScoped` | Fragment | Fragment 级别的数据 |
| `@ViewModelScoped` | ViewModel | ViewModel 级别的数据 |

---

## 8. ViewModel 集成

### 8.1 创建 Hilt ViewModel

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()
    
    init {
        loadUsers()
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _users.value = userRepository.getUsers()
        }
    }
}
```

### 8.2 在 Activity 中使用 ViewModel

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        lifecycleScope.launch {
            viewModel.users.collect { users ->
                // 更新 UI
            }
        }
    }
}
```

### 8.3 在 Fragment 中使用 ViewModel

```kotlin
@AndroidEntryPoint
class UserFragment : Fragment() {
    
    private val viewModel: UserViewModel by activityViewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.users.collect { users ->
                // 更新 UI
            }
        }
    }
}
```

### 8.4 带 SavedStateHandle 的 ViewModel

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    private val repository: UserRepository,
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    private val userId: String = savedStateHandle["userId"] ?: ""
    
    val user = repository.getUser(userId)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = null
        )
}
```

---

## 9. 测试支持

### 9.1 测试 Hilt 组件

**添加测试依赖：**

```gradle
dependencies {
    testImplementation 'com.google.dagger:hilt-android-testing:2.48'
    kaptTest 'com.google.dagger:hilt-android-compiler:2.48'
    
    androidTestImplementation 'com.google.dagger:hilt-android-testing:2.48'
    kaptAndroidTest 'com.google.dagger:hilt-android-compiler:2.48'
}
```

### 9.2 单元测试

```kotlin
@HiltAndroidTest
class UserRepositoryTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testGetUser() = runTest {
        val user = userRepository.getUser("1")
        assertEquals("John", user.name)
    }
}
```

### 9.3 UI 测试

```kotlin
@HiltAndroidTest
class MainActivityTest {
    
    @get:Rule
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Inject
    lateinit var userRepository: UserRepository
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun testUserDisplay() {
        composeTestRule.setContent {
            UserScreen(userRepository = userRepository)
        }
        
        composeTestRule.onNodeWithText("John").assertIsDisplayed()
    }
}
```

### 9.4 替换依赖进行测试

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class]
)
object TestNetworkModule {
    
    @Provides
    @Singleton
    fun provideTestApiService(): ApiService {
        return FakeApiService()  // 使用假 API
    }
}

@HiltAndroidTest
class UserRepositoryTest {
    
    @BindValue
    @JvmField
    val testRepository: UserRepository = FakeUserRepository()
}
```

---

## 10. 最佳实践

### 10.1 架构分层

```
Presentation Layer (UI)
    ↓ (注入)
Domain Layer (ViewModel)
    ↓ (注入)
Data Layer (Repository)
    ↓ (注入)
Data Sources (API, Database)
```

### 10.2 模块组织

**推荐的项目结构：**

```
di/
├── module/
│   ├── AppModule.kt
│   ├── NetworkModule.kt
│   ├── DatabaseModule.kt
│   └── RepositoryModule.kt
├── qualifier/
│   ├── AuthInterceptor.kt
│   └── LoggingInterceptor.kt
└── component/
    └── ApplicationComponent.kt
```

### 10.3 依赖注入原则

1. **构造函数注入优先**
```kotlin
// ✅ 推荐
class MyRepository @Inject constructor(
    private val api: ApiService
)

// ❌ 避免
class MyRepository {
    @Inject lateinit var api: ApiService
}
```

2. **使用接口而非实现**
```kotlin
// ✅ 推荐
interface UserRepository {
    fun getUser(): User
}

@Module
abstract class UserModule {
    @Binds
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}

// ❌ 避免
class MyRepository @Inject constructor(
    private val impl: UserRepositoryImpl  // 直接依赖实现
)
```

3. **合理使用作用域**
```kotlin
// ✅ 数据库使用 Singleton
@Singleton
class AppDatabase @Inject constructor()

// ✅ ViewModel 使用 ViewModelScoped
@HiltViewModel
class MyViewModel @Inject constructor()

// ✅ Repository 使用 Singleton
@Singleton
class UserRepository @Inject constructor()
```

### 10.4 避免循环依赖

```kotlin
// ❌ 循环依赖
class A @Inject constructor(private val b: B)
class B @Inject constructor(private val a: A)

// ✅ 解决方案：使用接口
interface AInterface
class AImpl @Inject constructor(private val b: B) : AInterface
class B @Inject constructor(private val a: AInterface)
```

---

## 11. 实战案例

### 11.1 完整的 Hilt 项目示例

#### 11.1.1 Application 类

```kotlin
@HiltAndroidApp
class MyApplication : Application() {
    // Hilt 自动生成依赖注入代码
}
```

#### 11.1.2 网络模块

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .addInterceptor(LoggingInterceptor())
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

#### 11.1.3 数据库模块

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        )
            .fallbackToDestructiveMigration()
            .build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}
```

#### 11.1.4 Repository 模块

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: ApiService,
        userDao: UserDao
    ): UserRepository {
        return UserRepositoryImpl(apiService, userDao)
    }
}
```

#### 11.1.5 Repository 实现

```kotlin
class UserRepositoryImpl @Inject constructor(
    private val apiService: ApiService,
    private val userDao: UserDao
) : UserRepository {
    
    override fun getUsers(): Flow<List<User>> {
        return userDao.getAllUsers()
    }
    
    override suspend fun refreshUsers() {
        val users = apiService.getUsers()
        userDao.insertUsers(users)
    }
}
```

#### 11.1.6 ViewModel

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                userRepository.refreshUsers()
                userRepository.getUsers().collect { users ->
                    _uiState.value = UiState.Success(users)
                }
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

sealed class UiState {
    object Loading : UiState()
    data class Success(val users: List<User>) : UiState()
    data class Error(val message: String) : UiState()
}
```

#### 11.1.7 Activity

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    private val viewModel: UserListViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                UserListScreen(viewModel = viewModel)
            }
        }
    }
}
```

#### 11.1.8 Composable

```kotlin
@Composable
fun UserListScreen(viewModel: UserListViewModel) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> {
            CircularProgressIndicator()
        }
        is UiState.Success -> {
            LazyColumn {
                items(uiState.users) { user ->
                    UserItem(user = user)
                }
            }
        }
        is UiState.Error -> {
            ErrorMessage(message = uiState.message)
        }
    }
}
```

### 11.2 WorkManager 集成

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val repository: UserRepository
) : CoroutineWorker(context, workerParams) {
    
    override suspend fun doWork(): Result {
        return try {
            repository.syncData()
            Result.success()
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// 使用
val workRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .build()
WorkManager.getInstance(context).enqueue(workRequest)
```

### 11.3 Navigation 集成

```kotlin
@HiltViewModel
class NavigationViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()

@Composable
fun NavGraph(
    navController: NavHostController,
    startDestination: String = "home"
) {
    val navBackStackEntry by navController.currentBackStackEntryFlow.collectAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable("home") {
            HomeScreen(
                navController = navController
            )
        }
        
        composable(
            route = "detail/{userId}",
            arguments = listOf(navArgument("userId") { type = NavType.StringType })
        ) { backStackEntry ->
            val userId = backStackEntry.arguments?.getString("userId") ?: ""
            DetailScreen(
                userId = userId,
                navController = navController
            )
        }
    }
}
```

---

## 12. 常见问题

### 12.1 编译错误

#### 问题：@AndroidEntryPoint 必须在支持 Hilt 的类上使用

**错误信息：**
```
@AndroidEntryPoint can only be used with context classes that are supported
```

**解决方案：**
```kotlin
// ✅ 支持的类
@AndroidEntryPoint
class MainActivity : AppCompatActivity()

@AndroidEntryPoint
class MyFragment : Fragment()

@AndroidEntryPoint
class MyService : Service()

// ❌ 不支持的类
@AndroidEntryPoint  // 错误
class MyClass
```

#### 问题：Module 没有正确安装

**错误信息：**
```
@Provides methods can only be present within a @Module
```

**解决方案：**
```kotlin
// ✅ 正确
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    fun provideService(): Service = ...
}

// ❌ 错误
object AppModule {
    @Provides
    fun provideService(): Service = ...
}
```

### 12.2 运行时错误

#### 问题：依赖无法注入

**错误信息：**
```
Attempt to request injection from an Activity that does not support Hilt
```

**解决方案：**
```kotlin
// ✅ 添加 @AndroidEntryPoint
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var repository: UserRepository
}
```

#### 问题：循环依赖

**错误信息：**
```
Dependency cycle detected
```

**解决方案：**
```kotlin
// 使用接口打破循环依赖
interface AInterface
class AImpl @Inject constructor(private val b: B) : AInterface
class B @Inject constructor(private val a: AInterface)
```

### 12.3 性能问题

#### 问题：启动速度慢

**解决方案：**
1. 延迟初始化非关键依赖
2. 使用 `@Lazy` 注解
3. 优化 Module 的依赖关系

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideHeavyService(): HeavyService {
        // 延迟初始化
        return HeavyService()
    }
}
```

### 12.4 调试技巧

#### 查看生成的代码

Hilt 会生成依赖注入代码，可以在以下位置查看：

```
app/build/generated/hilt/component_sources/
```

#### 使用 Hilt 图形化工具

```kotlin
// 在 Application 中启用 Hilt 图形化
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            // 启用 Hilt 图形化
            enableHiltGraphVisualization()
        }
    }
}
```

---

## 📚 学习资源

### 官方文档
- [Hilt 官方文档](https://dagger.dev/hilt/)
- [Android Developers - Hilt](https://developer.android.com/training/dependency-injection/hilt-android)

### 推荐文章
- [Hilt 完全指南](https://medium.com/androiddevelopers/hilt-jetpack-new-dependency-injection-on-android-dagger2-hilt-10d3c27f6d9e)
- [Hilt 最佳实践](https://proandroiddev.com/hilt-best-practices-8d2d5e9b7e9a)

### 视频教程
- [Android Hilt 教程](https://www.youtube.com/results?search_query=android+hilt+tutorial)
- [Kotlin Hilt 实战](https://www.youtube.com/results?search_query=kotlin+hilt+android)

---

## 🎯 总结

### Hilt 核心要点

1. ✅ **理解依赖注入**：掌握 DI 的核心概念
2. ✅ **熟悉常用注解**：@Inject、@Module、@Provides 等
3. ✅ **掌握作用域**：合理使用不同作用域
4. ✅ **ViewModel 集成**：使用 @HiltViewModel
5. ✅ **测试支持**：编写可测试的代码
6. ✅ **最佳实践**：遵循架构设计原则

### 学习路线

1. **第1周**：Hilt 基础概念和配置
2. **第2周**：Module 和依赖提供
3. **第3周**：ViewModel 和作用域
4. **第4周**：测试和最佳实践
5. **第5周**：实战项目

### 进阶方向

- 🔧 自定义 Hilt 组件
- 📊 性能优化
- 🧪 高级测试技巧
- 🚀 多模块项目架构

---

## 💡 快速参考

### 常用注解速查

| 注解 | 用途 |
|------|------|
| `@HiltAndroidApp` | 标记 Application 类 |
| `@AndroidEntryPoint` | 标记可注入的类 |
| `@Inject` | 标记需要注入的依赖 |
| `@Module` | 定义依赖提供模块 |
| `@Provides` | 提供依赖实例 |
| `@Binds` | 绑定接口到实现 |
| `@Singleton` | 单例作用域 |
| `@HiltViewModel` | 标记 ViewModel |
| `@Qualifier` | 区分相同类型依赖 |
| `@Named` | 字符串区分依赖 |

### 组件作用域速查

| 组件 | 作用域 | 生命周期 |
|------|--------|---------|
| `SingletonComponent` | `@Singleton` | 应用 |
| `ActivityRetainedComponent` | `@ActivityRetainedScoped` | Activity 重建 |
| `ActivityComponent` | `@ActivityScoped` | Activity |
| `FragmentComponent` | `@FragmentScoped` | Fragment |
| `ViewModelComponent` | `@ViewModelScoped` | ViewModel |

---

按照这份文档学习，您将全面掌握 Android Hilt 框架！🚀
