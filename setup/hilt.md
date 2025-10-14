# Hilt (Dependency Injection) Setup

## Overview
Hilt is Android's recommended dependency injection library, built on top of Dagger, that simplifies DI setup in Android apps.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
hilt = "2.52"
hilt-navigation-compose = "1.2.0"

[libraries]
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hilt-navigation-compose" }

[plugins]
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

### 2. Add Plugins and Dependencies

In root `build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.hilt) apply false
}
```

In app `build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

dependencies {
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
    
    // For Compose Navigation
    implementation(libs.hilt.navigation.compose)
}
```

### 3. Create Application Class

```kotlin
import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class MyApplication : Application()
```

Update `AndroidManifest.xml`:
```xml
<application
    android:name=".MyApplication"
    ...>
```

### 4. Inject into Activities

```kotlin
import androidx.appcompat.app.AppCompatActivity
import dagger.hilt.android.AndroidEntryPoint
import javax.inject.Inject

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    
    @Inject
    lateinit var repository: UserRepository
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // repository is injected and ready to use
    }
}
```

### 5. Inject into ViewModels

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    val users = repository.getAllUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

### 6. Create Modules

**Provide Dependencies:**

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideUserRepository(
        userDao: UserDao,
        apiService: ApiService
    ): UserRepository {
        return UserRepository(userDao, apiService)
    }
}
```

**Bind Interfaces:**

```kotlin
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

interface UserDataSource {
    suspend fun getUsers(): List<User>
}

class UserDataSourceImpl @Inject constructor(
    private val apiService: ApiService
) : UserDataSource {
    override suspend fun getUsers(): List<User> {
        return apiService.getUsers().body() ?: emptyList()
    }
}

@Module
@InstallIn(SingletonComponent::class)
abstract class DataModule {
    
    @Binds
    @Singleton
    abstract fun bindUserDataSource(
        impl: UserDataSourceImpl
    ): UserDataSource
}
```

## Component Scopes

| Component | Lifetime | Use For |
|-----------|----------|---------|
| `SingletonComponent` | Application | Singletons, Repository, Database |
| `ActivityRetainedComponent` | Activity (survives config changes) | ViewModels |
| `ActivityComponent` | Activity | Activity-specific dependencies |
| `FragmentComponent` | Fragment | Fragment-specific dependencies |
| `ViewComponent` | View | View-specific dependencies |
| `ServiceComponent` | Service | Service-specific dependencies |

```kotlin
// Application scope
@InstallIn(SingletonComponent::class)

// Activity scope
@InstallIn(ActivityComponent::class)

// Fragment scope
@InstallIn(FragmentComponent::class)
```

## Qualifiers

**Define custom qualifiers:**

```kotlin
import javax.inject.Qualifier

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class PlainOkHttpClient

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    @AuthInterceptorOkHttpClient
    fun provideAuthOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .build()
    }
    
    @Provides
    @Singleton
    @PlainOkHttpClient
    fun providePlainOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build()
    }
}

// Usage
@HiltViewModel
class MyViewModel @Inject constructor(
    @AuthInterceptorOkHttpClient private val authClient: OkHttpClient,
    @PlainOkHttpClient private val plainClient: OkHttpClient
) : ViewModel()
```

## Common Modules

### Database Module

```kotlin
import android.content.Context
import androidx.room.Room
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {
    
    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context
    ): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
    
    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }
}
```

### Network Module

```kotlin
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
    }
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            })
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        json: Json
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(
                json.asConverterFactory("application/json".toMediaType())
            )
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

### Dispatcher Module

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers
import javax.inject.Qualifier

@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class MainDispatcher

@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO
    
    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
    
    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}

// Usage
class UserRepository @Inject constructor(
    @IoDispatcher private val ioDispatcher: CoroutineDispatcher
) {
    suspend fun loadUsers() = withContext(ioDispatcher) {
        // IO operations
    }
}
```

## Compose Integration

### Inject ViewModel in Composable

```kotlin
import androidx.compose.runtime.Composable
import androidx.hilt.navigation.compose.hiltViewModel

@Composable
fun UserScreen(
    viewModel: UserViewModel = hiltViewModel()
) {
    val users by viewModel.users.collectAsState()
    
    LazyColumn {
        items(users) { user ->
            Text(user.name)
        }
    }
}
```

### Navigation with Hilt

```kotlin
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController

@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(navController, startDestination = "home") {
        composable("home") {
            HomeScreen(hiltViewModel())
        }
        composable("details/{id}") { backStackEntry ->
            DetailsScreen(hiltViewModel())
        }
    }
}
```

## Testing

### Test Module

```kotlin
@Module
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [AppModule::class]
)
object TestAppModule {
    
    @Provides
    @Singleton
    fun provideTestRepository(): UserRepository {
        return FakeUserRepository()
    }
}
```

### Hilt Test

```kotlin
import dagger.hilt.android.testing.HiltAndroidRule
import dagger.hilt.android.testing.HiltAndroidTest
import org.junit.Before
import org.junit.Rule
import org.junit.Test
import javax.inject.Inject

@HiltAndroidTest
class RepositoryTest {
    
    @get:Rule
    var hiltRule = HiltAndroidRule(this)
    
    @Inject
    lateinit var repository: UserRepository
    
    @Before
    fun init() {
        hiltRule.inject()
    }
    
    @Test
    fun testRepository() = runTest {
        val users = repository.getUsers()
        assertThat(users).isNotEmpty()
    }
}
```

## Best Practices

1. ✅ Use `@Singleton` for app-wide dependencies
2. ✅ Use `@Binds` instead of `@Provides` when possible (more efficient)
3. ✅ Keep modules focused and organized by feature
4. ✅ Use qualifiers for multiple instances of same type
5. ✅ Inject interfaces, not implementations
6. ✅ Use constructor injection when possible
7. ✅ Avoid field injection in fragments (use entry point or ViewModel)

## Common Issues

### Field Injection in Fragments

```kotlin
// Avoid
@AndroidEntryPoint
class MyFragment : Fragment() {
    @Inject
    lateinit var repository: UserRepository
}

// Better - use ViewModel
@AndroidEntryPoint
class MyFragment : Fragment() {
    private val viewModel: MyViewModel by viewModels()
}
```

### Scoped Dependencies

```kotlin
// Correct - matching scopes
@Singleton
class Repository @Inject constructor(
    private val dao: UserDao // Also @Singleton
)

// Wrong - scope mismatch
@ActivityScoped
class Repository @Inject constructor(
    private val dao: UserDao // @Singleton - will fail
)
```

## Resources

- [Official Documentation](https://dagger.dev/hilt/)
- [Android Hilt Guide](https://developer.android.com/training/dependency-injection/hilt-android)
- [Hilt Codelab](https://developer.android.com/codelabs/android-hilt)

