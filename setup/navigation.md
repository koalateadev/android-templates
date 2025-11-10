# Navigation Component Setup

## Overview
Navigation Component provides a framework for navigating between destinations in your Android app with type safety and back stack management.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
navigation = "2.8.3"

[libraries]
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }
androidx-navigation-fragment-ktx = { group = "androidx.navigation", name = "navigation-fragment-ktx", version.ref = "navigation" }
androidx-navigation-ui-ktx = { group = "androidx.navigation", name = "navigation-ui-ktx", version.ref = "navigation" }
androidx-navigation-testing = { group = "androidx.navigation", name = "navigation-testing", version.ref = "navigation" }

[plugins]
androidx-navigation-safeargs = { id = "androidx.navigation.safeargs.kotlin", version.ref = "navigation" }
```

### 2. Add Dependencies

**For Compose:**
```kotlin
dependencies {
    implementation(libs.androidx.navigation.compose)
    
    // With Hilt
    implementation(libs.hilt.navigation.compose)
}
```

**For XML-based:**
```kotlin
plugins {
    alias(libs.plugins.androidx.navigation.safeargs)
}

dependencies {
    implementation(libs.androidx.navigation.fragment.ktx)
    implementation(libs.androidx.navigation.ui.ktx)
}
```

## Jetpack Compose Navigation

### Basic Setup

```kotlin
import androidx.compose.runtime.Composable
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToDetails = { id ->
                    navController.navigate("details/$id")
                }
            )
        }
        
        composable("details/{id}") { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailsScreen(
                id = id,
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

### Type-Safe Navigation with Sealed Classes

```kotlin
import kotlinx.serialization.Serializable

sealed class Screen {
    @Serializable
    data object Home : Screen()
    
    @Serializable
    data class Details(val id: String) : Screen()
    
    @Serializable
    data class Profile(val userId: String, val showEdit: Boolean = false) : Screen()
}

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = Screen.Home
    ) {
        composable<Screen.Home> {
            HomeScreen(
                onNavigateToDetails = { id ->
                    navController.navigate(Screen.Details(id))
                }
            )
        }
        
        composable<Screen.Details> { backStackEntry ->
            val details = backStackEntry.toRoute<Screen.Details>()
            DetailsScreen(
                id = details.id,
                onNavigateBack = { navController.navigateUp() }
            )
        }
        
        composable<Screen.Profile> { backStackEntry ->
            val profile = backStackEntry.toRoute<Screen.Profile>()
            ProfileScreen(
                userId = profile.userId,
                showEdit = profile.showEdit
            )
        }
    }
}
```

### Nested Navigation

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController, startDestination = "main") {
        composable("main") {
            MainScreen(navController)
        }
        
        navigation(startDestination = "auth/login", route = "auth") {
            composable("auth/login") {
                LoginScreen()
            }
            composable("auth/register") {
                RegisterScreen()
            }
        }
    }
}
```

### Bottom Navigation

```kotlin
import androidx.compose.material3.*
import androidx.compose.runtime.getValue
import androidx.navigation.compose.currentBackStackEntryAsState

sealed class BottomNavScreen(val route: String, val label: String, val icon: ImageVector) {
    object Home : BottomNavScreen("home", "Home", Icons.Default.Home)
    object Search : BottomNavScreen("search", "Search", Icons.Default.Search)
    object Profile : BottomNavScreen("profile", "Profile", Icons.Default.Person)
}

@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute = navBackStackEntry?.destination?.route
    
    Scaffold(
        bottomBar = {
            NavigationBar {
                val items = listOf(
                    BottomNavScreen.Home,
                    BottomNavScreen.Search,
                    BottomNavScreen.Profile
                )
                
                items.forEach { screen ->
                    NavigationBarItem(
                        icon = { Icon(screen.icon, contentDescription = null) },
                        label = { Text(screen.label) },
                        selected = currentRoute == screen.route,
                        onClick = {
                            navController.navigate(screen.route) {
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
            startDestination = BottomNavScreen.Home.route,
            modifier = Modifier.padding(paddingValues)
        ) {
            composable(BottomNavScreen.Home.route) { HomeScreen() }
            composable(BottomNavScreen.Search.route) { SearchScreen() }
            composable(BottomNavScreen.Profile.route) { ProfileScreen() }
        }
    }
}
```

### Deep Links

```kotlin
NavHost(navController, startDestination = "home") {
    composable(
        route = "details/{id}",
        deepLinks = listOf(
            navDeepLink {
                uriPattern = "myapp://details/{id}"
            },
            navDeepLink {
                uriPattern = "https://myapp.com/details/{id}"
            }
        )
    ) { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id")
        DetailsScreen(id)
    }
}
```

Update `AndroidManifest.xml`:
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="myapp.com"
            android:pathPrefix="/details" />
    </intent-filter>
</activity>
```

### Shared ViewModels

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController, startDestination = "step1") {
        composable("step1") { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry("wizard")
            }
            val sharedViewModel: WizardViewModel = hiltViewModel(parentEntry)
            
            Step1Screen(sharedViewModel)
        }
        
        composable("step2") { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry("wizard")
            }
            val sharedViewModel: WizardViewModel = hiltViewModel(parentEntry)
            
            Step2Screen(sharedViewModel)
        }
    }
}
```

## XML-based Navigation (Fragments)

### Create Navigation Graph

Create `res/navigation/nav_graph.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_details"
            app:destination="@id/detailsFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailsFragment"
        android:name="com.example.DetailsFragment"
        android:label="Details">
        <argument
            android:name="userId"
            app:argType="long" />
        <argument
            android:name="userName"
            app:argType="string"
            android:defaultValue="Unknown" />
    </fragment>
</navigation>
```

### Setup NavHostFragment

In `activity_main.xml`:

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/nav_graph" />
```

### Navigate with Safe Args

```kotlin
// In HomeFragment
val action = HomeFragmentDirections.actionHomeToDetails(
    userId = 123L,
    userName = "John Doe"
)
findNavController().navigate(action)

// In DetailsFragment
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    
    val args: DetailsFragmentArgs by navArgs()
    val userId = args.userId
    val userName = args.userName
}
```

### Setup with Toolbar

```kotlin
class MainActivity : AppCompatActivity() {
    private lateinit var navController: NavController
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        
        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        navController = navHostFragment.navController
        
        setupActionBarWithNavController(navController)
    }
    
    override fun onSupportNavigateUp(): Boolean {
        return navController.navigateUp() || super.onSupportNavigateUp()
    }
}
```

## Navigation Options

### Pop Behavior

```kotlin
// Navigate and clear back stack
navController.navigate("details") {
    popUpTo("home") {
        inclusive = true
    }
}

// Single top (avoid multiple instances)
navController.navigate("details") {
    launchSingleTop = true
}

// Save and restore state
navController.navigate("details") {
    popUpTo("home") {
        saveState = true
    }
    launchSingleTop = true
    restoreState = true
}
```

### Animations

```kotlin
composable(
    "details",
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
    DetailsScreen()
}
```

## Testing

```kotlin
@Test
fun testNavigation() {
    val navController = TestNavHostController(ApplicationProvider.getApplicationContext())
    
    composeTestRule.setContent {
        navController.navigatorProvider.addNavigator(ComposeNavigator())
        
        NavHost(navController = navController, startDestination = "home") {
            composable("home") { HomeScreen() }
            composable("details") { DetailsScreen() }
        }
    }
    
    // Assert current destination
    assertThat(navController.currentDestination?.route).isEqualTo("home")
    
    // Navigate
    navController.navigate("details")
    assertThat(navController.currentDestination?.route).isEqualTo("details")
}
```

## Best Practices

- Use type-safe navigation with sealed classes or Safe Args
- Keep navigation logic in composable/fragment, not ViewModel
- Use single activity architecture with Compose
- Handle deep links for better UX
- Use launchSingleTop to prevent multiple instances
- Save and restore state for bottom navigation
- Test navigation flows

## Common Patterns

### Logout and Clear Stack

```kotlin
fun logout(navController: NavController) {
    navController.navigate("login") {
        popUpTo(0) { inclusive = true }
        launchSingleTop = true
    }
}
```

### Conditional Navigation

```kotlin
@Composable
fun AppNavigation(isLoggedIn: Boolean) {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = if (isLoggedIn) "home" else "login"
    ) {
        // ...
    }
}
```

## Resources

- [Navigation Component Guide](https://developer.android.com/guide/navigation)
- [Compose Navigation](https://developer.android.com/jetpack/compose/navigation)
- [Navigation Codelab](https://developer.android.com/codelabs/android-navigation)

