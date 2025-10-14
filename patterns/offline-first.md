# Offline-First Patterns

## Table of Contents
- [Network + Database Strategy](#network--database-strategy)
- [Connection Monitoring](#connection-monitoring)
- [Sync Patterns](#sync-patterns)
- [Conflict Resolution](#conflict-resolution)
- [Queue-Based Sync](#queue-based-sync)
- [Optimistic Updates](#optimistic-updates)

## Network + Database Strategy

### Basic Offline-First Repository

```kotlin
class OfflineFirstRepository(
    private val apiService: ApiService,
    private val dao: UserDao,
    private val networkMonitor: NetworkMonitor
) {
    fun getUsers(): Flow<Result<List<User>>> = flow {
        // Emit cached data immediately
        val cachedUsers = dao.getAllUsersOnce()
        if (cachedUsers.isNotEmpty()) {
            emit(Result.Success(cachedUsers, fromCache = true))
        } else {
            emit(Result.Loading)
        }
        
        // Fetch from network if connected
        if (networkMonitor.isConnected()) {
            try {
                val networkUsers = apiService.getUsers()
                dao.insertAll(networkUsers)
                emit(Result.Success(networkUsers, fromCache = false))
            } catch (e: Exception) {
                if (cachedUsers.isEmpty()) {
                    emit(Result.Error(e.message ?: "Network error"))
                }
                // If we have cache, silently fail network request
            }
        }
    }.flowOn(Dispatchers.IO)
}

sealed class Result<out T> {
    data object Loading : Result<Nothing>()
    data class Success<T>(val data: T, val fromCache: Boolean = false) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
}
```

### Cache with Expiration

```kotlin
@Entity(tableName = "cached_data")
data class CachedData(
    @PrimaryKey val id: String,
    val data: String,
    val cachedAt: Long = System.currentTimeMillis()
)

class CachedRepository(
    private val apiService: ApiService,
    private val dao: CachedDataDao
) {
    private val cacheValidityMs = 5 * 60 * 1000 // 5 minutes
    
    suspend fun getData(id: String, forceRefresh: Boolean = false): Result<Data> {
        // Check cache
        val cached = dao.getCachedData(id)
        val isCacheValid = cached != null && 
            System.currentTimeMillis() - cached.cachedAt < cacheValidityMs
        
        if (isCacheValid && !forceRefresh) {
            return Result.Success(cached.data.fromJson())
        }
        
        // Fetch from network
        return try {
            val networkData = apiService.getData(id)
            dao.insertCachedData(
                CachedData(
                    id = id,
                    data = networkData.toJson()
                )
            )
            Result.Success(networkData)
        } catch (e: Exception) {
            if (cached != null) {
                Result.Success(cached.data.fromJson())  // Return stale cache
            } else {
                Result.Error(e.message ?: "Error")
            }
        }
    }
}
```

## Connection Monitoring

### Network Monitor

```kotlin
import android.content.Context
import android.net.ConnectivityManager
import android.net.Network
import android.net.NetworkCapabilities
import android.net.NetworkRequest
import kotlinx.coroutines.channels.awaitClose
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.callbackFlow

class NetworkMonitor(private val context: Context) {
    
    private val connectivityManager = context.getSystemService(
        Context.CONNECTIVITY_SERVICE
    ) as ConnectivityManager
    
    fun isConnected(): Boolean {
        val network = connectivityManager.activeNetwork ?: return false
        val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false
        
        return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) &&
                capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
    }
    
    fun observeConnectivity(): Flow<Boolean> = callbackFlow {
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
            }
            
            override fun onLost(network: Network) {
                trySend(false)
            }
            
            override fun onCapabilitiesChanged(
                network: Network,
                networkCapabilities: NetworkCapabilities
            ) {
                val connected = networkCapabilities.hasCapability(
                    NetworkCapabilities.NET_CAPABILITY_VALIDATED
                )
                trySend(connected)
            }
        }
        
        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        
        connectivityManager.registerNetworkCallback(request, callback)
        
        // Send initial state
        trySend(isConnected())
        
        awaitClose {
            connectivityManager.unregisterNetworkCallback(callback)
        }
    }
}
```

### Use in Composable

```kotlin
@Composable
fun NetworkAwareScreen(networkMonitor: NetworkMonitor) {
    val isConnected by networkMonitor.observeConnectivity()
        .collectAsState(initial = true)
    
    Scaffold(
        topBar = {
            if (!isConnected) {
                Card(
                    colors = CardDefaults.cardColors(
                        containerColor = MaterialTheme.colorScheme.error
                    )
                ) {
                    Row(
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(8.dp),
                        horizontalArrangement = Arrangement.Center
                    ) {
                        Icon(Icons.Default.CloudOff, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("No internet connection")
                    }
                }
            }
        }
    ) { paddingValues ->
        Content(modifier = Modifier.padding(paddingValues))
    }
}
```

## Sync Patterns

### Background Sync with WorkManager

```kotlin
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncRepository: SyncRepository
) : CoroutineWorker(context, params) {
    
    override suspend fun doWork(): Result {
        return try {
            syncRepository.syncPendingChanges()
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

// Schedule sync
class SyncScheduler(private val context: Context) {
    
    fun schedulePeriodic() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
        
        val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            15, TimeUnit.MINUTES
        )
            .setConstraints(constraints)
            .build()
        
        WorkManager.getInstance(context)
            .enqueueUniquePeriodicWork(
                "periodic_sync",
                ExistingPeriodicWorkPolicy.KEEP,
                syncRequest
            )
    }
    
    fun syncNow() {
        val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
            .build()
        
        WorkManager.getInstance(context).enqueue(syncRequest)
    }
}
```

### Incremental Sync

```kotlin
class IncrementalSyncRepository(
    private val apiService: ApiService,
    private val dao: UserDao,
    private val syncStateDao: SyncStateDao
) {
    suspend fun sync() {
        val lastSyncTime = syncStateDao.getLastSyncTime()
        
        // Fetch only changes since last sync
        val changes = apiService.getChangesSince(lastSyncTime)
        
        changes.forEach { change ->
            when (change.type) {
                ChangeType.INSERT -> dao.insert(change.data)
                ChangeType.UPDATE -> dao.update(change.data)
                ChangeType.DELETE -> dao.delete(change.data.id)
            }
        }
        
        syncStateDao.updateLastSyncTime(System.currentTimeMillis())
    }
}
```

## Conflict Resolution

### Last Write Wins

```kotlin
data class SyncableEntity(
    val id: String,
    val data: String,
    val version: Long,
    val updatedAt: Long
)

class ConflictResolver {
    fun resolve(
        local: SyncableEntity,
        remote: SyncableEntity
    ): SyncableEntity {
        // Last write wins
        return if (local.updatedAt > remote.updatedAt) {
            local
        } else {
            remote
        }
    }
}
```

### Version-Based Resolution

```kotlin
class VersionConflictResolver(
    private val dao: EntityDao,
    private val apiService: ApiService
) {
    suspend fun sync(entity: SyncableEntity) {
        try {
            // Try to update with version check
            apiService.updateEntity(entity)
            dao.update(entity)
        } catch (e: ConflictException) {
            // Server has newer version
            val serverEntity = apiService.getEntity(entity.id)
            
            // Merge changes or prompt user
            val resolved = mergeChanges(entity, serverEntity)
            
            dao.update(resolved)
            apiService.updateEntity(resolved)
        }
    }
    
    private fun mergeChanges(
        local: SyncableEntity,
        remote: SyncableEntity
    ): SyncableEntity {
        // Custom merge logic
        return remote.copy(version = remote.version + 1)
    }
}
```

## Queue-Based Sync

### Sync Queue

```kotlin
@Entity(tableName = "sync_queue")
data class SyncQueueItem(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val entityType: String,
    val entityId: String,
    val operation: Operation,
    val data: String,
    val createdAt: Long = System.currentTimeMillis(),
    val retryCount: Int = 0
)

enum class Operation {
    CREATE, UPDATE, DELETE
}

class SyncQueueRepository(
    private val syncQueueDao: SyncQueueDao,
    private val apiService: ApiService
) {
    suspend fun enqueue(
        entityType: String,
        entityId: String,
        operation: Operation,
        data: String
    ) {
        val item = SyncQueueItem(
            entityType = entityType,
            entityId = entityId,
            operation = operation,
            data = data
        )
        syncQueueDao.insert(item)
    }
    
    suspend fun processQueue() {
        val items = syncQueueDao.getAllPending()
        
        items.forEach { item ->
            try {
                when (item.operation) {
                    Operation.CREATE -> apiService.create(item.data)
                    Operation.UPDATE -> apiService.update(item.entityId, item.data)
                    Operation.DELETE -> apiService.delete(item.entityId)
                }
                
                syncQueueDao.delete(item)
            } catch (e: Exception) {
                // Increment retry count
                syncQueueDao.update(
                    item.copy(retryCount = item.retryCount + 1)
                )
            }
        }
    }
}
```

## Optimistic Updates

### Optimistic UI Update

```kotlin
class OptimisticUpdateRepository(
    private val apiService: ApiService,
    private val dao: UserDao
) {
    suspend fun updateUser(user: User) {
        // Update local database immediately
        dao.update(user)
        
        try {
            // Sync with server
            val serverUser = apiService.updateUser(user.id, user)
            dao.update(serverUser)
        } catch (e: Exception) {
            // Revert on failure
            val originalUser = dao.getUserById(user.id)
            originalUser?.let { dao.update(it) }
            throw e
        }
    }
}
```

### Optimistic Delete

```kotlin
suspend fun deleteUserOptimistic(userId: String): Result<Unit> {
    // Get original for rollback
    val originalUser = dao.getUserById(userId)
    
    // Delete locally
    dao.deleteById(userId)
    
    return try {
        // Delete on server
        apiService.deleteUser(userId)
        Result.success(Unit)
    } catch (e: Exception) {
        // Rollback on failure
        originalUser?.let { dao.insert(it) }
        Result.failure(e)
    }
}
```

## ViewModel with Offline Support

```kotlin
class OfflineFirstViewModel(
    private val repository: OfflineFirstRepository,
    private val networkMonitor: NetworkMonitor
) : ViewModel() {
    
    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    val isOnline: StateFlow<Boolean> = networkMonitor.observeConnectivity()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = true
        )
    
    fun refresh() {
        viewModelScope.launch {
            if (isOnline.value) {
                repository.syncFromNetwork()
            }
        }
    }
    
    fun createUser(name: String, email: String) {
        viewModelScope.launch {
            val user = User(
                id = UUID.randomUUID().toString(),
                name = name,
                email = email,
                syncStatus = if (isOnline.value) SyncStatus.SYNCED else SyncStatus.PENDING
            )
            
            repository.insertUser(user)
            
            if (isOnline.value) {
                repository.syncUser(user)
            }
        }
    }
}

enum class SyncStatus {
    SYNCED, PENDING, FAILED
}
```

## Best Practices

1. ✅ Cache data locally (Room database)
2. ✅ Show cached data immediately
3. ✅ Update cache from network in background
4. ✅ Monitor network connectivity
5. ✅ Queue changes when offline
6. ✅ Sync when connection restored
7. ✅ Handle conflicts appropriately
8. ✅ Use optimistic updates for better UX
9. ✅ Show sync status to users
10. ✅ Implement proper error handling
11. ✅ Use WorkManager for background sync
12. ✅ Consider data retention policies
13. ✅ Handle partial sync failures
14. ✅ Test offline scenarios thoroughly
15. ✅ Provide manual sync option

## Resources

- [Offline-First Architecture](https://developer.android.com/topic/architecture/data-layer/offline-first)
- [WorkManager Guide](https://developer.android.com/topic/libraries/architecture/workmanager)
- [Room Database](https://developer.android.com/training/data-storage/room)
- [Network Connectivity](https://developer.android.com/training/monitoring-device-state/connectivity-status-type)

