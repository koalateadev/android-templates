# Navigation 3 Setup

## Overview

**⚠️ Experimental:** Navigation 3 is currently in alpha. The APIs may change in future releases.

Navigation 3 is a new experimental navigation library designed specifically for Compose that provides full control over your back stack. Unlike traditional navigation, it treats navigation as simply adding and removing items from a list, giving you complete flexibility.

According to the [official documentation](https://developer.android.com/guide/navigation/navigation-3), Navigation 3 offers:
- Full control over your back stack
- Simpler Compose integration
- Adaptive layout support (multiple destinations displayed simultaneously)
- Automatic UI updates with back stack changes
- State retention while items are in the back stack

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
navigation3 = "3.0.0-alpha01"  # Check for latest alpha version

[libraries]
androidx-navigation3 = { group = "androidx.navigation", name = "navigation-compose-experimental", version.ref = "navigation3" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.androidx.navigation3)
    
    // Required: Compose
    implementation(libs.androidx.compose.runtime)
    implementation(libs.androidx.compose.ui)
}
```

### 3. Enable Experimental APIs

Since Navigation 3 is experimental, you may need to opt-in:

```kotlin
@OptIn(ExperimentalNavigationApi::class)
```

## Core Concepts

Navigation 3 is built on three main concepts:

1. **Keys**: Unique identifiers for destinations
2. **Back Stack**: A list of keys representing navigation history
3. **NavDisplay**: UI component that displays the back stack

## Basic Implementation

### 1. Define Navigation Keys

```kotlin
import kotlinx.serialization.Serializable

@Serializable
sealed interface AppDestination {
    @Serializable
    data object Home : AppDestination
    
    @Serializable
    data class Profile(val userId: String) : AppDestination
    
    @Serializable
    data class Details(val itemId: String) : AppDestination
    
    @Serializable
    data class Settings : AppDestination
}
```

### 2. Create Content Resolver

Define how each key maps to actual UI content:

```kotlin
@Composable
fun ResolveDestination(destination: AppDestination) {
    when (destination) {
        is AppDestination.Home -> HomeScreen()
        is AppDestination.Profile -> ProfileScreen(userId = destination.userId)
        is AppDestination.Details -> DetailsScreen(itemId = destination.itemId)
        is AppDestination.Settings -> SettingsScreen()
    }
}
```

### 3. Create Back Stack

```kotlin
import androidx.compose.runtime.mutableStateListOf
import androidx.compose.runtime.remember

@Composable
fun AppNavigation() {
    // Create a mutable back stack
    val backStack = remember {
        mutableStateListOf<AppDestination>(AppDestination.Home)
    }
    
    // Navigate by adding to back stack
    fun navigate(destination: AppDestination) {
        backStack.add(destination)
    }
    
    // Go back by removing from back stack
    fun navigateBack() {
        if (backStack.size > 1) {
            backStack.removeLast()
        }
    }
    
    // Use NavDisplay to show content
    NavDisplay(
        backStack = backStack,
        onNavigateBack = ::navigateBack
    ) { destination ->
        ResolveDestination(destination)
    }
}
```

### 4. Basic NavDisplay Setup

```kotlin
@Composable
fun NavDisplay(
    backStack: List<AppDestination>,
    onNavigateBack: () -> Unit,
    content: @Composable (AppDestination) -> Unit
) {
    // Get the current destination (top of stack)
    val currentDestination = backStack.lastOrNull() ?: return
    
    // Handle back press
    BackHandler(enabled = backStack.size > 1) {
        onNavigateBack()
    }
    
    // Display content
    Box(modifier = Modifier.fillMaxSize()) {
        content(currentDestination)
    }
}
```

## Complete Example

### ViewModel with Back Stack Management

```kotlin
import androidx.lifecycle.ViewModel
import androidx.compose.runtime.mutableStateListOf
import androidx.compose.runtime.snapshots.SnapshotStateList

class NavigationViewModel : ViewModel() {
    private val _backStack = mutableStateListOf<AppDestination>(AppDestination.Home)
    val backStack: SnapshotStateList<AppDestination> = _backStack
    
    fun navigate(destination: AppDestination) {
        _backStack.add(destination)
    }
    
    fun navigateBack() {
        if (_backStack.size > 1) {
            _backStack.removeLast()
        }
    }
    
    fun navigateAndClear(destination: AppDestination) {
        _backStack.clear()
        _backStack.add(destination)
    }
    
    fun popUpTo(destination: AppDestination, inclusive: Boolean = false) {
        val index = _backStack.indexOfLast { it == destination }
        if (index != -1) {
            val removeCount = _backStack.size - index - (if (inclusive) 0 else 1)
            repeat(removeCount) {
                if (_backStack.size > 1) {
                    _backStack.removeLast()
                }
            }
        }
    }
}
```

### Main Navigation Setup

```kotlin
@Composable
fun App() {
    val navViewModel: NavigationViewModel = viewModel()
    val backStack = navViewModel.backStack
    
    NavDisplay(
        backStack = backStack,
        onNavigateBack = { navViewModel.navigateBack() }
    ) { destination ->
        when (destination) {
            is AppDestination.Home -> {
                HomeScreen(
                    onNavigateToProfile = { userId ->
                        navViewModel.navigate(AppDestination.Profile(userId))
                    },
                    onNavigateToSettings = {
                        navViewModel.navigate(AppDestination.Settings)
                    }
                )
            }
            
            is AppDestination.Profile -> {
                ProfileScreen(
                    userId = destination.userId,
                    onNavigateToDetails = { itemId ->
                        navViewModel.navigate(AppDestination.Details(itemId))
                    },
                    onNavigateBack = { navViewModel.navigateBack() }
                )
            }
            
            is AppDestination.Details -> {
                DetailsScreen(
                    itemId = destination.itemId,
                    onNavigateBack = { navViewModel.navigateBack() }
                )
            }
            
            is AppDestination.Settings -> {
                SettingsScreen(
                    onNavigateBack = { navViewModel.navigateBack() }
                )
            }
        }
    }
}
```

## Navigation Patterns

### Pop to Root

```kotlin
fun navigateToHomeAndClearStack() {
    backStack.clear()
    backStack.add(AppDestination.Home)
}
```

### Replace Current Screen

```kotlin
fun replaceCurrent(destination: AppDestination) {
    if (backStack.isNotEmpty()) {
        backStack.removeLast()
    }
    backStack.add(destination)
}
```

### Pop Multiple Screens

```kotlin
fun popMultiple(count: Int) {
    repeat(count) {
        if (backStack.size > 1) {
            backStack.removeLast()
        }
    }
}
```

### Check if Can Go Back

```kotlin
val canGoBack = backStack.size > 1
```

## Adaptive Layouts

Navigation 3 supports showing multiple destinations simultaneously:

```kotlin
@Composable
fun AdaptiveNavDisplay(
    backStack: List<AppDestination>,
    windowSizeClass: WindowSizeClass
) {
    when (windowSizeClass) {
        WindowSizeClass.Compact -> {
            // Phone: Show one destination
            SinglePaneLayout(backStack.lastOrNull())
        }
        
        WindowSizeClass.Medium, WindowSizeClass.Expanded -> {
            // Tablet: Show multiple destinations
            TwoPaneLayout(
                primary = backStack.getOrNull(backStack.size - 2),
                secondary = backStack.lastOrNull()
            )
        }
    }
}
```

## Transitions and Animations

### Basic Slide Transitions

```kotlin
import androidx.compose.animation.*
import androidx.compose.animation.core.tween

@Composable
fun AnimatedNavDisplay(
    backStack: List<AppDestination>,
    content: @Composable (AppDestination) -> Unit
) {
    val currentDestination = backStack.lastOrNull() ?: return
    
    AnimatedContent(
        targetState = currentDestination,
        transitionSpec = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            ) togetherWith slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        }
    ) { destination ->
        content(destination)
    }
}
```

## Deep Links

### Handle Deep Links

```kotlin
@Composable
fun HandleDeepLink(uri: Uri?, backStack: SnapshotStateList<AppDestination>) {
    LaunchedEffect(uri) {
        uri?.let {
            when (it.path) {
                "/profile" -> {
                    val userId = it.getQueryParameter("userId")
                    userId?.let { id ->
                        backStack.clear()
                        backStack.add(AppDestination.Home)
                        backStack.add(AppDestination.Profile(id))
                    }
                }
                "/details" -> {
                    val itemId = it.getQueryParameter("itemId")
                    itemId?.let { id ->
                        backStack.clear()
                        backStack.add(AppDestination.Home)
                        backStack.add(AppDestination.Details(id))
                    }
                }
            }
        }
    }
}
```

## Save and Restore Back Stack

### Save State

```kotlin
import androidx.compose.runtime.saveable.listSaver

val backStackSaver = listSaver<SnapshotStateList<AppDestination>, AppDestination>(
    save = { it.toList() },
    restore = { mutableStateListOf(*it.toTypedArray()) }
)

@Composable
fun rememberBackStack(): SnapshotStateList<AppDestination> {
    return rememberSaveable(saver = backStackSaver) {
        mutableStateListOf(AppDestination.Home)
    }
}
```

## Comparison with Jetpack Navigation

### Navigation 3 Advantages

✅ **Full Control**: Direct manipulation of back stack as a list  
✅ **Simpler API**: No NavHost, NavController complexity  
✅ **Adaptive Layouts**: Easy to show multiple destinations  
✅ **Compose-First**: Designed specifically for Compose  
✅ **Predictable**: Back stack is just a mutable list

### When to Use Jetpack Navigation

- Production apps (Navigation 3 is experimental)
- Need stability and long-term support
- Using Navigation with Fragments
- Require Safe Args plugin

### When to Use Navigation 3

- Experimental projects
- Need full back stack control
- Building adaptive UI layouts
- Want simpler Compose integration

## Migration from Jetpack Navigation

### Before (Jetpack Navigation)

```kotlin
val navController = rememberNavController()
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen() }
    composable("profile/{userId}") { backStackEntry ->
        val userId = backStackEntry.arguments?.getString("userId")
        ProfileScreen(userId)
    }
}
navController.navigate("profile/$userId")
```

### After (Navigation 3)

```kotlin
val backStack = remember { mutableStateListOf(AppDestination.Home) }
NavDisplay(backStack) { destination ->
    when (destination) {
        is AppDestination.Home -> HomeScreen()
        is AppDestination.Profile -> ProfileScreen(destination.userId)
    }
}
backStack.add(AppDestination.Profile(userId))
```

## Testing

### Test Back Stack Navigation

```kotlin
@Test
fun testNavigation() {
    val backStack = mutableStateListOf<AppDestination>(AppDestination.Home)
    
    // Navigate
    backStack.add(AppDestination.Profile("user123"))
    
    assertEquals(2, backStack.size)
    assertEquals(AppDestination.Profile("user123"), backStack.last())
    
    // Go back
    backStack.removeLast()
    
    assertEquals(1, backStack.size)
    assertEquals(AppDestination.Home, backStack.last())
}
```

## Best Practices

1. ✅ Keep back stack in ViewModel for proper lifecycle handling
2. ✅ Use sealed interfaces for type-safe destinations
3. ✅ Save back stack state for configuration changes
4. ✅ Validate back stack operations (don't pop empty stack)
5. ✅ Consider adaptive layouts for tablets
6. ✅ Use animations for better UX
7. ✅ Handle back press properly
8. ✅ Test back stack logic thoroughly
9. ✅ Document navigation flows
10. ✅ Monitor API changes (it's experimental!)

## Known Limitations

⚠️ **Experimental API**: APIs will change  
⚠️ **Limited Documentation**: Community resources are sparse  
⚠️ **No Safe Args**: Type safety through Kotlin Serialization only  
⚠️ **Breaking Changes**: Expect breaking changes between alpha versions

## Resources

- [Official Navigation 3 Documentation](https://developer.android.com/guide/navigation/navigation-3)
- [Navigation 3 Source Code (AOSP)](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:navigation/)
- [Navigation 3 Recipes Repository](https://github.com/android/compose-samples)
- [Navigation 3 Blog Post](https://medium.com/androiddevelopers) (Check Android Developers blog)

## Feedback

Since Navigation 3 is experimental, feedback is important:
- [Report issues on the Issue Tracker](https://issuetracker.google.com/issues?q=componentid:409828)
- Share feedback with the Android team
- Contribute to community discussions

---

**Note**: This guide is based on Navigation 3 alpha. Always check the [official documentation](https://developer.android.com/guide/navigation/navigation-3) for the latest updates and API changes.

