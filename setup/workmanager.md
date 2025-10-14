# WorkManager Setup

## Overview
WorkManager is the recommended solution for persistent work that needs guaranteed execution, even if the app process is killed or the device restarts.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
work = "2.9.1"

[libraries]
androidx-work-runtime-ktx = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "work" }
androidx-work-testing = { group = "androidx.work", name = "work-testing", version.ref = "work" }
androidx-hilt-work = { group = "androidx.hilt", name = "hilt-work", version = "1.2.0" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.androidx.work.runtime.ktx)
    
    // For Hilt integration
    implementation(libs.androidx.hilt.work)
    ksp(libs.hilt.compiler)
    
    // For testing
    androidTestImplementation(libs.androidx.work.testing)
}
```

## Basic Worker

### Create Worker

```kotlin
import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters

class SyncDataWorker(
    appContext: Context,
    workerParams: WorkerParameters
) : CoroutineWorker(appContext, workerParams) {
    
    override suspend fun doWork(): Result {
        return try {
            // Do your work here
            syncData()
            
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
    
    private suspend fun syncData() {
        // Perform sync operation
    }
    
    companion object {
        const val WORK_NAME = "sync_data_work"
    }
}
```

### Schedule Work

```kotlin
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.workDataOf

class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // One-time work request
        val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
            .build()
        
        WorkManager.getInstance(this)
            .enqueue(workRequest)
    }
}
```

## Work Constraints

### Add Constraints

```kotlin
import androidx.work.Constraints
import androidx.work.NetworkType
import androidx.work.OneTimeWorkRequestBuilder

val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)  // Requires network
    .setRequiresBatteryNotLow(true)                 // Battery not low
    .setRequiresCharging(false)                     // Charging not required
    .setRequiresStorageNotLow(true)                 // Storage not low
    .setRequiresDeviceIdle(false)                   // Device idle not required
    .build()

val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setConstraints(constraints)
    .build()
```

## Periodic Work

### Schedule Periodic Work

```kotlin
import androidx.work.PeriodicWorkRequestBuilder
import androidx.work.ExistingPeriodicWorkPolicy
import java.util.concurrent.TimeUnit

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        setupPeriodicWork()
    }
    
    private fun setupPeriodicWork() {
        val periodicWorkRequest = PeriodicWorkRequestBuilder<SyncDataWorker>(
            repeatInterval = 15,           // Minimum is 15 minutes
            repeatIntervalTimeUnit = TimeUnit.MINUTES,
            flexTimeInterval = 5,          // Flex interval
            flexTimeIntervalUnit = TimeUnit.MINUTES
        )
            .setConstraints(constraints)
            .build()
        
        WorkManager.getInstance(this)
            .enqueueUniquePeriodicWork(
                SyncDataWorker.WORK_NAME,
                ExistingPeriodicWorkPolicy.KEEP,  // or REPLACE, UPDATE
                periodicWorkRequest
            )
    }
}
```

## Input and Output Data

### Pass Input Data

```kotlin
import androidx.work.workDataOf

val inputData = workDataOf(
    "user_id" to 123,
    "action" to "sync"
)

val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setInputData(inputData)
    .build()

// In Worker
class SyncDataWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val userId = inputData.getInt("user_id", -1)
        val action = inputData.getString("action") ?: ""
        
        // Use the data
        
        return Result.success()
    }
}
```

### Return Output Data

```kotlin
class SyncDataWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val itemsSynced = syncData()
        
        val outputData = workDataOf(
            "items_synced" to itemsSynced,
            "sync_time" to System.currentTimeMillis()
        )
        
        return Result.success(outputData)
    }
}

// Observe output
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(workRequest.id)
    .observe(lifecycleOwner) { workInfo ->
        if (workInfo?.state == WorkInfo.State.SUCCEEDED) {
            val itemsSynced = workInfo.outputData.getInt("items_synced", 0)
            val syncTime = workInfo.outputData.getLong("sync_time", 0)
        }
    }
```

## Progress Updates

### Report Progress

```kotlin
class DownloadWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val totalItems = 100
        
        for (i in 1..totalItems) {
            downloadItem(i)
            
            // Update progress
            setProgress(workDataOf(
                "progress" to i,
                "total" to totalItems
            ))
        }
        
        return Result.success()
    }
}

// Observe progress
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(workRequest.id)
    .observe(lifecycleOwner) { workInfo ->
        val progress = workInfo.progress.getInt("progress", 0)
        val total = workInfo.progress.getInt("total", 100)
        val percentage = (progress * 100) / total
        
        updateProgressBar(percentage)
    }
```

## Work Chaining

### Chain Work

```kotlin
import androidx.work.OneTimeWorkRequestBuilder

val downloadWork = OneTimeWorkRequestBuilder<DownloadWorker>().build()
val processWork = OneTimeWorkRequestBuilder<ProcessWorker>().build()
val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>().build()

WorkManager.getInstance(context)
    .beginWith(downloadWork)
    .then(processWork)
    .then(uploadWork)
    .enqueue()
```

### Parallel Work

```kotlin
val work1 = OneTimeWorkRequestBuilder<Worker1>().build()
val work2 = OneTimeWorkRequestBuilder<Worker2>().build()
val work3 = OneTimeWorkRequestBuilder<Worker3>().build()
val finalWork = OneTimeWorkRequestBuilder<FinalWorker>().build()

WorkManager.getInstance(context)
    .beginWith(listOf(work1, work2, work3))  // Run in parallel
    .then(finalWork)                          // Run after all complete
    .enqueue()
```

### Combine Chains

```kotlin
val chain1 = WorkManager.getInstance(context)
    .beginWith(work1)
    .then(work2)

val chain2 = WorkManager.getInstance(context)
    .beginWith(work3)
    .then(work4)

WorkContinuation.combine(listOf(chain1, chain2))
    .then(finalWork)
    .enqueue()
```

## Unique Work

### Enqueue Unique Work

```kotlin
import androidx.work.ExistingWorkPolicy

WorkManager.getInstance(context)
    .enqueueUniqueWork(
        "sync_work",
        ExistingWorkPolicy.KEEP,      // REPLACE, APPEND, APPEND_OR_REPLACE
        workRequest
    )
```

### Policies

- **KEEP** - Keep existing work, ignore new
- **REPLACE** - Cancel existing, start new
- **APPEND** - Add to end of existing chain
- **APPEND_OR_REPLACE** - Append if existing running, else replace

## Work States

### Observe Work State

```kotlin
// Using LiveData
WorkManager.getInstance(context)
    .getWorkInfoByIdLiveData(workRequest.id)
    .observe(lifecycleOwner) { workInfo ->
        when (workInfo?.state) {
            WorkInfo.State.ENQUEUED -> {
                // Work is queued
            }
            WorkInfo.State.RUNNING -> {
                // Work is running
            }
            WorkInfo.State.SUCCEEDED -> {
                // Work completed successfully
            }
            WorkInfo.State.FAILED -> {
                // Work failed
            }
            WorkInfo.State.BLOCKED -> {
                // Work is blocked (dependencies not met)
            }
            WorkInfo.State.CANCELLED -> {
                // Work was cancelled
            }
            else -> {}
        }
    }

// Using Flow
WorkManager.getInstance(context)
    .getWorkInfoByIdFlow(workRequest.id)
    .collect { workInfo ->
        // Handle state
    }
```

## Cancel Work

### Cancel Work

```kotlin
// Cancel by ID
WorkManager.getInstance(context)
    .cancelWorkById(workRequest.id)

// Cancel by unique name
WorkManager.getInstance(context)
    .cancelUniqueWork("sync_work")

// Cancel by tag
WorkManager.getInstance(context)
    .cancelAllWorkByTag("sync_tag")

// Cancel all work
WorkManager.getInstance(context)
    .cancelAllWork()
```

## Hilt Integration

### Setup HiltWorker

```kotlin
import androidx.hilt.work.HiltWorker
import dagger.assisted.Assisted
import dagger.assisted.AssistedInject

@HiltWorker
class SyncDataWorker @AssistedInject constructor(
    @Assisted appContext: Context,
    @Assisted workerParams: WorkerParameters,
    private val repository: UserRepository
) : CoroutineWorker(appContext, workerParams) {
    
    override suspend fun doWork(): Result {
        return try {
            repository.syncData()
            Result.success()
        } catch (e: Exception) {
            Result.failure()
        }
    }
}
```

### Configure HiltWorkerFactory

```kotlin
import androidx.hilt.work.HiltWorkerFactory
import androidx.work.Configuration
import dagger.hilt.android.HiltAndroidApp
import javax.inject.Inject

@HiltAndroidApp
class MyApplication : Application(), Configuration.Provider {
    
    @Inject
    lateinit var workerFactory: HiltWorkerFactory
    
    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

Remove default WorkManager initialization in `AndroidManifest.xml`:

```xml
<application>
    <provider
        android:name="androidx.startup.InitializationProvider"
        android:authorities="${applicationId}.androidx-startup"
        android:exported="false"
        tools:node="merge">
        <meta-data
            android:name="androidx.work.WorkManagerInitializer"
            android:value="androidx.startup"
            tools:node="remove" />
    </provider>
</application>
```

## Advanced Features

### Exponential Backoff

```kotlin
import androidx.work.BackoffPolicy
import java.util.concurrent.TimeUnit

val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setBackoffCriteria(
        BackoffPolicy.EXPONENTIAL,
        10,                           // Initial delay
        TimeUnit.SECONDS
    )
    .build()
```

### Initial Delay

```kotlin
val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .setInitialDelay(30, TimeUnit.SECONDS)
    .build()
```

### Add Tags

```kotlin
val workRequest = OneTimeWorkRequestBuilder<SyncDataWorker>()
    .addTag("sync")
    .addTag("important")
    .build()

// Query by tag
val workInfos = WorkManager.getInstance(context)
    .getWorkInfosByTag("sync")
```

### Expedited Work

```kotlin
import androidx.work.ExpeditedWorkPolicy

val workRequest = OneTimeWorkRequestBuilder<UrgentWorker>()
    .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
    .build()
```

## Foreground Work

### Long Running Worker

```kotlin
import androidx.work.ForegroundInfo
import androidx.core.app.NotificationCompat

class DownloadWorker(...) : CoroutineWorker(...) {
    
    override suspend fun doWork(): Result {
        setForeground(createForegroundInfo())
        
        // Do long running work
        
        return Result.success()
    }
    
    private fun createForegroundInfo(): ForegroundInfo {
        val notification = NotificationCompat.Builder(
            applicationContext,
            CHANNEL_ID
        )
            .setContentTitle("Downloading")
            .setContentText("Download in progress")
            .setSmallIcon(R.drawable.ic_download)
            .setOngoing(true)
            .build()
        
        return ForegroundInfo(NOTIFICATION_ID, notification)
    }
    
    companion object {
        const val NOTIFICATION_ID = 1
        const val CHANNEL_ID = "download_channel"
    }
}
```

## Testing

### Test Worker

```kotlin
import androidx.work.testing.TestListenableWorkerBuilder
import kotlinx.coroutines.test.runTest
import org.junit.Test

class SyncDataWorkerTest {
    
    private lateinit var context: Context
    
    @Before
    fun setup() {
        context = ApplicationProvider.getApplicationContext()
    }
    
    @Test
    fun testSyncDataWorker() = runTest {
        val worker = TestListenableWorkerBuilder<SyncDataWorker>(context)
            .setInputData(workDataOf("user_id" to 123))
            .build()
        
        val result = worker.doWork()
        
        assertThat(result).isInstanceOf(Result.Success::class.java)
    }
}
```

### Test WorkManager

```kotlin
import androidx.work.testing.WorkManagerTestInitHelper
import androidx.work.testing.TestDriver

class WorkManagerTest {
    
    @Before
    fun setup() {
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(Log.DEBUG)
            .build()
        
        WorkManagerTestInitHelper.initializeTestWorkManager(context, config)
    }
    
    @Test
    fun testPeriodicWork() {
        val request = PeriodicWorkRequestBuilder<SyncDataWorker>(15, TimeUnit.MINUTES)
            .build()
        
        val workManager = WorkManager.getInstance(context)
        val testDriver = WorkManagerTestInitHelper.getTestDriver(context)!!
        
        workManager.enqueue(request).result.get()
        
        // Trigger periodic work
        testDriver.setPeriodDelayMet(request.id)
        
        val workInfo = workManager.getWorkInfoById(request.id).get()
        assertThat(workInfo.state).isEqualTo(WorkInfo.State.ENQUEUED)
    }
}
```

## Best Practices

1. ✅ Use WorkManager for deferrable, guaranteed work
2. ✅ Set appropriate constraints
3. ✅ Handle retry logic properly
4. ✅ Use unique work names to avoid duplicates
5. ✅ Observe work state for UI updates
6. ✅ Keep workers lightweight (< 10 minutes)
7. ✅ Use foreground service for long operations
8. ✅ Test workers thoroughly
9. ✅ Don't use for real-time operations
10. ✅ Clean up work when no longer needed

## Common Use Cases

### Background Sync

```kotlin
class DataSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: DataRepository
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            repository.syncWithServer()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                Result.failure()
            }
        }
    }
}
```

### Image Upload

```kotlin
class ImageUploadWorker(...) : CoroutineWorker(...) {
    override suspend fun doWork(): Result {
        val imageUri = inputData.getString("image_uri") ?: return Result.failure()
        
        return try {
            val file = File(imageUri)
            uploadImage(file)
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}
```

### Database Cleanup

```kotlin
val cleanupWork = PeriodicWorkRequestBuilder<DatabaseCleanupWorker>(
    1, TimeUnit.DAYS
)
    .setConstraints(
        Constraints.Builder()
            .setRequiresBatteryNotLow(true)
            .setRequiresDeviceIdle(true)
            .build()
    )
    .build()

WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork(
        "database_cleanup",
        ExistingPeriodicWorkPolicy.KEEP,
        cleanupWork
    )
```

## Resources

- [Official Documentation](https://developer.android.com/topic/libraries/architecture/workmanager)
- [WorkManager Codelab](https://developer.android.com/codelabs/android-workmanager)
- [Best Practices](https://developer.android.com/topic/libraries/architecture/workmanager/advanced)

