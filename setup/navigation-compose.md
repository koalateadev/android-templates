# Navigation Compose (Type-Safe Navigation)

## Overview
Navigation Compose with Kotlin Serialization provides type-safe navigation for Jetpack Compose apps with compile-time safety for arguments.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
navigation = "2.8.3"
kotlinx-serialization = "1.7.3"

[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

### 2. Apply Plugin and Add Dependencies

```kotlin
plugins {
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(libs.androidx.navigation.compose)
    implementation(libs.kotlinx.serialization.json)
    
    // Optional: Hilt integration
    implementation(libs.hilt.navigation.compose)
}
```

## Type-Safe Navigation

### 1. Define Routes with Serialization

```kotlin
import kotlinx.serialization.Serializable

@Serializable
object HomeRoute

@Serializable
data class ProfileRoute(val userId: String)

@Serializable
data class DetailsRoute(
    val itemId: String,
    val isEditable: Boolean = false
)

@Serializable
data class SearchRoute(
    val query: String = "",
    val categoryId: Int? = null
)
```

### 2. Setup NavHost

```kotlin
import androidx.compose.runtime.Composable
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.toRoute

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToProfile = { userId ->
                    navController.navigate(ProfileRoute(userId))
                },
                onNavigateToDetails = { itemId, editable ->
                    navController.navigate(DetailsRoute(itemId, editable))
                }
            )
        }
        
        composable<ProfileRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<ProfileRoute>()
            ProfileScreen(
                userId = route.userId,
                onNavigateBack = { navController.navigateUp() }
            )
        }
        
        composable<DetailsRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<DetailsRoute>()
            DetailsScreen(
                itemId = route.itemId,
                isEditable = route.isEditable,
                onNavigateBack = { navController.navigateUp() }
            )
        }
        
        composable<SearchRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<SearchRoute>()
            SearchScreen(
                initialQuery = route.query,
                categoryId = route.categoryId
            )
        }
    }
}
```

### 3. Navigate Between Screens

```kotlin
@Composable
fun HomeScreen(
    onNavigateToProfile: (String) -> Unit,
    onNavigateToDetails: (String, Boolean) -> Unit
) {
    Column {
        Button(onClick = { onNavigateToProfile("user123") }) {
            Text("Go to Profile")
        }
        
        Button(onClick = { onNavigateToDetails("item456", true) }) {
            Text("Edit Details")
        }
    }
}
```

## Nested Navigation

### Define Nested Graph

```kotlin
@Serializable
object AuthGraph

@Serializable
object LoginRoute

@Serializable
object RegisterRoute

@Serializable
object ForgotPasswordRoute
```

### Setup Nested Navigation

```kotlin
import androidx.navigation.navigation

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = HomeRoute
    ) {
        composable<HomeRoute> {
            HomeScreen()
        }
        
        navigation<AuthGraph>(startDestination = LoginRoute) {
            composable<LoginRoute> {
                LoginScreen(
                    onNavigateToRegister = {
                        navController.navigate(RegisterRoute)
                    },
                    onNavigateToForgotPassword = {
                        navController.navigate(ForgotPasswordRoute)
                    },
                    onLoginSuccess = {
                        navController.navigate(HomeRoute) {
                            popUpTo<AuthGraph> { inclusive = true }
                        }
                    }
                )
            }
            
            composable<RegisterRoute> {
                RegisterScreen(
                    onNavigateBack = { navController.navigateUp() }
                )
            }
            
            composable<ForgotPasswordRoute> {
                ForgotPasswordScreen(
                    onNavigateBack = { navController.navigateUp() }
                )
            }
        }
    }
}
```

## Bottom Navigation

### Define Bottom Nav Routes

```kotlin
@Serializable
object HomeTab

@Serializable
object SearchTab

@Serializable
object NotificationsTab

@Serializable
object ProfileTab
```

### Bottom Navigation Setup

```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.getValue
import androidx.navigation.compose.currentBackStackEntryAsState

sealed class BottomNavItem(
    val route: Any,
    val title: String,
    val icon: ImageVector
) {
    object Home : BottomNavItem(HomeTab, "Home", Icons.Default.Home)
    object Search : BottomNavItem(SearchTab, "Search", Icons.Default.Search)
    object Notifications : BottomNavItem(NotificationsTab, "Notifications", Icons.Default.Notifications)
    object Profile : BottomNavItem(ProfileTab, "Profile", Icons.Default.Person)
}

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentDestination = navBackStackEntry?.destination
    
    val items = listOf(
        BottomNavItem.Home,
        BottomNavItem.Search,
        BottomNavItem.Notifications,
        BottomNavItem.Profile
    )
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                items.forEach { item ->
                    NavigationBarItem(
                        icon = { Icon(item.icon, contentDescription = item.title) },
                        label = { Text(item.title) },
                        selected = currentDestination?.route == item.route::class.qualifiedName,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.startDestinationId) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        }
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = HomeTab,
            modifier = Modifier.padding(paddingValues)
        ) {
            composable<HomeTab> { HomeScreen() }
            composable<SearchTab> { SearchScreen() }
            composable<NotificationsTab> { NotificationsScreen() }
            composable<ProfileTab> { ProfileScreen() }
        }
    }
}
```

## Navigation with ViewModels

### Shared ViewModel Across Screens

```kotlin
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.navigation.NavBackStackEntry

@Composable
fun WizardNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = WizardStep1Route
    ) {
        composable<WizardStep1Route> { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry(WizardGraphRoute)
            }
            val sharedViewModel: WizardViewModel = hiltViewModel(parentEntry)
            
            Step1Screen(
                viewModel = sharedViewModel,
                onNext = { navController.navigate(WizardStep2Route) }
            )
        }
        
        composable<WizardStep2Route> { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry(WizardGraphRoute)
            }
            val sharedViewModel: WizardViewModel = hiltViewModel(parentEntry)
            
            Step2Screen(
                viewModel = sharedViewModel,
                onNext = { navController.navigate(WizardStep3Route) },
                onBack = { navController.navigateUp() }
            )
        }
    }
}
```

## Deep Links

### Define Deep Links

```kotlin
import androidx.navigation.navDeepLink

@Serializable
data class ProductRoute(val productId: String)

NavHost(navController, startDestination = HomeRoute) {
    composable<ProductRoute>(
        deepLinks = listOf(
            navDeepLink<ProductRoute>(basePath = "myapp://product")
        )
    ) { backStackEntry ->
        val route = backStackEntry.toRoute<ProductRoute>()
        ProductScreen(productId = route.productId)
    }
}
```

### Handle Deep Links in Manifest

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="myapp"
            android:host="product" />
    </intent-filter>
    
    <!-- HTTP/HTTPS deep links -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="www.myapp.com"
            android:pathPrefix="/product" />
    </intent-filter>
</activity>
```

## Custom Types in Navigation

### Custom Serializer

```kotlin
import kotlinx.serialization.KSerializer
import kotlinx.serialization.Serializable
import kotlinx.serialization.descriptors.PrimitiveKind
import kotlinx.serialization.descriptors.PrimitiveSerialDescriptor
import kotlinx.serialization.encoding.Decoder
import kotlinx.serialization.encoding.Encoder
import java.util.UUID

object UUIDSerializer : KSerializer<UUID> {
    override val descriptor = PrimitiveSerialDescriptor("UUID", PrimitiveKind.STRING)
    
    override fun deserialize(decoder: Decoder): UUID {
        return UUID.fromString(decoder.decodeString())
    }
    
    override fun serialize(encoder: Encoder, value: UUID) {
        encoder.encodeString(value.toString())
    }
}

@Serializable
data class ItemRoute(
    @Serializable(with = UUIDSerializer::class)
    val itemId: UUID
)
```

### Enum Navigation

```kotlin
enum class UserType {
    ADMIN, REGULAR, GUEST
}

@Serializable
data class UserRoute(
    val userId: String,
    val userType: UserType = UserType.REGULAR
)
```

## Navigation Options

### Pop Behavior

```kotlin
// Navigate and clear back stack
navController.navigate(HomeRoute) {
    popUpTo<LoginRoute> { inclusive = true }
}

// Single top
navController.navigate(DetailsRoute("123")) {
    launchSingleTop = true
}

// Save and restore state
navController.navigate(ProfileRoute("user123")) {
    popUpTo<HomeRoute> {
        saveState = true
    }
    launchSingleTop = true
    restoreState = true
}
```

### Conditional Navigation

```kotlin
@Composable
fun AppNavigation(isLoggedIn: Boolean) {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = if (isLoggedIn) HomeRoute else LoginRoute
    ) {
        composable<LoginRoute> {
            LoginScreen(
                onLoginSuccess = {
                    navController.navigate(HomeRoute) {
                        popUpTo<LoginRoute> { inclusive = true }
                    }
                }
            )
        }
        
        composable<HomeRoute> {
            HomeScreen(
                onLogout = {
                    navController.navigate(LoginRoute) {
                        popUpTo(0) { inclusive = true }
                    }
                }
            )
        }
    }
}
```

## Animations

### Add Transition Animations

```kotlin
import androidx.compose.animation.*
import androidx.navigation.compose.composable

NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute>(
        enterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        exitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Left,
                animationSpec = tween(300)
            )
        },
        popEnterTransition = {
            slideIntoContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(300)
            )
        },
        popExitTransition = {
            slideOutOfContainer(
                towards = AnimatedContentTransitionScope.SlideDirection.Right,
                animationSpec = tween(300)
            )
        }
    ) {
        HomeScreen()
    }
}
```

### Fade Animations

```kotlin
composable<DetailsRoute>(
    enterTransition = { fadeIn(animationSpec = tween(300)) },
    exitTransition = { fadeOut(animationSpec = tween(300)) }
) { backStackEntry ->
    val route = backStackEntry.toRoute<DetailsRoute>()
    DetailsScreen(itemId = route.itemId)
}
```

## Result Handling

### Return Results

```kotlin
// Set result
navController.previousBackStackEntry?.savedStateHandle?.set("result", "value")
navController.navigateUp()

// Get result
@Composable
fun HomeScreen(navController: NavController) {
    val result = navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<String?>("result", null)
        ?.collectAsState()
    
    LaunchedEffect(result.value) {
        result.value?.let { value ->
            // Handle result
            navController.currentBackStackEntry?.savedStateHandle?.remove<String>("result")
        }
    }
}
```

## Testing

### Test Navigation

```kotlin
import androidx.compose.ui.test.junit4.createComposeRule
import androidx.navigation.compose.ComposeNavigator
import androidx.navigation.testing.TestNavHostController
import androidx.test.core.app.ApplicationProvider
import org.junit.Rule
import org.junit.Test

class NavigationTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    private lateinit var navController: TestNavHostController
    
    @Test
    fun testNavigation() {
        composeTestRule.setContent {
            navController = TestNavHostController(ApplicationProvider.getApplicationContext())
            navController.navigatorProvider.addNavigator(ComposeNavigator())
            
            AppNavigation(navController)
        }
        
        // Assert start destination
        val route = navController.currentBackStackEntry?.destination?.route
        assert(route == HomeRoute::class.qualifiedName)
        
        // Navigate
        composeTestRule.runOnIdle {
            navController.navigate(ProfileRoute("user123"))
        }
        
        // Assert navigation
        val newRoute = navController.currentBackStackEntry?.destination?.route
        assert(newRoute == ProfileRoute::class.qualifiedName)
    }
}
```

## Best Practices

- Use type-safe routes with Kotlin Serialization
- Keep navigation logic out of ViewModels
- Use single activity architecture
- Handle deep links properly
- Use launchSingleTop to avoid duplicates
- Save and restore state for tabs
- Test navigation flows
- Handle back press correctly

## Common Patterns

### Logout Flow

```kotlin
fun NavController.logout() {
    navigate(LoginRoute) {
        popUpTo(0) { inclusive = true }
        launchSingleTop = true
    }
}
```

### Multi-Step Form

```kotlin
@Serializable
object FormStep1Route

@Serializable
data class FormStep2Route(val name: String)

@Serializable
data class FormStep3Route(val name: String, val email: String)

NavHost(navController, startDestination = FormStep1Route) {
    composable<FormStep1Route> {
        FormStep1Screen(
            onNext = { name ->
                navController.navigate(FormStep2Route(name))
            }
        )
    }
    
    composable<FormStep2Route> { backStackEntry ->
        val route = backStackEntry.toRoute<FormStep2Route>()
        FormStep2Screen(
            name = route.name,
            onNext = { email ->
                navController.navigate(FormStep3Route(route.name, email))
            }
        )
    }
}
```

## Migration from String-based Routes

### Before (String-based)

```kotlin
composable("profile/{userId}") { backStackEntry ->
    val userId = backStackEntry.arguments?.getString("userId")
    ProfileScreen(userId = userId ?: "")
}

navController.navigate("profile/user123")
```

### After (Type-safe)

```kotlin
@Serializable
data class ProfileRoute(val userId: String)

composable<ProfileRoute> { backStackEntry ->
    val route = backStackEntry.toRoute<ProfileRoute>()
    ProfileScreen(userId = route.userId)
}

navController.navigate(ProfileRoute("user123"))
```

## Resources

- [Navigation Compose Documentation](https://developer.android.com/jetpack/compose/navigation)
- [Type-Safe Navigation](https://developer.android.com/guide/navigation/design/type-safety)
- [Navigation Codelab](https://developer.android.com/codelabs/jetpack-compose-navigation)
- [Kotlin Serialization](https://kotlinlang.org/docs/serialization.html)

