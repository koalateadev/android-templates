# LeakCanary Setup

## Overview
LeakCanary is a memory leak detection library for Android that automatically finds and reports memory leaks in your app during development.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
leakcanary = "2.14"

[libraries]
leakcanary-android = { group = "com.squareup.leakcanary", name = "leakcanary-android", version.ref = "leakcanary" }
leakcanary-android-instrumentation = { group = "com.squareup.leakcanary", name = "leakcanary-android-instrumentation", version.ref = "leakcanary" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    // Automatic installation for debug builds
    debugImplementation(libs.leakcanary.android)
    
    // For instrumented tests (optional)
    androidTestImplementation(libs.leakcanary.android.instrumentation)
}
```

### 3. That's It!

LeakCanary automatically installs itself in debug builds. No initialization code needed!

## How It Works

LeakCanary automatically:
1. Watches for destroyed Activities and Fragments
2. Detects memory leaks
3. Dumps the heap when a leak is detected
4. Analyzes the heap dump
5. Shows a notification with leak details

## Viewing Leaks

### Notification

When a leak is detected:
1. You'll see a notification
2. Tap the notification to view details
3. LeakCanary shows the leak trace

### In-App

LeakCanary adds an app icon to your launcher (debug builds only):
- View all detected leaks
- See leak traces
- Share leak reports

## Configuration

### Custom Configuration

Create a custom configuration in your Application class:

```kotlin
import android.app.Application
import leakcanary.AppWatcher
import leakcanary.LeakCanary

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Configure LeakCanary
        LeakCanary.config = LeakCanary.config.copy(
            dumpHeap = true,                    // Enable heap dumps
            dumpHeapWhenDebugging = false,      // Disable when debugger attached
            retainedVisibleThreshold = 5,       // Trigger after 5 retained objects
            referenceMatchers = AppWatcher.config.referenceMatchers,
            objectInspectors = AppWatcher.config.objectInspectors,
            onHeapAnalyzedListener = OnHeapAnalyzedListener.Default,
            metadataExtractor = MetadataExtractor.Default,
            computeRetainedHeapSize = true,     // Calculate retained heap size
            maxStoredHeapDumps = 7,            // Keep last 7 heap dumps
            requestWriteExternalStoragePermission = false
        )
        
        // Configure AppWatcher
        AppWatcher.config = AppWatcher.config.copy(
            enabled = true,                     // Enable leak detection
            watchActivities = true,             // Watch Activity leaks
            watchFragments = true,              // Watch Fragment leaks
            watchViewModels = true,             // Watch ViewModel leaks
            watchDurationMillis = 5000L         // Wait 5s before checking
        )
    }
}
```

### Disable in Tests

```kotlin
class MyTestRunner : AndroidJUnitRunner() {
    override fun onCreate(arguments: Bundle?) {
        super.onCreate(arguments)
        // Disable LeakCanary in tests
        AppWatcher.config = AppWatcher.config.copy(enabled = false)
    }
}
```

## Watching Custom Objects

### Watch Specific Objects

```kotlin
import leakcanary.AppWatcher

class MyRepository {
    private val disposables = CompositeDisposable()
    
    fun initialize() {
        // Watch for leaks when this object should be GC'd
        AppWatcher.objectWatcher.watch(
            watchedObject = this,
            description = "MyRepository instance"
        )
    }
    
    fun cleanup() {
        disposables.clear()
    }
}
```

### Watch ViewModels

```kotlin
import androidx.lifecycle.ViewModel
import leakcanary.AppWatcher

class MyViewModel : ViewModel() {
    
    init {
        // Automatically watched if AppWatcher.config.watchViewModels = true
    }
    
    override fun onCleared() {
        super.onCleared()
        // Clean up resources
    }
}
```

## Ignoring Known Leaks

### Add Reference Matchers

```kotlin
import leakcanary.ReferenceMatcher
import shark.IgnoredReferenceMatcher
import shark.LibraryLeakReferenceMatcher

LeakCanary.config = LeakCanary.config.copy(
    referenceMatchers = LeakCanary.config.referenceMatchers + listOf(
        // Ignore known Android system leaks
        IgnoredReferenceMatcher(
            pattern = "android.view.inputmethod.InputMethodManager.mServedView",
            description = "Known Android system leak",
            patternApplies = { true }
        ),
        
        // Ignore third-party library leaks
        LibraryLeakReferenceMatcher(
            pattern = "com.google.android.gms.maps.MapView",
            description = "Google Maps leak",
            patternApplies = { true }
        )
    )
)
```

## Leak Traces

### Understanding Leak Traces

Example leak trace:
```
┬───
│ GC Root: System class
│
├─ android.app.ActivityThread instance
│    Leaking: NO
│    ↓ ActivityThread.mActivities
│
├─ android.util.ArrayMap instance
│    Leaking: UNKNOWN
│    ↓ ArrayMap[0]
│
├─ android.app.ActivityThread$ActivityClientRecord instance
│    Leaking: UNKNOWN
│    ↓ ActivityClientRecord.activity
│
├─ com.example.MainActivity instance
│    Leaking: YES (Activity destroyed but still referenced)
│    ↓ MainActivity.leakedObject
│
╰→ com.example.LeakedObject instance
     Leaking: YES
```

### Leak Categories

- **Leaking: YES** - Object should be garbage collected
- **Leaking: NO** - Object is supposed to be in memory
- **Leaking: UNKNOWN** - Cannot determine

## Common Leak Patterns

### Activity Leaks

**Problem: Static Reference**
```kotlin
// DON'T DO THIS
class MainActivity : AppCompatActivity() {
    companion object {
        var instance: MainActivity? = null  // LEAK!
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        instance = this  // Retains Activity forever
    }
}
```

**Solution: Use Application Context**
```kotlin
class MyManager(private val context: Context) {
    // Use Application context, not Activity
}

// In Activity
val manager = MyManager(applicationContext)
```

### Listener Leaks

**Problem: Unregistered Listener**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        someService.registerListener(object : Listener {
            override fun onEvent() {
                // This holds reference to Activity
            }
        })
        // LEAK: Never unregistered
    }
}
```

**Solution: Unregister in onDestroy**
```kotlin
class MyActivity : AppCompatActivity() {
    private val listener = object : Listener {
        override fun onEvent() {
            // Handle event
        }
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        someService.registerListener(listener)
    }
    
    override fun onDestroy() {
        someService.unregisterListener(listener)
        super.onDestroy()
    }
}
```

### Handler Leaks

**Problem: Non-Static Handler**
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper()) {
        // This holds reference to Activity
        true
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed({ /* work */ }, 60000)  // LEAK if Activity destroyed
    }
}
```

**Solution: Static Handler or Cancel Messages**
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    
    override fun onDestroy() {
        handler.removeCallbacksAndMessages(null)  // Cancel all messages
        super.onDestroy()
    }
}
```

### Thread Leaks

**Problem: Anonymous Thread**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        Thread {
            // Long running work
            // This holds reference to Activity
        }.start()  // LEAK!
    }
}
```

**Solution: Use Coroutines with Lifecycle**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch {
            // Automatically cancelled when Activity destroyed
            withContext(Dispatchers.IO) {
                // Long running work
            }
        }
    }
}
```

## Integration with CI/CD

### Fail Build on Leaks (Instrumented Tests)

```kotlin
import leakcanary.DetectLeaksAfterTestSuccess
import org.junit.Rule

class MyInstrumentedTest {
    
    @get:Rule
    val detectLeaksRule = DetectLeaksAfterTestSuccess()
    
    @Test
    fun testFeature() {
        // Test code
        // Fails if memory leaks detected
    }
}
```

### Generate Reports

```kotlin
import leakcanary.LeakCanary

// In your test setup
LeakCanary.config = LeakCanary.config.copy(
    onHeapAnalyzedListener = { heapAnalysis ->
        // Save report to file
        val reportFile = File(context.filesDir, "leak_report.txt")
        reportFile.writeText(heapAnalysis.toString())
        
        // Or upload to server
        uploadLeakReport(heapAnalysis)
    }
)
```

## Shark (Heap Analyzer)

LeakCanary uses Shark for heap analysis. You can use Shark directly:

```kotlin
import shark.HeapAnalyzer
import shark.SharkLog

// Analyze a heap dump file
val heapAnalyzer = HeapAnalyzer(OnAnalysisProgressListener.NONE)
val heapAnalysis = heapAnalyzer.analyze(
    heapDumpFile = File("/path/to/heap_dump.hprof"),
    leakingObjectFinder = KeyedWeakReferenceFinder,
    referenceMatchers = AndroidReferenceMatchers.appDefaults,
    computeRetainedHeapSize = true,
    objectInspectors = AndroidObjectInspectors.appDefaults,
    metadataExtractor = AndroidMetadataExtractor
)

when (heapAnalysis) {
    is HeapAnalysisSuccess -> {
        heapAnalysis.applicationLeaks.forEach { leak ->
            println(leak)
        }
    }
    is HeapAnalysisFailure -> {
        println("Analysis failed: ${heapAnalysis.exception}")
    }
}
```

## Best Practices

- Use LeakCanary in all debug builds
- Fix leaks as soon as detected
- Use lifecycle-aware components
- Unregister listeners in onDestroy()
- Avoid static references to Activities/Views
- Cancel background work when Activity destroyed
- Use Application context for long-lived objects
- Test for leaks in automated tests

## Troubleshooting

### LeakCanary Not Working

1. Check it's only in `debugImplementation`
2. Verify it's a debug build
3. Check if disabled in Application class
4. Ensure enough retained objects threshold

### Too Many False Positives

```kotlin
// Increase threshold
AppWatcher.config = AppWatcher.config.copy(
    watchDurationMillis = 10000L  // Wait longer before checking
)

LeakCanary.config = LeakCanary.config.copy(
    retainedVisibleThreshold = 10  // Require more retained objects
)
```

### Performance Impact

LeakCanary only runs in debug builds and has minimal impact. If needed:

```kotlin
// Disable during performance testing
AppWatcher.config = AppWatcher.config.copy(enabled = false)
```

## Comparison with Other Tools

| Feature | LeakCanary | Android Profiler | MAT |
|---------|------------|------------------|-----|
| Automatic Detection | ✅ | ❌ | ❌ |
| Real-time | ✅ | ✅ | ❌ |
| Easy Setup | ✅ | ✅ | ❌ |
| Detailed Analysis | ✅ | ✅ | ✅ |
| Production Use | ❌ | ❌ | ✅ |

## Resources

- [Official Documentation](https://square.github.io/leakcanary/)
- [Fundamentals](https://square.github.io/leakcanary/fundamentals/)
- [Recipes](https://square.github.io/leakcanary/recipes/)
- [GitHub Repository](https://github.com/square/leakcanary)
- [Shark Library](https://square.github.io/leakcanary/shark/)

