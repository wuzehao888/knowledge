# Compose Navigation 2 完全指南

## 一、Compose Navigation 2 是什么？

Compose Navigation 2 是专门为 Jetpack Compose 设计的导航库，它帮助开发者在 Compose 应用中管理页面跳转和导航流程。

简单来说，它就像 Compose 世界里的"导航地图"：
- 告诉你**有哪些屏幕**（目的地）
- 告诉你**如何在屏幕间移动**（导航路径）
- 帮你**处理跳转逻辑**（导航操作）
- 记住**你去过哪里**（返回栈管理）

## 二、为什么需要 Compose Navigation 2？

### Compose 的特殊性

Compose 是**声明式 UI**框架，与传统的 XML 布局完全不同：
- 没有 Fragment 的概念，而是由 Composable 函数构建 UI
- UI 是状态驱动的，而不是命令式的
- 重组（recomposition）机制使得 UI 更新更加频繁

### 传统导航方案的问题

在 Compose 中使用传统的 Navigation Component 会遇到以下问题：
1. **Fragment 依赖**：传统 Navigation 强依赖 Fragment，而 Compose 不需要 Fragment
2. **声明式不匹配**：传统 Navigation 是命令式的，与 Compose 的声明式风格不符
3. **重组问题**：传统 Navigation 可能在 Compose 重组时产生意外行为

### Compose Navigation 2 的优势

1. **完全适配 Compose**：
   - 基于 Composable 函数设计
   - 支持 Compose 的声明式语法
   - 与重组机制完美配合

2. **简化导航逻辑**：
   - 集中式导航配置
   - 自动返回栈管理
   - 类型安全的参数传递

3. **功能强大**：
   - 支持深层链接
   - 支持导航动画
   - 支持嵌套导航图

4. **与 Compose 生态集成**：
   - 与 Material 3 组件无缝配合
   - 支持 BottomNavigation、Drawer 等常见模式

## 三、Compose Navigation 2 的核心机制

### 1. 核心概念

| 概念 | 大白话解释 | 作用 |
|------|-----------|------|
| 导航图 (NavGraph) | 应用的"地图"，显示所有屏幕和它们之间的连接 | 集中管理所有导航路径 |
| 目的地 (Destination) | 应用中的一个屏幕（Composable 函数） | 导航的起点和终点 |
| 导航控制器 (NavController) | 导航的"驾驶员"，负责执行导航操作 | 处理实际的屏幕跳转逻辑 |
| 导航宿主 (NavHost) | 屏幕的"容器"，显示当前导航到的目的地 | 作为 Composable 函数的容器 |
| 类型安全路由 | 安全的"导航路径"，确保参数正确传递 | 类型安全地定义路由和参数 |

### 2. 工作流程

**Compose Navigation 2 的工作流程可以概括为：**

1. **创建导航图**：定义所有目的地和它们之间的连接
2. **创建导航控制器**：负责管理导航状态和执行导航操作
3. **创建导航宿主**：显示当前目的地的 Composable 函数
4. **执行导航操作**：通过导航控制器在目的地之间跳转

### 3. 导航图结构

在 Compose Navigation 中，导航图是通过 Kotlin 代码（而不是 XML）定义的：

```kotlin
// 定义导航图
fun NavGraphBuilder.appNavGraph(navController: NavController) {
    composable(route = Screen.Home.route) {
        HomeScreen(navController = navController)
    }
    
    composable(
        route = Screen.Detail.route,
        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
    ) {backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId")
        DetailScreen(navController = navController, itemId = itemId)
    }
}

// 定义屏幕路由
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Detail : Screen("detail/{itemId}")
}
```

## 四、Compose Navigation 2 的应用场景

### 1. 单 Activity + Compose 架构

**最适合的场景**：现代 Compose 应用的标准架构，整个应用只有一个 Activity，所有 UI 都由 Compose 构建。

**优势**：
- 完全摆脱 Fragment 的限制
- 充分利用 Compose 的声明式优势
- 简化应用架构

### 2. 多屏幕应用

**适合的场景**：
- 需要多个屏幕的应用（如首页、详情页、设置页等）
- 需要复杂导航逻辑的应用（如多级嵌套导航）
- 需要底部导航栏、侧边栏的应用

### 3. 与其他 Compose 组件配合

**适合的场景**：
- 与 Material 3 组件（如 BottomNavigation、Drawer）配合
- 与 ViewModel 配合，在屏幕跳转时保持数据
- 与 Hilt 配合，实现依赖注入

### 4. 不适合的场景

- **极其简单的应用**：如果应用只有一两个屏幕，可能不需要导航库
- **需要高度自定义导航逻辑**：如果你的导航逻辑非常特殊，可能会觉得 Navigation 的约束太多

## 五、如何使用 Compose Navigation 2？

### 1. 准备工作

**步骤 1：添加依赖**

在 app 模块的 build.gradle 文件中添加以下依赖：

```gradle
dependencies {
    // Compose Navigation 核心依赖
    implementation "androidx.navigation:navigation-compose:2.7.7"
}
```

### 2. 基本使用

**步骤 1：定义屏幕路由**

```kotlin
// 定义屏幕路由
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Detail : Screen("detail/{itemId}")
    
    // 为带参数的路由创建构造函数
    fun createRoute(itemId: String) = "detail/$itemId"
}
```

**步骤 2：设置导航宿主**

在主 Activity 中设置导航宿主：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyApplicationTheme {
                // 创建导航控制器
                val navController = rememberNavController()
                
                // 创建导航宿主
                NavHost(
                    navController = navController,
                    startDestination = Screen.Home.route
                ) {
                    // 定义导航图
                    composable(Screen.Home.route) {
                        HomeScreen(navController = navController)
                    }
                    composable(
                        route = Screen.Detail.route,
                        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
                    ) {backStackEntry ->
                        val itemId = backStackEntry.arguments?.getString("itemId")
                        DetailScreen(navController = navController, itemId = itemId)
                    }
                }
            }
        }
    }
}
```

**步骤 3：在 Composable 中导航**

```kotlin
@Composable
fun HomeScreen(navController: NavController) {
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Home Screen")
        Button(
            onClick = {
                // 导航到详情页
                navController.navigate(Screen.Detail.createRoute("item123"))
            }
        ) {
            Text(text = "Go to Detail")
        }
    }
}

@Composable
fun DetailScreen(navController: NavController, itemId: String?) {
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Detail Screen")
        Text(text = "Item ID: $itemId")
        Button(
            onClick = {
                // 返回上一页
                navController.popBackStack()
            }
        ) {
            Text(text = "Go Back")
        }
    }
}
```

### 3. 高级功能

#### 类型安全的参数传递

在 Compose 项目中，**不推荐使用 nav_graph.xml 文件**，因为它与 Compose 的声明式理念不匹配。相反，我们可以通过以下方式实现类型安全的参数传递：

**方法 1：使用密封类和扩展函数**

```kotlin
// 1. 定义路由密封类
sealed class Screen(val route: String) {
    object Home : Screen("home")
    data class Detail(val itemId: String) : Screen("detail/{itemId}") {
        // 生成带参数的路由
        fun createRoute() = "detail/$itemId"
    }
}

// 2. 定义导航扩展函数
fun NavController.navigate(screen: Screen) {
    when (screen) {
        is Screen.Home -> navigate(screen.route)
        is Screen.Detail -> navigate(screen.createRoute())
    }
}

// 3. 在导航图中定义
fun NavGraphBuilder.appNavGraph() {
    composable(Screen.Home.route) {
        HomeScreen()
    }
    composable(
        route = "detail/{itemId}",
        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
    ) { backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        DetailScreen(itemId = itemId)
    }
}

// 4. 使用方式
@Composable
fun HomeScreen() {
    val navController = rememberNavController()
    
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Home Screen")
        Button(
            onClick = {
                navController.navigate(Screen.Detail(itemId = "123"))
            }
        ) {
            Text(text = "Go to Detail")
        }
    }
}

@Composable
fun DetailScreen(itemId: String) {
    val navController = rememberNavController()
    
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Detail Screen")
        Text(text = "Item ID: $itemId")
        Button(
            onClick = {
                navController.popBackStack()
            }
        ) {
            Text(text = "Go Back")
        }
    }
}
```

**方法 2：使用类型安全的导航助手**

```kotlin
// 1. 定义导航目标接口
interface NavDestination {
    val route: String
}

// 2. 定义具体目标
object HomeDestination : NavDestination {
    override val route = "home"
}

data class DetailDestination(val itemId: String) : NavDestination {
    override val route = "detail/$itemId"
    
    companion object {
        const val ROUTE_PATTERN = "detail/{itemId}"
    }
}

// 3. 导航扩展函数
fun NavController.navigate(destination: NavDestination) {
    navigate(destination.route)
}

// 4. 在导航图中使用
fun NavGraphBuilder.appNavGraph() {
    composable(HomeDestination.route) {
        HomeScreen()
    }
    composable(
        route = DetailDestination.ROUTE_PATTERN,
        arguments = listOf(navArgument("itemId") { type = NavType.StringType })
    ) {
        backStackEntry ->
        val itemId = backStackEntry.arguments?.getString("itemId") ?: ""
        DetailScreen(itemId = itemId)
    }
}
```

**为什么不推荐使用 nav_graph.xml？**

1. **与 Compose 风格不符**：Compose 是声明式 UI，所有逻辑都应该在 Kotlin 代码中
2. **代码更加简洁**：使用 `NavGraphBuilder` 在代码中定义导航图更加直观
3. **避免额外的构建步骤**：不需要通过 Safe Args 插件生成代码
4. **更好的类型安全**：可以直接使用 Kotlin 的类型系统确保安全

通过使用密封类、扩展函数等 Kotlin 语言特性，我们可以在不使用 XML 的情况下，实现更加类型安全、更加优雅的导航方案。

#### 与 Bottom Navigation 集成

**步骤 1：定义底部导航项**

```kotlin
sealed class BottomNavItem(val route: String, val label: String, val icon: Int) {
    object Home : BottomNavItem("home", "Home", R.drawable.ic_home)
    object Explore : BottomNavItem("explore", "Explore", R.drawable.ic_explore)
    object Profile : BottomNavItem("profile", "Profile", R.drawable.ic_profile)
}
```

**步骤 2：创建底部导航**

```kotlin
@Composable
fun BottomNavigationBar(
    navController: NavController,
    items: List<BottomNavItem>,
    modifier: Modifier = Modifier
) {
    NavigationBar(modifier = modifier) {
        items.forEach {item ->
            NavigationBarItem(
                label = { Text(item.label) },
                icon = { Icon(painterResource(item.icon), contentDescription = item.label) },
                selected = currentRoute(navController) == item.route,
                onClick = {
                    if (currentRoute(navController) != item.route) {
                        navController.navigate(item.route) {
                            // 避免重复添加到返回栈
                            launchSingleTop = true
                            // 点击时重置返回栈
                            popUpTo(navController.graph.findStartDestination().id) {
                                saveState = true
                            }
                            // 恢复保存的状态
                            restoreState = true
                        }
                    }
                }
            )
        }
    }
}

// 辅助函数：获取当前路由
@Composable
private fun currentRoute(navController: NavController): String? {
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    return navBackStackEntry?.destination?.route
}
```

**步骤 3：集成到主布局**

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    
    Scaffold(
        bottomBar = {
            BottomNavigationBar(
                navController = navController,
                items = listOf(
                    BottomNavItem.Home,
                    BottomNavItem.Explore,
                    BottomNavItem.Profile
                )
            )
        }
    ) {innerPadding ->
        NavHost(
            navController = navController,
            startDestination = BottomNavItem.Home.route,
            modifier = Modifier.padding(innerPadding)
        ) {
            composable(BottomNavItem.Home.route) {
                HomeScreen(navController = navController)
            }
            composable(BottomNavItem.Explore.route) {
                ExploreScreen(navController = navController)
            }
            composable(BottomNavItem.Profile.route) {
                ProfileScreen(navController = navController)
            }
        }
    }
}
```

## 六、Compose Navigation 2 核心流程图

### 1. 基本导航流程

```
+----------------+      +----------------+      +----------------+
|                |      |                |      |                |
|  用户点击按钮   +----->+  NavController  +----->+  目标屏幕      |
|                |      |  执行导航操作   |      |                |
+----------------+      +----------------+      +----------------+
                            |
                            v
                    +----------------+
                    |                |
                    | 更新返回栈     |
                    | 重组目标屏幕   |
                    +----------------+
```

### 2. 与 Bottom Navigation 集成流程

```
+----------------+      +----------------+      +----------------+
|                |      |                |      |                |
|  用户点击底部   +----->+  NavigationBar +----->+  NavController  |
|  导航项        |      |  触发导航事件  |      |  执行屏幕切换  |
|                |      |                |      |                |
+----------------+      +----------------+      +----------------+
                            |
                            v
                    +----------------+
                    |                |
                    | 更新选中状态   |
                    | 恢复屏幕状态   |
                    +----------------+
```

### 3. 导航图结构流程

```
+----------------+      +----------------+      +----------------+
|                |      |                |      |                |
|  导航图定义    +----->+  NavHost      +----->+  Composable     |
|  (NavGraphBuilder) |  (导航容器)     |  (屏幕实现)     |
|                |      |                |      |                |
+----------------+      +----------------+      +----------------+
```

## 七、常见问题与解决方案

### 1. 导航时重组问题

**问题**：导航操作可能在 Compose 重组时被重复执行。

**解决方案**：使用 `LaunchedEffect` 或确保导航操作只在用户交互时执行：

```kotlin
// 正确的做法：在按钮点击时导航
Button(
    onClick = {
        navController.navigate(Screen.Detail.createRoute("item123"))
    }
) {
    Text(text = "Go to Detail")
}

// 错误的做法：在 Composable 函数体中直接导航
// 这会在每次重组时执行导航
if (someCondition) {
    navController.navigate(Screen.Detail.createRoute("item123"))
}

// 正确的做法：使用 LaunchedEffect
LaunchedEffect(someCondition) {
    if (someCondition) {
        navController.navigate(Screen.Detail.createRoute("item123"))
    }
}
```

### 2. 底部导航栏状态保存

**问题**：切换底部导航项时，屏幕状态丢失。

**解决方案**：在导航时设置 `saveState` 和 `restoreState`：

```kotlin
navController.navigate(item.route) {
    launchSingleTop = true
    popUpTo(navController.graph.findStartDestination().id) {
        saveState = true
    }
    restoreState = true
}
```

### 3. 处理返回按钮

**问题**：需要自定义返回按钮行为。

**解决方案**：使用 `BackHandler` Composable：

```kotlin
@Composable
fun DetailScreen(navController: NavController, itemId: String?) {
    // 自定义返回按钮行为
    BackHandler {
        // 执行自定义逻辑
        println("Back button pressed")
        // 然后返回
        navController.popBackStack()
    }
    
    // 屏幕内容
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(text = "Detail Screen")
        Text(text = "Item ID: $itemId")
        Button(
            onClick = {
                navController.popBackStack()
            }
        ) {
            Text(text = "Go Back")
        }
    }
}
```

## 八、最佳实践

### 1. 导航图设计

- **集中管理导航逻辑**：将导航图定义放在单独的文件中
- **使用密封类定义路由**：避免字符串硬编码，提高类型安全性
- **合理组织导航结构**：对于大型应用，使用嵌套导航图

### 2. 参数传递

- **使用密封类定义路由**：避免字符串硬编码，提高类型安全性
- **传递最小必要参数**：只传递标识符，大型对象通过 ViewModel 共享
- **使用默认值**：为可选参数设置默认值
- **避免使用 nav_graph.xml**：与 Compose 风格不符，推荐使用纯代码方式

### 3. 性能优化

- **使用 `launchSingleTop`**：避免在导航到相同目的地时创建新实例
- **合理使用 `popUpTo`**：在适当的时候清理返回栈
- **避免在重组时执行导航**：使用 `LaunchedEffect` 或用户交互触发导航

### 4. 用户体验

- **添加转场动画**：使用 `navController.navigate()` 的 `enterTransition` 和 `exitTransition` 参数
- **处理加载状态**：在导航过程中显示加载指示器
- **提供清晰的返回路径**：确保用户始终知道如何返回上一页

## 九、总结

Compose Navigation 2 是为 Jetpack Compose 量身定制的导航解决方案，它通过以下方式改善了 Compose 应用的导航体验：

1. **完全适配 Compose**：与 Compose 的声明式风格完美配合
2. **简化导航逻辑**：集中管理导航配置，自动处理返回栈
3. **提高代码质量**：类型安全的参数传递，减少运行时错误
4. **增强用户体验**：支持转场动画，提供一致的导航行为
5. **与生态系统集成**：与 Material 3、ViewModel 等组件无缝配合

虽然 Compose Navigation 2 有一些学习曲线，但一旦掌握，它会成为你开发 Compose 应用的得力助手，帮助你构建更加专业、可靠的导航系统。

---

**小贴士**：
- 开始使用 Compose Navigation 2 时，建议从小项目或新功能开始尝试
- 参考 Google 的官方示例代码，学习最佳实践
- 结合其他 Jetpack 组件一起使用，发挥最大优势

希望这份指南能帮助你理解和使用 Compose Navigation 2！