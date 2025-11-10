# Kotlin Flow Setup

## Overview
Flow is a cold asynchronous stream that sequentially emits values and completes normally or with an exception. It's part of Kotlin Coroutines.

## Setup

Flow is included with Kotlin Coroutines:

```toml
[versions]
coroutines = "1.9.0"

[libraries]
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutines" }
```

## Basic Flow

### Creating Flows

```kotlin
import kotlinx.coroutines.flow.*

// Simple flow
fun simpleFlow(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

// From collection
val numbersFlow = listOf(1, 2, 3).asFlow()

// From channel
val channel = Channel<Int>()
val channelFlow = channel.consumeAsFlow()

// flowOf
val simpleFlow = flowOf(1, 2, 3)

// Empty flow
val emptyFlow = emptyFlow<Int>()
```

### Collecting Flows

```kotlin
// Basic collection
suspend fun collectExample() {
    simpleFlow().collect { value ->
        println(value)
    }
}

// In ViewModel
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            repository.dataFlow.collect { data ->
                // Handle data
            }
        }
    }
}

// Lifecycle-aware collection in Composable
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val data by viewModel.dataFlow.collectAsState(initial = emptyList())
    
    // Use data
}
```

## Flow Operators

### Transformation

```kotlin
// map - transform values
val squaredFlow = (1..5).asFlow().map { it * it }

// filter - filter values
val evenFlow = (1..10).asFlow().filter { it % 2 == 0 }

// transform - emit multiple values
val transformedFlow = (1..3).asFlow().transform { value ->
    emit("Start $value")
    delay(100)
    emit("End $value")
}

// flatMapConcat - sequential flattening
val flatFlow = (1..3).asFlow().flatMapConcat { value ->
    flow {
        emit("$value: First")
        emit("$value: Second")
    }
}

// flatMapMerge - concurrent flattening
val mergedFlow = (1..3).asFlow().flatMapMerge { value ->
    flow {
        delay(100)
        emit(value * 2)
    }
}

// flatMapLatest - cancel previous, emit latest
val latestFlow = searchQuery.flatMapLatest { query ->
    repository.search(query)
}
```

### Filtering & Limiting

```kotlin
// take - first n items
val firstThree = (1..10).asFlow().take(3)

// drop - skip first n items
val skipFirst = (1..10).asFlow().drop(3)

// distinctUntilChanged - skip duplicates
val distinct = flowOf(1, 1, 2, 2, 3).distinctUntilChanged()

// debounce - emit only after quiet period
val debounced = searchQuery.debounce(300)

// sample - emit latest at intervals
val sampled = flow.sample(1000)
```

### Combining Flows

```kotlin
// combine - combine latest from both
val combined = combine(flow1, flow2) { a, b ->
    "$a + $b"
}

// zip - pair corresponding values
val zipped = flow1.zip(flow2) { a, b ->
    "$a + $b"
}

// merge - merge emissions
val merged = merge(flow1, flow2, flow3)
```

## StateFlow

### Basic StateFlow

```kotlin
class MyViewModel : ViewModel() {
    // Mutable state (private)
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    
    // Immutable state (public)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = repository.getData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message ?: "Error")
            }
        }
    }
}

sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: Data) : UiState
    data class Error(val message: String) : UiState
}
```

### Update StateFlow

```kotlin
// Direct assignment
_count.value = 10

// Update based on current value
_count.update { it + 1 }

// Update with lambda
_uiState.update { currentState ->
    if (currentState is UiState.Success) {
        currentState.copy(isRefreshing = true)
    } else {
        currentState
    }
}
```

### Collect StateFlow

```kotlin
// In Composable
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> SuccessScreen((uiState as UiState.Success).data)
        is UiState.Error -> ErrorScreen((uiState as UiState.Error).message)
    }
}

// Lifecycle-aware collection
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}
```

## SharedFlow

### Basic SharedFlow

```kotlin
class EventViewModel : ViewModel() {
    // Hot flow for events
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events.asSharedFlow()
    
    fun triggerEvent(message: String) {
        viewModelScope.launch {
            _events.emit(Event.ShowSnackbar(message))
        }
    }
}

sealed interface Event {
    data class ShowSnackbar(val message: String) : Event
    data class Navigate(val route: String) : Event
}
```

### SharedFlow Configuration

```kotlin
// With replay and buffer
private val _events = MutableSharedFlow<Event>(
    replay = 1,        // Keep last 1 event for new subscribers
    extraBufferCapacity = 64,  // Buffer for slow collectors
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)
```

### Collect SharedFlow

```kotlin
@Composable
fun MyScreen(viewModel: EventViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is Event.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is Event.Navigate -> {
                    // Handle navigation
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // Content
    }
}
```

## Converting to StateFlow

### stateIn Operator

```kotlin
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    // Convert Flow to StateFlow
    val users: StateFlow<List<User>> = repository.getAllUsers()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
}

// SharingStarted strategies:
// - WhileSubscribed(stopTimeoutMillis) - active while there are subscribers
// - Eagerly - start immediately, never stop
// - Lazily - start on first subscriber, never stop
```

## Repository Patterns

### Room Database Flow

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getUserById(userId: Long): Flow<User?>
}

class UserRepository(private val userDao: UserDao) {
    val allUsers: Flow<List<User>> = userDao.getAllUsers()
    
    fun getUserById(id: Long): Flow<User?> = userDao.getUserById(id)
}
```

### Network + Cache Pattern

```kotlin
class UserRepository(
    private val apiService: ApiService,
    private val userDao: UserDao
) {
    fun getUsers(forceRefresh: Boolean = false): Flow<Result<List<User>>> = flow {
        // Emit loading
        emit(Result.Loading)
        
        // Emit cached data first
        val cachedUsers = userDao.getAllUsersOnce()
        if (cachedUsers.isNotEmpty() && !forceRefresh) {
            emit(Result.Success(cachedUsers))
        }
        
        // Fetch from network
        try {
            val response = apiService.getUsers()
            if (response.isSuccessful && response.body() != null) {
                val users = response.body()!!
                userDao.insertUsers(users)
                emit(Result.Success(users))
            } else {
                emit(Result.Error("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            if (cachedUsers.isEmpty()) {
                emit(Result.Error(e.message ?: "Unknown error"))
            }
        }
    }.flowOn(Dispatchers.IO)
}
```

### Combine Multiple Sources

```kotlin
class DashboardRepository(
    private val userDao: UserDao,
    private val statsDao: StatsDao
) {
    val dashboardData: Flow<DashboardData> = combine(
        userDao.getCurrentUser(),
        statsDao.getStats()
    ) { user, stats ->
        DashboardData(user, stats)
    }
}
```

## Advanced Patterns

### Retry with Exponential Backoff

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
        true
    }
}

// Usage
val dataFlow = repository.getData()
    .retryWithBackoff(maxRetries = 3)
```

### Caching Flow Results

```kotlin
class CachedFlowRepository {
    private var cachedFlow: Flow<List<Data>>? = null
    
    fun getData(forceRefresh: Boolean = false): Flow<List<Data>> {
        if (cachedFlow == null || forceRefresh) {
            cachedFlow = flow {
                val data = apiService.getData()
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

### Search with Debounce

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val repository: SearchRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val searchResults: StateFlow<List<SearchResult>> = _searchQuery
        .debounce(300)
        .filter { it.length >= 2 }
        .distinctUntilChanged()
        .flatMapLatest { query ->
            if (query.isEmpty()) {
                flowOf(emptyList())
            } else {
                repository.search(query)
                    .catch { emit(emptyList()) }
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )
    
    fun updateSearchQuery(query: String) {
        _searchQuery.value = query
    }
}
```

### Pagination

```kotlin
class PaginatedRepository {
    private var currentPage = 0
    
    fun getPagedData(): Flow<List<Item>> = flow {
        while (true) {
            val items = apiService.getItems(page = currentPage)
            if (items.isEmpty()) break
            
            emit(items)
            currentPage++
        }
    }
}

// In ViewModel
val pagedItems = repository.getPagedData()
    .scan(emptyList<Item>()) { accumulated, newItems ->
        accumulated + newItems
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
```

## Error Handling

```kotlin
// catch operator
val dataFlow = repository.getData()
    .catch { exception ->
        emit(emptyList()) // Emit default value
    }

// try-catch in collector
viewModelScope.launch {
    try {
        repository.getData().collect { data ->
            handleData(data)
        }
    } catch (e: Exception) {
        handleError(e)
    }
}

// onEach for side effects
val dataFlow = repository.getData()
    .onEach { data -> 
        println("Received: $data")
    }
    .catch { exception ->
        println("Error: $exception")
    }
```

## Testing Flows

### Basic Test

```kotlin
@Test
fun `test flow emissions`() = runTest {
    val flow = flowOf(1, 2, 3)
    
    val result = flow.toList()
    
    assertEquals(listOf(1, 2, 3), result)
}
```

### Test with Turbine

```kotlin
@Test
fun `test flow with turbine`() = runTest {
    val flow = flow {
        emit(1)
        delay(100)
        emit(2)
        emit(3)
    }
    
    flow.test {
        assertEquals(1, awaitItem())
        assertEquals(2, awaitItem())
        assertEquals(3, awaitItem())
        awaitComplete()
    }
}
```

## Best Practices

- Use StateFlow for UI state
- Use SharedFlow for one-time events
- Collect flows in lifecycle-aware manner
- Use appropriate operators (debounce, distinctUntilChanged)
- Handle errors with catch operator
- Use flowOn to specify dispatcher
- Convert Flow to StateFlow for UI layer
- Cancel flows when no longer needed

## Resources

- [Flow Documentation](https://kotlinlang.org/docs/flow.html)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)
- [Android Flow Guide](https://developer.android.com/kotlin/flow)

