# Background Work Deep Dive

## Overview

Understanding when to use WorkManager, AlarmManager, or JobScheduler is crucial for reliable background operations.

## Background Execution Limits

### Android Evolution

```
Android 8.0 (Oreo)   → Background execution limits introduced
Android 9.0 (Pie)    → Restrictions tightened
Android 10 (Q)       → Background location restricted
Android 11 (R)       → Background camera/mic restricted
Android 12 (S)       → Exact alarms restricted
Android 13 (T)       → Notification permission required
```

### App Standby Buckets

```
Active      → No restrictions
Working Set → Deferred jobs, alarms delayed
Frequent    → Jobs run few times per day
Rare        → Jobs run once per day
Restricted  → Severe restrictions (bad actors)
```

### Doze Mode

```
Screen Off → App Idle → Doze Mode
    ↓
Network suspended
Jobs deferred
Alarms delayed (except exact alarms)
Wakelocks ignored

Maintenance Windows:
    Brief periods where work executes
    Frequency decreases over time
```

## WorkManager vs AlarmManager vs JobScheduler

### Comparison Table

| Feature | WorkManager | AlarmManager | JobScheduler |
|---------|------------|--------------|--------------|
| **Use Case** | Deferrable guaranteed work | Exact timing needed | System-level jobs |
| **Doze Compatibility** | ✅ Yes | ⚠️ Limited | ✅ Yes |
| **Constraints** | ✅ Network, battery, storage | ❌ No | ✅ Network, charging, idle |
| **Retries** | ✅ Built-in | ❌ Manual | ⚠️ Limited |
| **Min API** | 14+ | All | 21+ (deprecated 23+) |
| **Exact Timing** | ❌ No | ✅ Yes | ❌ No |
| **Chaining** | ✅ Yes | ❌ No | ❌ No |
| **Guaranteed** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Recommended** | ✅ Most cases | Alarms only | ❌ Use WorkManager |

### When to Use Each

**WorkManager** (Recommended for most cases)
```kotlin
// ✅ Use for:
- API sync
- Upload files
- Database cleanup
- Analytics uploads
- App updates
- Any deferrable work that must complete

// ❌ Don't use for:
- Exact timing required (use AlarmManager)
- Real-time updates (use FCM)
- Foreground tasks (use Service)
```

**AlarmManager** (Exact timing)
```kotlin
// ✅ Use for:
- Calendar reminders
- Medication reminders
- Alarm clock apps
- Time-sensitive notifications
- Exact periodic triggers

// ❌ Don't use for:
- Background sync (use WorkManager)
- Network operations (use WorkManager)
```

**JobScheduler** (Deprecated - use WorkManager)
```kotlin
// ⚠️ Deprecated API 23+
// WorkManager uses JobScheduler internally on API 23+
```

## WorkManager Internals

### How WorkManager Works

```
Enqueue Work
    ↓
WorkManager checks constraints
    ↓
If constraints met:
    API 14-22: AlarmManager + BroadcastReceiver
    API 23+:   JobScheduler
    ↓
Worker.doWork() executed
    ↓
Result returned (Success/Failure/Retry)
```

### Worker Execution

```kotlin
class MyWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    
    override fun doWork(): Result {
        // Runs on background thread (not main thread)
        // Should complete in ~10 minutes
        
        return try {
            performWork()
            Result.success()  // Mark as complete
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()  // Retry with backoff
            } else {
                Result.failure()  // Give up
            }
        }
    }
}
```

### CoroutineWorker

```kotlin
class MyCoroutineWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        // Runs in coroutine (can use suspend functions)
        // Default dispatcher: Dispatchers.Default
        
        return try {
            withContext(Dispatchers.IO) {
                fetchDataFromNetwork()
            }
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

## AlarmManager Internals

### How Alarms Work

```
Schedule Alarm
    ↓
AlarmManager stores trigger time
    ↓
At trigger time (even if app killed):
    ↓
System wakes device (if needed)
    ↓
Sends broadcast/starts activity
    ↓
Your code executes
```

### Alarm Types

```kotlin
// Elapsed Realtime - Based on device boot time
AlarmManager.ELAPSED_REALTIME              // Doesn't wake device
AlarmManager.ELAPSED_REALTIME_WAKEUP       // Wakes device

// RTC - Based on wall clock time
AlarmManager.RTC                           // Doesn't wake device
AlarmManager.RTC_WAKEUP                    // Wakes device
```

### Exact vs Inexact Alarms

```kotlin
val alarmManager = context.getSystemService<AlarmManager>()!!

// Inexact alarm (battery friendly)
alarmManager.set(
    AlarmManager.RTC,
    triggerTime,
    pendingIntent
)

// Exact alarm (requires permission on Android 12+)
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    if (alarmManager.canScheduleExactAlarms()) {
        alarmManager.setExact(
            AlarmManager.RTC_WAKEUP,
            triggerTime,
            pendingIntent
        )
    }
}
```

### Exact Alarm Permission

In `AndroidManifest.xml`:
```xml
<!-- Android 12+ -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
```

Request permission:
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
    if (!alarmManager.canScheduleExactAlarms()) {
        val intent = Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM)
        context.startActivity(intent)
    }
}
```

## Foreground Services

### When to Use

```kotlin
// ✅ Use foreground service for:
- Music playback
- Active navigation
- Ongoing file download
- Fitness tracking
- Video call

// ❌ Don't use for:
- Background sync (use WorkManager)
- Periodic updates (use WorkManager)
```

### Foreground Service Types (Android 10+)

```xml
<service
    android:name=".MusicService"
    android:foregroundServiceType="mediaPlayback" />

<!-- Types: -->
<!-- camera, connectedDevice, dataSync, location, mediaPlayback, -->
<!-- mediaProjection, microphone, phoneCall, remoteMessaging, -->
<!-- shortService, specialUse, systemExempted -->
```

### Start Foreground Service

```kotlin
class MusicService : Service() {
    
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            startForeground(
                NOTIFICATION_ID,
                notification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK
            )
        } else {
            startForeground(NOTIFICATION_ID, notification)
        }
        
        return START_NOT_STICKY
    }
    
    override fun onBind(intent: Intent?): IBinder? = null
}
```

## Decision Tree

### Choose the Right Tool

```
Need exact timing (alarm clock, reminder)?
    YES → AlarmManager
    NO  ↓

User-initiated task (download, upload)?
    YES → Foreground Service
    NO  ↓

Ongoing user-aware task (music, navigation)?
    YES → Foreground Service
    NO  ↓

Deferrable work that must complete?
    YES → WorkManager
    NO  ↓

Real-time updates?
    YES → Firebase Cloud Messaging
    NO  ↓

Scheduled task at exact time?
    YES → AlarmManager + WorkManager
```

## Background Work Best Practices

### 1. Choose Right Tool

```kotlin
// ✅ WorkManager for sync
WorkManager.getInstance(context)
    .enqueue(SyncWorker::class.java)

// ✅ AlarmManager for exact time
alarmManager.setExact(time, pendingIntent)

// ✅ Foreground Service for ongoing work
startForegroundService(intent)
```

### 2. Handle Doze Mode

```kotlin
// WorkManager automatically handles Doze
val workRequest = OneTimeWorkRequestBuilder<SyncWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .build()

// Work deferred during Doze, executes in maintenance window
```

### 3. Respect Battery

```kotlin
// Set battery constraints
val constraints = Constraints.Builder()
    .setRequiresBatteryNotLow(true)
    .setRequiresCharging(true)  // Only when charging
    .build()
```

### 4. Use Appropriate Frequency

```kotlin
// ❌ Bad - Too frequent
PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.MINUTES)  // Can't be < 15 min

// ✅ Good - Reasonable interval
PeriodicWorkRequestBuilder<SyncWorker>(15, TimeUnit.MINUTES)  // Minimum

// ✅ Better - Less frequent when possible
PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
```

## Power Management

### Battery Optimization

```kotlin
// Check if app is battery optimized
val powerManager = context.getSystemService<PowerManager>()!!
val packageName = context.packageName

val isIgnoringBatteryOptimizations = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    powerManager.isIgnoringBatteryOptimizations(packageName)
} else {
    true
}

// Request exemption (use sparingly)
if (!isIgnoringBatteryOptimizations) {
    val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
        data = Uri.parse("package:$packageName")
    }
    context.startActivity(intent)
}
```

### Wake Lock (Use Sparingly)

```kotlin
val powerManager = context.getSystemService<PowerManager>()!!
val wakeLock = powerManager.newWakeLock(
    PowerManager.PARTIAL_WAKE_LOCK,
    "MyApp::MyWakeLockTag"
)

// Acquire
wakeLock.acquire(10 * 60 * 1000L)  // 10 minutes timeout

try {
    // Critical work
} finally {
    if (wakeLock.isHeld) {
        wakeLock.release()
    }
}
```

## Summary

**WorkManager:**
- Default choice for deferrable work
- Handles Doze mode
- Guaranteed execution

**AlarmManager:**
- For exact timing needs
- Requires permission (Android 12+)

**Foreground Services:**
- For user-aware ongoing tasks
- Must show notification
- Requires service type

**Considerations:**
- Doze mode delays work
- Battery optimization limits
- Minimize frequency
- Batch operations

## Resources

- [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager)
- [AlarmManager](https://developer.android.com/training/scheduling/alarms)
- [Background Tasks](https://developer.android.com/guide/background)
- [Doze Mode](https://developer.android.com/training/monitoring-device-state/doze-standby)
- [Power Management](https://developer.android.com/about/versions/pie/power)
- [Foreground Services](https://developer.android.com/guide/components/foreground-services)

