# Koin (Dependency Injection) Setup

## Overview
Koin is a lightweight dependency injection framework for Kotlin that uses DSL instead of annotation processing, making it faster to compile than Hilt/Dagger.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
koin = "3.5.6"
koin-compose = "3.5.6"

[libraries]
koin-android = { group = "io.insert-koin", name = "koin-android", version.ref = "koin" }
koin-androidx-compose = { group = "io.insert-koin", name = "koin-androidx-compose", version.ref = "koin-compose" }
koin-androidx-navigation = { group = "io.insert-koin", name = "koin-androidx-navigation", version.ref = "koin" }
koin-test = { group = "io.insert-koin", name = "koin-test", version.ref = "koin" }
koin-test-junit4 = { group = "io.insert-koin", name = "koin-test-junit4", version.ref = "koin" }

[bundles]
koin = ["koin-android", "koin-androidx-compose"]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.koin)
    
    // Optional: Navigation integration
    implementation(libs.koin.androidx.navigation)
    
    // Testing
    testImplementation(libs.koin.test)
    testImplementation(libs.koin.test.junit4)
}
```

### 3. Initialize Koin

In your Application class:

```kotlin
import android.app.Application
import org.koin.android.ext.koin.androidContext
import org.koin.android.ext.koin.androidLogger
import org.koin.core.context.startKoin
import org.koin.core.logger.Level

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        startKoin {
            // Log Koin into Android logger
            androidLogger(if (BuildConfig.DEBUG) Level.ERROR else Level.NONE)
            
            // Reference Android context
            androidContext(this@MyApplication)
            
            // Load modules
            modules(
                appModule,
                networkModule,
                databaseModule,
                repositoryModule,
                viewModelModule
            )
        }
    }
}
```

## Define Modules

### App Module

```kotlin
import org.koin.dsl.module

val appModule = module {
    // Singleton
    single { PreferencesManager(androidContext()) }
    
    // Factory (new instance each time)
    factory { SomeHelper() }
}
```

### Network Module

```kotlin
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import org.koin.dsl.module
import retrofit2.Retrofit

val networkModule = module {
    
    // JSON serializer
    single {
        Json {
            ignoreUnknownKeys = true
            coerceInputValues = true
        }
    }
    
    // OkHttpClient
    single {
        OkHttpClient.Builder()
            .addInterceptor(
                HttpLoggingInterceptor().apply {
                    level = if (BuildConfig.DEBUG) {
                        HttpLoggingInterceptor.Level.BODY
                    } else {
                        HttpLoggingInterceptor.Level.NONE
                    }
                }
            )
            .build()
    }
    
    // Retrofit
    single {
        Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(get())
            .addConverterFactory(
                get<Json>().asConverterFactory("application/json".toMediaType())
            )
            .build()
    }
    
    // API Service
    single { get<Retrofit>().create(ApiService::class.java) }
}
```

### Database Module

```kotlin
import androidx.room.Room
import org.koin.android.ext.koin.androidContext
import org.koin.dsl.module

val databaseModule = module {
    
    // Database
    single {
        Room.databaseBuilder(
            androidContext(),
            AppDatabase::class.java,
            "app_database"
        ).build()
    }
    
    // DAOs
    single { get<AppDatabase>().userDao() }
    single { get<AppDatabase>().postDao() }
}
```

### Repository Module

```kotlin
import org.koin.dsl.module

val repositoryModule = module {
    single { UserRepository(get(), get()) }
    single { PostRepository(get(), get()) }
}
```

### ViewModel Module

```kotlin
import org.koin.androidx.viewmodel.dsl.viewModel
import org.koin.androidx.viewmodel.dsl.viewModelOf
import org.koin.dsl.module

val viewModelModule = module {
    // Traditional way
    viewModel { UserViewModel(get()) }
    
    // Constructor DSL (simpler)
    viewModelOf(::HomeViewModel)
    viewModelOf(::DetailsViewModel)
    
    // With parameters
    viewModel { (userId: Long) ->
        UserDetailViewModel(userId, get())
    }
}
```

## Injection in Code

### In Activities

```kotlin
import org.koin.android.ext.android.inject
import org.koin.androidx.viewmodel.ext.android.viewModel

class MainActivity : AppCompatActivity() {
    
    // Lazy inject
    private val repository: UserRepository by inject()
    
    // ViewModel
    private val viewModel: HomeViewModel by viewModel()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Use repository and viewModel
    }
}
```

### In Fragments

```kotlin
import org.koin.android.ext.android.inject
import org.koin.androidx.viewmodel.ext.android.viewModel

class UserFragment : Fragment() {
    
    private val repository: UserRepository by inject()
    private val viewModel: UserViewModel by viewModel()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // Use dependencies
    }
}
```

### In Compose

```kotlin
import org.koin.androidx.compose.koinViewModel
import org.koin.compose.koinInject

@Composable
fun UserScreen(
    viewModel: UserViewModel = koinViewModel()
) {
    val repository: UserRepository = koinInject()
    
    // Use viewModel and repository
}
```

### With Parameters

```kotlin
// Define ViewModel with parameters
viewModel { (userId: Long) ->
    UserDetailViewModel(userId, get())
}

// In Activity/Fragment
private val viewModel: UserDetailViewModel by viewModel {
    parametersOf(123L)
}

// In Compose
@Composable
fun UserDetailScreen(userId: Long) {
    val viewModel: UserDetailViewModel = koinViewModel {
        parametersOf(userId)
    }
}
```

## Scopes

### Define Scopes

```kotlin
import org.koin.core.qualifier.named
import org.koin.dsl.module

val scopedModule = module {
    // Activity scope
    scope<MainActivity> {
        scoped { ActivityScopedHelper() }
    }
    
    // Named scope
    scope(named("UserSession")) {
        scoped { UserSessionData() }
    }
}
```

### Use Scopes

```kotlin
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Create scope
        val scope = createScope(this)
        
        // Get scoped dependency
        val helper = scope.get<ActivityScopedHelper>()
    }
    
    override fun onDestroy() {
        // Close scope
        closeScope()
        super.onDestroy()
    }
}
```

## Qualifiers

### Named Qualifiers

```kotlin
import org.koin.core.qualifier.named
import org.koin.dsl.module

val qualifiedModule = module {
    single(named("AuthClient")) {
        OkHttpClient.Builder()
            .addInterceptor(AuthInterceptor())
            .build()
    }
    
    single(named("PlainClient")) {
        OkHttpClient.Builder().build()
    }
}

// Usage
class MyRepository(
    @Named("AuthClient") private val authClient: OkHttpClient,
    @Named("PlainClient") private val plainClient: OkHttpClient
) {
    // Use clients
}

// Or with get()
val authClient: OkHttpClient = get(named("AuthClient"))
```

### Custom Qualifiers

```kotlin
import org.koin.core.qualifier.Qualifier
import org.koin.core.qualifier.QualifierValue

sealed class ClientQualifier(override val value: String) : Qualifier {
    object Auth : ClientQualifier("AuthClient")
    object Plain : ClientQualifier("PlainClient")
}

val module = module {
    single(ClientQualifier.Auth) { /* Auth client */ }
    single(ClientQualifier.Plain) { /* Plain client */ }
}

// Usage
val authClient: OkHttpClient = get(ClientQualifier.Auth)
```

## Property Injection

### Define Properties

```kotlin
// In Application
startKoin {
    androidContext(this@MyApplication)
    
    // Load properties from assets
    androidFileProperties("koin.properties")
    
    // Or define programmatically
    properties(
        mapOf(
            "api_url" to "https://api.example.com",
            "api_key" to "your_key"
        )
    )
    
    modules(appModule)
}
```

### Use Properties

```kotlin
import org.koin.core.component.KoinComponent
import org.koin.core.component.get

class ApiConfig : KoinComponent {
    private val apiUrl: String = getKoin().getProperty("api_url", "default_url")
    private val apiKey: String = getKoin().getProperty("api_key") ?: ""
}

// Or in module
val module = module {
    single {
        ApiService(
            baseUrl = getProperty("api_url"),
            apiKey = getProperty("api_key")
        )
    }
}
```

## Testing

### Unit Tests with Koin

```kotlin
import org.junit.After
import org.junit.Before
import org.junit.Test
import org.koin.core.context.startKoin
import org.koin.core.context.stopKoin
import org.koin.dsl.module
import org.koin.test.KoinTest
import org.koin.test.inject

class RepositoryTest : KoinTest {
    
    private val repository: UserRepository by inject()
    
    @Before
    fun setup() {
        startKoin {
            modules(
                module {
                    single { FakeApiService() }
                    single { FakeUserDao() }
                    single { UserRepository(get(), get()) }
                }
            )
        }
    }
    
    @After
    fun teardown() {
        stopKoin()
    }
    
    @Test
    fun testRepository() {
        // Test with injected repository
    }
}
```

### Test with Mock

```kotlin
import io.mockk.mockk
import org.koin.dsl.module
import org.koin.test.KoinTest

class ViewModelTest : KoinTest {
    
    private val mockRepository = mockk<UserRepository>()
    
    @Before
    fun setup() {
        startKoin {
            modules(
                module {
                    single { mockRepository }
                    single { UserViewModel(get()) }
                }
            )
        }
    }
    
    @Test
    fun testViewModel() {
        val viewModel: UserViewModel by inject()
        // Test viewModel with mocked repository
    }
}
```

### Declare Mock

```kotlin
import org.koin.test.mock.declareMock

class MyTest : KoinTest {
    
    @Test
    fun test() {
        startKoin { modules(myModule) }
        
        // Declare mock for a dependency
        val mockRepo = declareMock<UserRepository>()
        
        // Configure mock
        coEvery { mockRepo.getUsers() } returns listOf(testUser)
        
        // Test
    }
}
```

## KoinComponent Interface

### Use Outside Android Components

```kotlin
import org.koin.core.component.KoinComponent
import org.koin.core.component.inject

class MyHelper : KoinComponent {
    
    private val repository: UserRepository by inject()
    
    fun doSomething() {
        // Use repository
    }
}
```

## Migration from Hilt

### Hilt to Koin Comparison

| Hilt | Koin |
|------|------|
| `@Inject constructor` | Constructor injection (automatic) |
| `@HiltViewModel` | `viewModel { }` in module |
| `@Singleton` | `single { }` |
| `@Binds` | `bind<Interface>()` |
| `@Provides` | `single { }` or `factory { }` |
| `@Named` | `named("name")` |
| `@InstallIn(SingletonComponent::class)` | Global module |

### Example Migration

**Hilt:**
```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    @Provides
    @Singleton
    fun provideUserRepository(
        apiService: ApiService,
        userDao: UserDao
    ): UserRepository {
        return UserRepository(apiService, userDao)
    }
}
```

**Koin:**
```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel()

val viewModelModule = module {
    viewModel { UserViewModel(get()) }
}

val repositoryModule = module {
    single { UserRepository(get(), get()) }
}
```

## Best Practices

1. ✅ Organize modules by feature or layer
2. ✅ Use `single` for singletons, `factory` for new instances
3. ✅ Prefer constructor injection
4. ✅ Use scopes for lifecycle-bound dependencies
5. ✅ Test modules independently
6. ✅ Use qualifiers for multiple instances of same type
7. ✅ Keep modules focused and small
8. ✅ Document complex dependency graphs
9. ✅ Use `viewModel` DSL for ViewModels
10. ✅ Close scopes when done

## Advantages over Hilt

✅ **Faster compilation** - No code generation
✅ **Easier to learn** - Simpler DSL
✅ **Runtime DI** - More flexible
✅ **Smaller library** - Less overhead
✅ **Better error messages** - Clearer at runtime
✅ **No annotation processing** - Faster builds

## Disadvantages

❌ **Runtime errors** - No compile-time verification
❌ **Less IDE support** - No navigation to providers
❌ **Reflection-based** - Slightly slower at runtime
❌ **Manual setup** - Must declare all dependencies

## Advanced Features

### Lazy Module Loading

```kotlin
startKoin {
    androidContext(this@MyApplication)
    
    // Load modules initially
    modules(coreModule)
}

// Load modules later
loadKoinModules(featureModule)

// Unload modules
unloadKoinModules(featureModule)
```

### Override Definitions

```kotlin
val testModule = module {
    single<UserRepository>(override = true) {
        FakeUserRepository()
    }
}
```

### Module Includes

```kotlin
val coreModule = module {
    includes(networkModule, databaseModule)
}
```

## Resources

- [Official Documentation](https://insert-koin.io/)
- [Koin for Android](https://insert-koin.io/docs/reference/koin-android/start)
- [Koin Compose](https://insert-koin.io/docs/reference/koin-android/compose)
- [GitHub Repository](https://github.com/InsertKoinIO/koin)

