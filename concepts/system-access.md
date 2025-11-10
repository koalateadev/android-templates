# System Access Deep Dive

## Overview

Understanding Context and system services is essential for accessing Android system features safely and efficiently.

## Context Types

### Application Context vs Activity Context

```kotlin
// Application Context
// - Lives for entire app lifetime
// - Safe to store
// - No UI operations
// - Survives configuration changes

class Repository(
    @ApplicationContext private val context: Context  // ✅ Safe
) {
    fun doWork() {
        val appName = context.getString(R.string.app_name)
        // Use for non-UI operations
    }
}

// Activity Context
// - Lives only during activity lifetime
// - Required for UI operations
// - Shows dialogs, starts activities
// - Destroyed on configuration changes

class LeakyRepository(
    private val activity: Activity  // ❌ Memory leak!
) {
    // Activity reference prevents garbage collection
}
```

### When to Use Each Context

| Operation | Application Context | Activity Context |
|-----------|-------------------|------------------|
| Start Activity | ❌ (needs FLAG_ACTIVITY_NEW_TASK) | ✅ |
| Show Dialog | ❌ | ✅ |
| Inflate Layout | ❌ (wrong theme) | ✅ |
| Get Resources | ✅ | ✅ |
| Access System Service | ✅ | ✅ |
| Start Service | ✅ | ✅ |
| Send Broadcast | ✅ | ✅ |
| Store in Singleton | ✅ | ❌ |

### Get Application Context

```kotlin
// In Activity
val appContext = applicationContext

// In Fragment
val appContext = requireContext().applicationContext

// In ViewModel (with Hilt)
class MyViewModel(
    @ApplicationContext private val context: Context
) : ViewModel()

// In any Context
fun Context.getAppContext(): Context = applicationContext
```

## System Services

### Common System Services

```kotlin
class SystemServicesExample(private val context: Context) {
    
    // Connectivity
    val connectivityManager = context.getSystemService(
        Context.CONNECTIVITY_SERVICE
    ) as ConnectivityManager
    
    // Location
    val locationManager = context.getSystemService(
        Context.LOCATION_SERVICE
    ) as LocationManager
    
    // Notifications
    val notificationManager = context.getSystemService(
        Context.NOTIFICATION_SERVICE
    ) as NotificationManager
    
    // Alarm
    val alarmManager = context.getSystemService(
        Context.ALARM_SERVICE
    ) as AlarmManager
    
    // Vibration
    val vibrator = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        val vibratorManager = context.getSystemService(
            Context.VIBRATOR_MANAGER_SERVICE
        ) as VibratorManager
        vibratorManager.defaultVibrator
    } else {
        @Suppress("DEPRECATION")
        context.getSystemService(Context.VIBRATOR_SERVICE) as Vibrator
    }
    
    // Display
    val windowManager = context.getSystemService(
        Context.WINDOW_SERVICE
    ) as WindowManager
    
    // Input Method
    val inputMethodManager = context.getSystemService(
        Context.INPUT_METHOD_SERVICE
    ) as InputMethodManager
    
    // Clipboard
    val clipboardManager = context.getSystemService(
        Context.CLIPBOARD_SERVICE
    ) as ClipboardManager
    
    // Download
    val downloadManager = context.getSystemService(
        Context.DOWNLOAD_SERVICE
    ) as DownloadManager
    
    // Audio
    val audioManager = context.getSystemService(
        Context.AUDIO_SERVICE
    ) as AudioManager
}
```

### Modern API (API 23+)

```kotlin
// Type-safe service access
val connectivityManager = context.getSystemService<ConnectivityManager>()
val notificationManager = context.getSystemService<NotificationManager>()
```

## Content Providers

### How Content Providers Work

```
App A (Provider)
    ↓
Content Provider (data interface)
    ↓
App B (Consumer) ← queries via ContentResolver
```

### Access Content Provider

```kotlin
fun queryContacts(context: Context): List<Contact> {
    val contacts = mutableListOf<Contact>()
    val contentResolver = context.contentResolver
    
    val cursor = contentResolver.query(
        ContactsContract.Contacts.CONTENT_URI,
        null,
        null,
        null,
        null
    )
    
    cursor?.use {
        while (it.moveToNext()) {
            val id = it.getString(
                it.getColumnIndexOrThrow(ContactsContract.Contacts._ID)
            )
            val name = it.getString(
                it.getColumnIndexOrThrow(ContactsContract.Contacts.DISPLAY_NAME)
            )
            contacts.add(Contact(id, name))
        }
    }
    
    return contacts
}
```

## Broadcast Receivers

### How Broadcasts Work

```
System/App
    ↓
Sends broadcast (Intent)
    ↓
Registered receivers notified
    ↓
onReceive() called
```

### Register Receiver

```kotlin
class MyActivity : AppCompatActivity() {
    
    private val batteryReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            when (intent.action) {
                Intent.ACTION_BATTERY_LOW -> {
                    // Handle low battery
                }
                Intent.ACTION_POWER_CONNECTED -> {
                    // Handle power connected
                }
            }
        }
    }
    
    override fun onStart() {
        super.onStart()
        val filter = IntentFilter().apply {
            addAction(Intent.ACTION_BATTERY_LOW)
            addAction(Intent.ACTION_POWER_CONNECTED)
        }
        registerReceiver(batteryReceiver, filter)
    }
    
    override fun onStop() {
        unregisterReceiver(batteryReceiver)
        super.onStop()
    }
}
```

## Services

### How Services Work

```
App starts Service
    ↓
Service.onCreate()
    ↓
Service.onStartCommand() (for started service)
    or
Service.onBind() (for bound service)
    ↓
Service runs in background
    ↓
Service.onDestroy()
```

### Started vs Bound Services

```kotlin
// Started Service - Runs independently
class MyService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Do background work
        doWork()
        stopSelf()  // Stop when done
        return START_NOT_STICKY
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
}

// Bound Service - Provides client-server interface
class MyBoundService : Service() {
    private val binder = LocalBinder()
    
    inner class LocalBinder : Binder() {
        fun getService(): MyBoundService = this@MyBoundService
    }
    
    override fun onBind(intent: Intent): IBinder {
        return binder
    }
    
    fun doSomething() {
        // Service method called by client
    }
}
```

## Package Manager

### Query Installed Apps

```kotlin
fun getInstalledApps(context: Context): List<String> {
    val packageManager = context.packageManager
    val packages = packageManager.getInstalledApplications(PackageManager.GET_META_DATA)
    
    return packages.map { it.packageName }
}
```

### Check if App Installed

```kotlin
fun isAppInstalled(context: Context, packageName: String): Boolean {
    return try {
        context.packageManager.getPackageInfo(packageName, 0)
        true
    } catch (e: PackageManager.NameNotFoundException) {
        false
    }
}
```

## Resources Access

### How Resources Work

```
Resources stored in:
    /res/values/strings.xml
    /res/drawable/
    /res/layout/
        ↓
Compiled into R.java
        ↓
Accessed via context.resources
        ↓
Returned in correct density/language
```

### Access Resources

```kotlin
// Strings
val appName = context.getString(R.string.app_name)
val formatted = context.getString(R.string.welcome, userName)

// Colors
val color = context.getColor(R.color.primary)

// Dimensions
val padding = context.resources.getDimensionPixelSize(R.dimen.padding)

// Drawables
val drawable = context.getDrawable(R.drawable.icon)

// Arrays
val items = context.resources.getStringArray(R.array.categories)
```

## SharedPreferences

### How SharedPreferences Work

```
Data stored as XML in:
/data/data/[package]/shared_prefs/[name].xml
    ↓
Loaded into memory on first access
    ↓
Changes written asynchronously (apply) or synchronously (commit)
```

### Thread Safety

```kotlin
// SharedPreferences is thread-safe for reading
val value = prefs.getString("key", "default")

// Use apply() for async writes (recommended)
prefs.edit {
    putString("key", "value")
    apply()  // Async, returns immediately
}

// Use commit() only when you need confirmation
val success = prefs.edit {
    putString("key", "value")
    commit()  // Sync, blocks until written
}
```

## Asset Manager

### Access Assets

```kotlin
// Assets stored in /assets/ folder
fun readAsset(context: Context, fileName: String): String {
    return context.assets.open(fileName).use { inputStream ->
        inputStream.bufferedReader().use { it.readText() }
    }
}

// List assets
fun listAssets(context: Context, path: String = ""): Array<String> {
    return context.assets.list(path) ?: emptyArray()
}
```

## Display Metrics

### Get Screen Information

```kotlin
fun getScreenInfo(context: Context): ScreenInfo {
    val displayMetrics = context.resources.displayMetrics
    
    val widthPx = displayMetrics.widthPixels
    val heightPx = displayMetrics.heightPixels
    val density = displayMetrics.density
    val densityDpi = displayMetrics.densityDpi
    
    val widthDp = widthPx / density
    val heightDp = heightPx / density
    
    return ScreenInfo(
        widthPx = widthPx,
        heightPx = heightPx,
        widthDp = widthDp.toInt(),
        heightDp = heightDp.toInt(),
        density = density,
        densityDpi = densityDpi
    )
}
```

## Best Practices

### 1. Use Appropriate Context

```kotlin
// ✅ Good
class Repository(@ApplicationContext context: Context)

// ❌ Bad
class Repository(activity: Activity)
```

### 2. Don't Leak Activity Context

```kotlin
// ❌ Bad - Leaks Activity
object Singleton {
    lateinit var context: Context
}
Singleton.context = activity

// ✅ Good - Safe to store
object Singleton {
    lateinit var context: Context
}
Singleton.context = activity.applicationContext
```

### 3. Use System Services Efficiently

```kotlin
// ❌ Bad - Gets service every time
fun checkConnection() {
    val cm = context.getSystemService<ConnectivityManager>()
    return cm.activeNetwork != null
}

// ✅ Good - Reuse service
class NetworkMonitor(context: Context) {
    private val connectivityManager = 
        context.getSystemService<ConnectivityManager>()!!
    
    fun isConnected(): Boolean {
        return connectivityManager.activeNetwork != null
    }
}
```

## Key Points

**Context:**
- Gateway to system services and resources
- Application context: long-lived, no UI
- Activity context: short-lived, UI operations

**System Services:**
- Singletons shared across app
- Thread-safe
- Cached by system

**Resources:**
- Automatically localized
- Density-aware
- Managed by Android system

## Resources

- [Context Documentation](https://developer.android.com/reference/android/content/Context)
- [System Services](https://developer.android.com/reference/android/content/Context#getSystemService(java.lang.String))
- [Content Providers](https://developer.android.com/guide/topics/providers/content-providers)
- [Broadcast Receivers](https://developer.android.com/guide/components/broadcasts)
- [Services](https://developer.android.com/guide/components/services)

