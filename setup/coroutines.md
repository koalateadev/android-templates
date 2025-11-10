# Kotlin Coroutines Setup

## Overview
Kotlin Coroutines provide a way to write asynchronous code that is sequential, making it easier to handle long-running tasks without blocking the main thread.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
coroutines = "1.9.0"

[libraries]
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }

[bundles]
coroutines = ["kotlinx-coroutines-core", "kotlinx-coroutines-android"]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.coroutines)
    
    // For testing
    testImplementation(libs.kotlinx.coroutines.test)
}
```

## Basic Usage

### Launching Coroutines

```kotlin
import kotlinx.coroutines.*

// In ViewModel
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            // Automatically cancelled when ViewModel is cleared
            val data = repository.getData()
            updateUI(data)
        }
    }
}

// In Activity/Fragment
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch {
            // Automatically cancelled when Activity is destroyed
            val result = performNetworkCall()
        }
    }
}

// Custom scope
fun loadData() {
    CoroutineScope(Dispatchers.IO).launch {
        val data = fetchData()
        withContext(Dispatchers.Main) {
            updateUI(data)
        }
    }
}
```

### Dispatchers

```kotlin
// Main - UI operations (default for lifecycleScope/viewModelScope)
withContext(Dispatchers.Main) {
    textView.text = "Updated"
}

// IO - Network, database, file operations
withContext(Dispatchers.IO) {
    val data = database.query()
}

// Default - CPU-intensive work
withContext(Dispatchers.Default) {
    val result = complexCalculation()
}

// Unconfined - Not recommended for production
```

### Suspend Functions

```kotlin
// Suspend function
suspend fun fetchUser(userId: Long): User {
    return withContext(Dispatchers.IO) {
        apiService.getUser(userId)
    }
}

// Calling from coroutine
viewModelScope.launch {
    val user = fetchUser(123)
    println(user.name)
}

// Multiple sequential calls
suspend fun loadUserData(userId: Long): UserData {
    val user = fetchUser(userId)
    val posts = fetchPosts(userId)
    val friends = fetchFriends(userId)
    return UserData(user, posts, friends)
}
```

## Parallel Execution

### async and await

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    // Start all operations in parallel
    val userDeferred = async { fetchUser() }
    val statsDeferred = async { fetchStats() }
    val postsDeferred = async { fetchPosts() }
    
    // Wait for all to complete
    Dashboard(
        user = userDeferred.await(),
        stats = statsDeferred.await(),
        posts = postsDeferred.await()
    )
}
```

### Structured Concurrency

```kotlin
suspend fun processUsers(userIds: List<Long>) = coroutineScope {
    // If any fails, all are cancelled
    userIds.map { userId ->
        async {
            processUser(userId)
        }
    }.awaitAll()
}

// With exception handling
suspend fun loadData() = coroutineScope {
    try {
        val results = listOf(
            async { operation1() },
            async { operation2() },
            async { operation3() }
        ).awaitAll()
        
        results
    } catch (e: Exception) {
        // Handle error
        emptyList()
    }
}
```

## Error Handling

### Try-Catch

```kotlin
viewModelScope.launch {
    try {
        val data = repository.fetchData()
        _uiState.value = UiState.Success(data)
    } catch (e: IOException) {
        _uiState.value = UiState.Error("Network error")
    } catch (e: Exception) {
        _uiState.value = UiState.Error("Unknown error")
    }
}
```

### CoroutineExceptionHandler

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught $exception")
}

val scope = CoroutineScope(Dispatchers.Main + handler)

scope.launch {
    throw RuntimeException("Error!")
}
```

### SupervisorJob

```kotlin
// Regular Job - one failure cancels all siblings
val scope = CoroutineScope(Job())

scope.launch {
    launch { throw Exception("Error 1") } // Cancels sibling
    launch { delay(5000); println("Never prints") }
}

// SupervisorJob - failures are independent
val supervisorScope = CoroutineScope(SupervisorJob())

supervisorScope.launch {
    launch { throw Exception("Error 1") } // Doesn't affect sibling
    launch { delay(1000); println("Still prints") }
}
```

## Cancellation

### Cooperative Cancellation

```kotlin
// Check if cancelled
suspend fun processItems(items: List<Item>) {
    for (item in items) {
        if (!isActive) return // Exit if cancelled
        processItem(item)
    }
}

// Using ensureActive()
suspend fun processItems(items: List<Item>) {
    for (item in items) {
        ensureActive() // Throws if cancelled
        processItem(item)
    }
}

// Cancellable operation
withContext(Dispatchers.IO) {
    while (isActive) {
        // Work that can be cancelled
    }
}
```

### Timeout

```kotlin
// Timeout with exception
suspend fun fetchWithTimeout(): Data {
    return withTimeout(5000) {
        apiService.fetchData()
    }
}

// Timeout returning null
suspend fun fetchWithTimeout(): Data? {
    return withTimeoutOrNull(5000) {
        apiService.fetchData()
    }
}
```

### Cancellation Cleanup

```kotlin
suspend fun downloadFile() {
    try {
        // Download operations
    } finally {
        withContext(NonCancellable) {
            // Cleanup that must complete
            closeResources()
        }
    }
}
```

## Repository Pattern

### With Coroutines

```kotlin
class UserRepository(
    private val apiService: ApiService,
    private val userDao: UserDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    // Single data fetch
    suspend fun getUser(userId: Long): Result<User> {
        return withContext(ioDispatcher) {
            try {
                val response = apiService.getUser(userId)
                if (response.isSuccessful && response.body() != null) {
                    Result.success(response.body()!!)
                } else {
                    Result.failure(Exception("Error: ${response.code()}"))
                }
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
    
    // Cache with database
    suspend fun getUserCached(userId: Long): User {
        return withContext(ioDispatcher) {
            // Try database first
            userDao.getUserById(userId) ?: run {
                // Fetch from network
                val user = apiService.getUser(userId).body()
                    ?: throw Exception("User not found")
                // Save to database
                userDao.insertUser(user)
                user
            }
        }
    }
    
    // Parallel fetching
    suspend fun getUserWithDetails(userId: Long): UserDetails = coroutineScope {
        val userDeferred = async { getUser(userId) }
        val postsDeferred = async { getPosts(userId) }
        val friendsDeferred = async { getFriends(userId) }
        
        UserDetails(
            user = userDeferred.await().getOrThrow(),
            posts = postsDeferred.await().getOrThrow(),
            friends = friendsDeferred.await().getOrThrow()
        )
    }
}
```

## Advanced Patterns

### Retry Logic

```kotlin
suspend fun <T> retryIO(
    times: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
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
suspend fun fetchData(): Data {
    return retryIO(times = 3) {
        apiService.getData()
    }
}
```

### Debounce

```kotlin
fun <T> Flow<T>.debounce(timeoutMillis: Long): Flow<T> {
    return this.debounce(timeoutMillis)
}

// Usage in ViewModel
val searchResults = searchQuery
    .debounce(300)
    .filter { it.isNotBlank() }
    .flatMapLatest { query ->
        repository.search(query)
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
```

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

### Channels for Communication

```kotlin
class DataProducer {
    private val channel = Channel<Data>(Channel.UNLIMITED)
    
    fun start() = CoroutineScope(Dispatchers.IO).launch {
        repeat(10) {
            val data = fetchData()
            channel.send(data)
        }
        channel.close()
    }
    
    suspend fun consume() {
        for (data in channel) {
            processData(data)
        }
    }
}
```

## Testing

### Basic Test

```kotlin
import kotlinx.coroutines.test.*
import org.junit.Test

@OptIn(ExperimentalCoroutinesApi::class)
class RepositoryTest {
    
    @Test
    fun `fetch user returns success`() = runTest {
        // Arrange
        val repository = UserRepository(mockApiService, mockDao)
        
        // Act
        val result = repository.getUser(123)
        
        // Assert
        assertTrue(result.isSuccess)
    }
}
```

### With Dispatcher

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.After
import org.junit.Before

@OptIn(ExperimentalCoroutinesApi::class)
class ViewModelTest {
    private val testDispatcher = StandardTestDispatcher()
    
    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
    }
    
    @After
    fun teardown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `test coroutine`() = runTest {
        // Test code
        viewModel.loadData()
        advanceUntilIdle()
        
        // Assertions
    }
}
```

### Test with Delays

```kotlin
@Test
fun `test with delay`() = runTest {
    val result = async {
        delay(1000)
        "Done"
    }
    
    advanceTimeBy(1000)
    assertEquals("Done", result.await())
}
```

## Best Practices

- Use structured concurrency (viewModelScope, lifecycleScope)
- Use appropriate dispatchers
- Handle cancellation properly
- Use supervisorScope for independent operations
- Avoid GlobalScope
- Use withContext for switching dispatchers
- Prefer Flow over callbacks
- Handle errors appropriately
- Use timeout for network operations

## Common Mistakes

**Blocking the main thread:**
```kotlin
// Bad
runBlocking {
    val data = fetchData()
}

// Good
viewModelScope.launch {
    val data = fetchData()
}
```

**Not handling cancellation:**
```kotlin
// Bad
while (true) {
    work()
}

// Good
while (isActive) {
    work()
}
```

**Using GlobalScope:**
```kotlin
// Bad
GlobalScope.launch {
    // Can leak
}

// Good
viewModelScope.launch {
    // Lifecycle-aware
}
```

## Resources

- [Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Android Coroutines](https://developer.android.com/kotlin/coroutines)
- [Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

