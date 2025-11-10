# Flow & Coroutine Common Patterns

## Table of Contents
- [Flow Basics](#flow-basics)
- [StateFlow Patterns](#stateflow-patterns)
- [SharedFlow Patterns](#sharedflow-patterns)
- [Combining Flows](#combining-flows)
- [Error Handling](#error-handling)
- [Retry Logic](#retry-logic)
- [Caching](#caching)
- [Polling](#polling)
- [Debounce & Throttle](#debounce--throttle)
- [Repository Patterns](#repository-patterns)

## Flow Basics

### Simple Flow Creation

```kotlin
fun numbersFlow(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)
    }
}

// Usage
viewModelScope.launch {
    numbersFlow().collect { number ->
        println(number)
    }
}
```

### Flow from Callback

```kotlin
fun locationFlow(): Flow<Location> = callbackFlow {
    val locationCallback = object : LocationCallback() {
        override fun onLocationResult(result: LocationResult) {
            trySend(result.lastLocation)
        }
    }
    
    fusedLocationClient.requestLocationUpdates(
        locationRequest,
        locationCallback,
        Looper.getMainLooper()
    )
    
    awaitClose {
        fusedLocationClient.removeLocationUpdates(locationCallback)
    }
}
```

### Channel Flow

```kotlin
fun dataStream(): Flow<Data> = channelFlow {
    val listener = object : DataListener {
        override fun onDataReceived(data: Data) {
            trySend(data)
        }
    }
    
    dataSource.addListener(listener)
    
    awaitClose {
        dataSource.removeListener(listener)
    }
}
```

## StateFlow Patterns

### Basic StateFlow

```kotlin
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadData()
    }
    
    private fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = repository.getData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}

sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: List<Item>) : UiState
    data class Error(val message: String) : UiState
}
```

### Update StateFlow

```kotlin
// Update with value
_uiState.value = UiState.Loading

// Update based on current value
_uiState.update { currentState ->
    if (currentState is UiState.Success) {
        currentState.copy(isRefreshing = true)
    } else {
        currentState
    }
}

// Update with function
_uiState.updateAndGet { currentState ->
    // Returns new state
    UiState.Success(newData)
}
```

### Complex State Management

```kotlin
data class ScreenState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val error: String? = null,
    val selectedItems: Set<String> = emptySet()
)

class ListViewModel : ViewModel() {
    private val _state = MutableStateFlow(ScreenState())
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    fun loadItems() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            repository.getItems()
                .onSuccess { items ->
                    _state.update { it.copy(
                        items = items,
                        isLoading = false
                    )}
                }
                .onFailure { error ->
                    _state.update { it.copy(
                        isLoading = false,
                        error = error.message
                    )}
                }
        }
    }
    
    fun refresh() {
        viewModelScope.launch {
            _state.update { it.copy(isRefreshing = true) }
            
            repository.refreshItems()
                .onSuccess { items ->
                    _state.update { it.copy(
                        items = items,
                        isRefreshing = false
                    )}
                }
                .onFailure {
                    _state.update { it.copy(isRefreshing = false) }
                }
        }
    }
    
    fun toggleSelection(itemId: String) {
        _state.update { state ->
            val newSelection = if (itemId in state.selectedItems) {
                state.selectedItems - itemId
            } else {
                state.selectedItems + itemId
            }
            state.copy(selectedItems = newSelection)
        }
    }
}
```

## SharedFlow Patterns

### One-Time Events

```kotlin
class EventViewModel : ViewModel() {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    fun deleteItem(itemId: String) {
        viewModelScope.launch {
            try {
                repository.deleteItem(itemId)
                _events.emit(Event.ShowSnackbar("Item deleted"))
            } catch (e: Exception) {
                _events.emit(Event.ShowError(e.message ?: "Delete failed"))
            }
        }
    }
}

sealed interface Event {
    data class ShowSnackbar(val message: String) : Event
    data class ShowError(val message: String) : Event
    data class Navigate(val route: String) : Event
}

// Collect in Composable
@Composable
fun Screen(viewModel: EventViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is Event.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is Event.ShowError -> {
                    snackbarHostState.showSnackbar(
                        message = event.message,
                        duration = SnackbarDuration.Long
                    )
                }
                is Event.Navigate -> {
                    navController.navigate(event.route)
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // Content
    }
}
```

### Replay Events

```kotlin
// Replay last event for new subscribers
private val _events = MutableSharedFlow<Event>(
    replay = 1,
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

## Combining Flows

### Combine Multiple Flows

```kotlin
class DashboardViewModel(
    private val userRepository: UserRepository,
    private val statsRepository: StatsRepository,
    private val notificationRepository: NotificationRepository
) : ViewModel() {
    
    val dashboardState: StateFlow<DashboardState> = combine(
        userRepository.currentUser,
        statsRepository.stats,
        notificationRepository.unreadCount
    ) { user, stats, unreadCount ->
        DashboardState(
            user = user,
            stats = stats,
            unreadNotifications = unreadCount
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardState()
    )
}
```

### Zip Flows

```kotlin
val result = flow1.zip(flow2) { value1, value2 ->
    CombinedResult(value1, value2)
}
```

### Merge Flows

```kotlin
val mergedFlow = merge(
    repository1.dataFlow(),
    repository2.dataFlow(),
    repository3.dataFlow()
)
```

### FlatMapLatest (Switch Map)

```kotlin
class SearchViewModel : ViewModel() {
    private val _searchQuery = MutableStateFlow("")
    
    val searchResults = _searchQuery
        .debounce(300)
        .filter { it.length >= 2 }
        .flatMapLatest { query ->
            repository.search(query)
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    fun updateQuery(query: String) {
        _searchQuery.value = query
    }
}
```

## Error Handling

### Try-Catch in Flow

```kotlin
fun dataFlow(): Flow<Result<Data>> = flow {
    emit(Result.Loading)
    try {
        val data = apiService.getData()
        emit(Result.Success(data))
    } catch (e: IOException) {
        emit(Result.Error("Network error: ${e.message}"))
    } catch (e: Exception) {
        emit(Result.Error("Unknown error: ${e.message}"))
    }
}
```

### Catch Operator

```kotlin
val safeFlow = repository.dataFlow()
    .catch { exception ->
        emit(emptyList()) // Emit default value
        // Or
        emit(Result.Error(exception.message))
    }
```

### Retry on Error

```kotlin
val flowWithRetry = repository.dataFlow()
    .retry(retries = 3) { cause ->
        cause is IOException // Only retry on network errors
    }
```

## Retry Logic

### Simple Retry

```kotlin
suspend fun <T> retryIO(
    times: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) {
        try {
            return block()
        } catch (e: Exception) {
            // Log error
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
    }
    return block() // Last attempt
}

// Usage
suspend fun fetchData() = retryIO(times = 3) {
    apiService.getData()
}
```

### Flow Retry with Exponential Backoff

```kotlin
fun <T> Flow<T>.retryWithBackoff(
    maxRetries: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0
): Flow<T> = retryWhen { cause, attempt ->
    if (attempt >= maxRetries) {
        false
    } else {
        val delay = (initialDelay * factor.pow(attempt.toDouble())).toLong()
            .coerceAtMost(maxDelay)
        delay(delay)
        cause is IOException
    }
}

// Usage
val data = repository.dataFlow()
    .retryWithBackoff(maxRetries = 3)
```

## Caching

### In-Memory Cache

```kotlin
class CachedRepository(private val apiService: ApiService) {
    private var cachedData: List<Item>? = null
    private var cacheTime: Long = 0
    private val cacheTimeout = 5 * 60 * 1000 // 5 minutes
    
    suspend fun getItems(forceRefresh: Boolean = false): List<Item> {
        val currentTime = System.currentTimeMillis()
        
        return if (!forceRefresh &&
            cachedData != null &&
            currentTime - cacheTime < cacheTimeout
        ) {
            cachedData!!
        } else {
            val newData = apiService.getItems()
            cachedData = newData
            cacheTime = currentTime
            newData
        }
    }
}
```

### Flow Cache

```kotlin
class CachedFlowRepository(private val apiService: ApiService) {
    private var cachedFlow: Flow<List<Item>>? = null
    
    fun getItems(forceRefresh: Boolean = false): Flow<List<Item>> {
        if (cachedFlow == null || forceRefresh) {
            cachedFlow = flow {
                val data = apiService.getItems()
                emit(data)
            }.shareIn(
                scope = CoroutineScope(Dispatchers.IO),
                started = SharingStarted.WhileSubscribed(5000),
                replay = 1
            )
        }
        return cachedFlow!!
    }
}
```

## Polling

### Simple Polling

```kotlin
fun pollData(intervalMs: Long = 5000): Flow<Data> = flow {
    while (currentCoroutineContext().isActive) {
        val data = apiService.getData()
        emit(data)
        delay(intervalMs)
    }
}

// Usage
viewModelScope.launch {
    pollData(5000).collect { data ->
        updateUI(data)
    }
}
```

### Conditional Polling

```kotlin
fun conditionalPoll(
    condition: () -> Boolean,
    intervalMs: Long = 5000
): Flow<Data> = flow {
    while (currentCoroutineContext().isActive && condition()) {
        val data = apiService.getData()
        emit(data)
        delay(intervalMs)
    }
}
```

## Debounce & Throttle

### Debounce (Wait for Silence)

```kotlin
val searchResults = searchQuery
    .debounce(300) // Wait 300ms after last input
    .filter { it.isNotEmpty() }
    .flatMapLatest { query ->
        repository.search(query)
    }
```

### Throttle (Sample at Intervals)

```kotlin
val locationUpdates = locationFlow
    .sample(1000) // Emit latest every 1 second
```

### Distinct Until Changed

```kotlin
val uniqueValues = dataFlow
    .distinctUntilChanged() // Only emit when value changes
```

## Repository Patterns

### Network + Database Pattern

```kotlin
class UserRepository(
    private val apiService: ApiService,
    private val userDao: UserDao
) {
    fun getUsers(forceRefresh: Boolean = false): Flow<Result<List<User>>> = flow {
        // Emit loading state
        emit(Result.Loading)
        
        // Emit cached data if available
        val cachedUsers = userDao.getAllUsers()
        if (cachedUsers.isNotEmpty() && !forceRefresh) {
            emit(Result.Success(cachedUsers))
        }
        
        // Fetch from network
        try {
            val networkUsers = apiService.getUsers()
            userDao.insertAll(networkUsers)
            emit(Result.Success(networkUsers))
        } catch (e: Exception) {
            // If we have cached data, don't emit error
            if (cachedUsers.isEmpty()) {
                emit(Result.Error(e.message ?: "Unknown error"))
            }
        }
    }.flowOn(Dispatchers.IO)
}
```

### Room Flow Pattern

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserById(userId: String): Flow<User?>
}

class UserRepository(private val userDao: UserDao) {
    fun observeUsers(): Flow<List<User>> = userDao.getAllUsers()
    
    fun observeUser(userId: String): Flow<User?> = userDao.getUserById(userId)
}

// ViewModel
class UserViewModel(repository: UserRepository) : ViewModel() {
    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}
```

### Transform Flow Data

```kotlin
class TransformRepository(private val userDao: UserDao) {
    fun getActiveUsers(): Flow<List<UiUser>> = userDao.getAllUsers()
        .map { users ->
            users
                .filter { it.isActive }
                .map { it.toUiModel() }
        }
        .flowOn(Dispatchers.Default)
}
```

### Combine Network and Database

```kotlin
fun getUser(userId: String): Flow<User> = combine(
    userDao.getUserById(userId).filterNotNull(),
    userApi.observeUserStatus(userId)
) { user, status ->
    user.copy(onlineStatus = status)
}
```

## Advanced Patterns

### Rate Limiting

```kotlin
class RateLimiter(
    private val permits: Int,
    private val timeWindowMs: Long
) {
    private val timestamps = mutableListOf<Long>()
    
    suspend fun <T> execute(block: suspend () -> T): T {
        cleanOldTimestamps()
        
        while (timestamps.size >= permits) {
            val oldestTimestamp = timestamps.first()
            val delay = timeWindowMs - (System.currentTimeMillis() - oldestTimestamp)
            if (delay > 0) {
                delay(delay)
            }
            cleanOldTimestamps()
        }
        
        timestamps.add(System.currentTimeMillis())
        return block()
    }
    
    private fun cleanOldTimestamps() {
        val cutoff = System.currentTimeMillis() - timeWindowMs
        timestamps.removeAll { it < cutoff }
    }
}

// Usage
val limiter = RateLimiter(permits = 10, timeWindowMs = 1000)

suspend fun apiCall() {
    limiter.execute {
        apiService.getData()
    }
}
```

### Timeout Pattern

```kotlin
suspend fun fetchWithTimeout(timeoutMs: Long = 5000): Data {
    return withTimeout(timeoutMs) {
        apiService.getData()
    }
}

// Or with nullable return
suspend fun fetchWithTimeoutOrNull(timeoutMs: Long = 5000): Data? {
    return withTimeoutOrNull(timeoutMs) {
        apiService.getData()
    }
}
```

### Parallel Execution

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val userDeferred = async { fetchUser() }
    val statsDeferred = async { fetchStats() }
    val postsDeferred = async { fetchPosts() }
    
    Dashboard(
        user = userDeferred.await(),
        stats = statsDeferred.await(),
        posts = postsDeferred.await()
    )
}
```

## Best Practices

- Use StateFlow for UI state
- Use SharedFlow for one-time events
- Use flowOn to specify dispatcher
- Handle errors with try-catch or catch operator
- Cancel flows when no longer needed
- Use stateIn to convert Flow to StateFlow
- Collect flows in lifecycle-aware manner
- Test flows with Turbine
- Use debounce for search and user input

## Resources

- [Kotlin Flow Documentation](https://kotlinlang.org/docs/flow.html)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Flow Best Practices](https://developer.android.com/kotlin/flow/best-practices)

