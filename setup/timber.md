# Timber (Logging) Setup

## Overview
Timber is a lightweight logging library for Android that makes logging cleaner and more powerful than standard Android Log.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
timber = "5.0.1"

[libraries]
timber = { group = "com.jakewharton.timber", name = "timber", version.ref = "timber" }
```

### 2. Add Dependency

```kotlin
dependencies {
    implementation(libs.timber)
}
```

### 3. Initialize Timber

In your Application class:

```kotlin
import android.app.Application
import timber.log.Timber

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            // Debug build: plant DebugTree
            Timber.plant(Timber.DebugTree())
        } else {
            // Release build: plant custom tree for crash reporting
            Timber.plant(CrashReportingTree())
        }
    }
}
```

## Basic Usage

### Simple Logging

```kotlin
import timber.log.Timber

class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Different log levels
        Timber.v("Verbose log")
        Timber.d("Debug log")
        Timber.i("Info log")
        Timber.w("Warning log")
        Timber.e("Error log")
        
        // With format strings
        Timber.d("User logged in: %s", userName)
        Timber.i("Items loaded: %d", itemCount)
        
        // With multiple arguments
        Timber.d("User: %s, Age: %d", userName, age)
    }
}
```

### Logging Exceptions

```kotlin
try {
    riskyOperation()
} catch (e: Exception) {
    // Log exception
    Timber.e(e, "Error performing operation")
    
    // Or with additional message
    Timber.e(e, "Failed to load user: %s", userId)
}
```

### Lazy String Evaluation

```kotlin
// String is only evaluated if DEBUG level is enabled
Timber.d { "Expensive string operation: ${expensiveOperation()}" }
```

## Custom Trees

### Crash Reporting Tree

```kotlin
import timber.log.Timber
import com.google.firebase.crashlytics.FirebaseCrashlytics

class CrashReportingTree : Timber.Tree() {
    
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority == Log.VERBOSE || priority == Log.DEBUG) {
            return // Don't log verbose/debug in production
        }
        
        // Log to Crashlytics
        val crashlytics = FirebaseCrashlytics.getInstance()
        crashlytics.log(message)
        
        if (t != null) {
            crashlytics.recordException(t)
        }
    }
}
```

### File Logging Tree

```kotlin
import android.content.Context
import timber.log.Timber
import java.io.File
import java.text.SimpleDateFormat
import java.util.*

class FileLoggingTree(private val context: Context) : Timber.Tree() {
    
    private val dateFormat = SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS", Locale.US)
    
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        val logFile = getLogFile()
        val timestamp = dateFormat.format(Date())
        val level = when (priority) {
            Log.VERBOSE -> "V"
            Log.DEBUG -> "D"
            Log.INFO -> "I"
            Log.WARN -> "W"
            Log.ERROR -> "E"
            else -> "?"
        }
        
        val logMessage = "$timestamp $level/$tag: $message\n"
        
        try {
            logFile.appendText(logMessage)
            
            if (t != null) {
                logFile.appendText(t.stackTraceToString() + "\n")
            }
        } catch (e: Exception) {
            // Handle logging error
        }
    }
    
    private fun getLogFile(): File {
        val logsDir = File(context.filesDir, "logs")
        logsDir.mkdirs()
        return File(logsDir, "app_log.txt")
    }
}
```

### Tagged Tree

```kotlin
class TaggedTree(private val tag: String) : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        super.log(priority, this.tag, message, t)
    }
}

// Usage
Timber.plant(TaggedTree("MyApp"))
```

### Debug Tree with Enhanced Info

```kotlin
class EnhancedDebugTree : Timber.DebugTree() {
    
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        val stackTrace = Throwable().stackTrace
        if (stackTrace.size >= 5) {
            val element = stackTrace[5]
            val className = element.className.substringAfterLast('.')
            val methodName = element.methodName
            val lineNumber = element.lineNumber
            
            val enhancedTag = "$tag ($className.$methodName:$lineNumber)"
            super.log(priority, enhancedTag, message, t)
        } else {
            super.log(priority, tag, message, t)
        }
    }
    
    override fun createStackElementTag(element: StackTraceElement): String {
        return "(${element.fileName}:${element.lineNumber})#${element.methodName}"
    }
}
```

## Multiple Trees

### Plant Multiple Trees

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            // Debug: multiple trees
            Timber.plant(Timber.DebugTree())
            Timber.plant(FileLoggingTree(this))
        } else {
            // Release: crash reporting only
            Timber.plant(CrashReportingTree())
        }
    }
}
```

### Uproot Trees

```kotlin
// Remove specific tree
val tree = Timber.DebugTree()
Timber.plant(tree)
// Later...
Timber.uproot(tree)

// Remove all trees
Timber.uprootAll()
```

## Best Practices

### Use Extension Functions

```kotlin
// Create extension functions for common patterns
fun Any.logd(message: String) {
    Timber.tag(this::class.java.simpleName).d(message)
}

fun Any.loge(throwable: Throwable, message: String = "") {
    Timber.tag(this::class.java.simpleName).e(throwable, message)
}

// Usage
class MyClass {
    fun doSomething() {
        logd("Doing something")
    }
}
```

### Conditional Logging

```kotlin
// Don't do this (string is always evaluated)
Timber.d("Result: " + expensiveOperation())

// Do this (only evaluated if logging is enabled)
if (BuildConfig.DEBUG) {
    Timber.d("Result: %s", expensiveOperation())
}

// Or use lambda
Timber.d { "Result: ${expensiveOperation()}" }
```

### Network Logging

```kotlin
class NetworkLoggingInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        
        Timber.d("→ %s %s", request.method, request.url)
        
        val startTime = System.currentTimeMillis()
        val response = chain.proceed(request)
        val duration = System.currentTimeMillis() - startTime
        
        Timber.d("← %s %s (%dms)", response.code, request.url, duration)
        
        return response
    }
}
```

## Integration with Other Tools

### With Hilt

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object LoggingModule {
    
    @Provides
    @Singleton
    fun provideTimber(
        @ApplicationContext context: Context
    ): Timber.Tree {
        return if (BuildConfig.DEBUG) {
            Timber.DebugTree()
        } else {
            CrashReportingTree()
        }
    }
}
```

### With Firebase Crashlytics

```kotlin
class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.WARN) return
        
        val crashlytics = FirebaseCrashlytics.getInstance()
        
        // Set custom keys for better debugging
        crashlytics.setCustomKey("priority", priority)
        tag?.let { crashlytics.setCustomKey("tag", it) }
        
        // Log message
        crashlytics.log("$priority/$tag: $message")
        
        // Record exception if present
        t?.let { crashlytics.recordException(it) }
    }
}
```

### With Logcat Formatting

```kotlin
class ColoredDebugTree : Timber.DebugTree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        val emoji = when (priority) {
            Log.VERBOSE -> "💬"
            Log.DEBUG -> "🐛"
            Log.INFO -> "ℹ️"
            Log.WARN -> "⚠️"
            Log.ERROR -> "❌"
            else -> ""
        }
        
        super.log(priority, tag, "$emoji $message", t)
    }
}
```

## ViewModel Logging

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    private val tag = this::class.java.simpleName
    
    fun loadUsers() {
        Timber.tag(tag).d("Loading users")
        
        viewModelScope.launch {
            try {
                val users = repository.getUsers()
                Timber.tag(tag).i("Loaded %d users", users.size)
            } catch (e: Exception) {
                Timber.tag(tag).e(e, "Failed to load users")
            }
        }
    }
}
```

## Repository Logging

```kotlin
class UserRepository @Inject constructor(
    private val apiService: ApiService,
    private val userDao: UserDao
) {
    private val tag = this::class.java.simpleName
    
    suspend fun getUsers(): Result<List<User>> {
        Timber.tag(tag).d("Fetching users from API")
        
        return try {
            val response = apiService.getUsers()
            
            if (response.isSuccessful && response.body() != null) {
                val users = response.body()!!
                Timber.tag(tag).i("API returned %d users", users.size)
                
                // Cache in database
                userDao.insertUsers(users)
                Timber.tag(tag).d("Cached users in database")
                
                Result.success(users)
            } else {
                Timber.tag(tag).w("API error: %d", response.code())
                Result.failure(Exception("API Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            Timber.tag(tag).e(e, "Network error")
            Result.failure(e)
        }
    }
}
```

## Performance Logging

```kotlin
inline fun <T> measureTimber(tag: String, message: String, block: () -> T): T {
    val startTime = System.currentTimeMillis()
    val result = block()
    val duration = System.currentTimeMillis() - startTime
    Timber.tag(tag).d("$message took ${duration}ms")
    return result
}

// Usage
val users = measureTimber("Repository", "Fetch users") {
    repository.getUsers()
}
```

## ProGuard Rules

```proguard
# Timber
-dontwarn org.jetbrains.annotations.**
-keep class timber.log.** { *; }
```

## Comparison with Android Log

### Android Log
```kotlin
Log.d("MyTag", "Debug message")
Log.e("MyTag", "Error message", exception)
```

### Timber
```kotlin
Timber.d("Debug message")  // Tag automatically added
Timber.e(exception, "Error message")
```

### Benefits
- No need for TAG constants
- Automatic class name tagging
- Easy to disable in production
- Extensible with custom trees
- Better string formatting
- Lazy evaluation support

## Best Practices

- Initialize Timber in Application.onCreate()
- Use DebugTree for debug builds only
- Create custom trees for production
- Don't construct expensive strings in log statements
- Use appropriate log levels
- Log exceptions with context
- Remove verbose/debug logs in production
- Integrate with crash reporting tools

## Resources

- [Official Documentation](https://github.com/JakeWharton/timber)
- [Timber Samples](https://github.com/JakeWharton/timber/tree/trunk/timber-sample)

