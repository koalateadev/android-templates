# Turbine Setup

## Overview
Turbine is a small testing library for Kotlin Flow that makes it easy to assert emissions from a Flow in your tests.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
turbine = "1.1.0"
coroutines-test = "1.9.0"

[libraries]
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines-test" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    testImplementation(libs.turbine)
    testImplementation(libs.kotlinx.coroutines.test)
}
```

## Basic Usage

### Simple Flow Test

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.test.runTest
import org.junit.Test
import kotlin.test.assertEquals

class FlowTest {
    
    @Test
    fun `test simple flow emissions`() = runTest {
        val flow = flow {
            emit(1)
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
}
```

### Testing StateFlow

```kotlin
import app.cash.turbine.test
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.test.runTest
import org.junit.Test

class StateFlowTest {
    
    @Test
    fun `test StateFlow updates`() = runTest {
        val stateFlow = MutableStateFlow(0)
        
        stateFlow.test {
            assertEquals(0, awaitItem()) // Initial value
            
            stateFlow.value = 1
            assertEquals(1, awaitItem())
            
            stateFlow.value = 2
            assertEquals(2, awaitItem())
            
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Testing ViewModel

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count
    
    fun increment() {
        viewModelScope.launch {
            _count.value++
        }
    }
    
    fun decrement() {
        viewModelScope.launch {
            _count.value--
        }
    }
}

// Test
import app.cash.turbine.test
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.StandardTestDispatcher
import kotlinx.coroutines.test.resetMain
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.setMain
import org.junit.After
import org.junit.Before
import org.junit.Test

@OptIn(ExperimentalCoroutinesApi::class)
class CounterViewModelTest {
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var viewModel: CounterViewModel
    
    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        viewModel = CounterViewModel()
    }
    
    @After
    fun teardown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `test counter increment`() = runTest(testDispatcher) {
        viewModel.count.test {
            assertEquals(0, awaitItem()) // Initial value
            
            viewModel.increment()
            assertEquals(1, awaitItem())
            
            viewModel.increment()
            assertEquals(2, awaitItem())
            
            cancelAndIgnoreRemainingEvents()
        }
    }
    
    @Test
    fun `test counter decrement`() = runTest(testDispatcher) {
        viewModel.count.test {
            skipItems(1) // Skip initial value
            
            viewModel.decrement()
            assertEquals(-1, awaitItem())
            
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

## Advanced Usage

### Testing Repository with Flow

```kotlin
class UserRepository(private val apiService: ApiService) {
    fun getUsers(): Flow<Result<List<User>>> = flow {
        emit(Result.Loading)
        try {
            val response = apiService.getUsers()
            if (response.isSuccessful && response.body() != null) {
                emit(Result.Success(response.body()!!))
            } else {
                emit(Result.Error("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            emit(Result.Error(e.message ?: "Unknown error"))
        }
    }
}

sealed interface Result<out T> {
    data object Loading : Result<Nothing>
    data class Success<T>(val data: T) : Result<T>
    data class Error(val message: String) : Result<Nothing>
}

// Test
@Test
fun `test repository success flow`() = runTest {
    // Mock API response
    val mockUsers = listOf(User(1, "John", "john@example.com"))
    coEvery { apiService.getUsers() } returns Response.success(mockUsers)
    
    repository.getUsers().test {
        // First emission should be Loading
        assertTrue(awaitItem() is Result.Loading)
        
        // Second emission should be Success with data
        val success = awaitItem()
        assertTrue(success is Result.Success)
        assertEquals(mockUsers, (success as Result.Success).data)
        
        awaitComplete()
    }
}

@Test
fun `test repository error flow`() = runTest {
    coEvery { apiService.getUsers() } throws IOException("Network error")
    
    repository.getUsers().test {
        assertTrue(awaitItem() is Result.Loading)
        
        val error = awaitItem()
        assertTrue(error is Result.Error)
        assertTrue((error as Result.Error).message.contains("Network error"))
        
        awaitComplete()
    }
}
```

### Testing with Delays

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.flow

@Test
fun `test flow with delays`() = runTest {
    val flow = flow {
        emit(1)
        delay(1000)
        emit(2)
        delay(1000)
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

### Testing Error Handling

```kotlin
@Test
fun `test flow error`() = runTest {
    val flow = flow<Int> {
        emit(1)
        throw IllegalStateException("Something went wrong")
    }
    
    flow.test {
        assertEquals(1, awaitItem())
        val error = awaitError()
        assertTrue(error is IllegalStateException)
        assertEquals("Something went wrong", error.message)
    }
}
```

### Testing Multiple Flows

```kotlin
@Test
fun `test multiple flows combined`() = runTest {
    val flow1 = MutableStateFlow(1)
    val flow2 = MutableStateFlow("A")
    
    combine(flow1, flow2) { num, str -> "$str$num" }.test {
        assertEquals("A1", awaitItem())
        
        flow1.value = 2
        assertEquals("A2", awaitItem())
        
        flow2.value = "B"
        assertEquals("B2", awaitItem())
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Testing SharedFlow

```kotlin
import kotlinx.coroutines.flow.MutableSharedFlow

@Test
fun `test SharedFlow emissions`() = runTest {
    val sharedFlow = MutableSharedFlow<String>()
    
    sharedFlow.test {
        sharedFlow.emit("Hello")
        assertEquals("Hello", awaitItem())
        
        sharedFlow.emit("World")
        assertEquals("World", awaitItem())
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Common Turbine Methods

| Method | Description |
|--------|-------------|
| `awaitItem()` | Wait for and return next emission |
| `awaitComplete()` | Assert flow completes successfully |
| `awaitError()` | Wait for and return error |
| `skipItems(count)` | Skip specified number of items |
| `expectNoEvents()` | Assert no events are emitted |
| `cancelAndIgnoreRemainingEvents()` | Cancel and cleanup |
| `expectMostRecentItem()` | Get most recent emission |

## Testing Patterns

### Test ViewModel with Loading States

```kotlin
sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: String) : UiState
    data class Error(val message: String) : UiState
}

class DataViewModel(private val repository: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState
    
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            repository.getData()
                .onSuccess { _uiState.value = UiState.Success(it) }
                .onFailure { _uiState.value = UiState.Error(it.message ?: "Error") }
        }
    }
}

@Test
fun `test loading states`() = runTest(testDispatcher) {
    coEvery { repository.getData() } returns Result.success("Test Data")
    
    viewModel.uiState.test {
        // Initial state
        assertTrue(awaitItem() is UiState.Loading)
        
        viewModel.loadData()
        
        // Loading state
        assertTrue(awaitItem() is UiState.Loading)
        
        // Success state
        val success = awaitItem()
        assertTrue(success is UiState.Success)
        assertEquals("Test Data", (success as UiState.Success).data)
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Testing Debounced Search

```kotlin
import kotlinx.coroutines.FlowPreview
import kotlinx.coroutines.flow.debounce

@OptIn(FlowPreview::class)
class SearchViewModel : ViewModel() {
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery
    
    val searchResults = _searchQuery
        .debounce(300)
        .filter { it.isNotBlank() }
        .map { query -> performSearch(query) }
    
    fun updateQuery(query: String) {
        _searchQuery.value = query
    }
}

@Test
fun `test debounced search`() = runTest(testDispatcher) {
    viewModel.searchResults.test {
        viewModel.updateQuery("a")
        viewModel.updateQuery("an")
        viewModel.updateQuery("and")
        
        // Only final query after debounce
        advanceTimeBy(300)
        assertEquals(searchResults("and"), awaitItem())
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Best Practices

- Always call awaitComplete() or cancelAndIgnoreRemainingEvents()
- Use runTest for coroutine tests
- Set up TestDispatcher for ViewModels
- Test all flow states
- Use skipItems() to ignore initial values
- Clean up properly in teardown

## Common Pitfalls

**Common mistakes:**
- Forgetting to collect: Flow tests must be inside .test {} block
- Not using TestDispatcher: ViewModels need proper test dispatcher setup
- Not completing tests: Always end with awaitComplete() or cancel
- Testing hot flows incorrectly: StateFlow always has a value

## Resources

- [Official Documentation](https://github.com/cashapp/turbine)
- [Kotlin Coroutines Test Guide](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)

