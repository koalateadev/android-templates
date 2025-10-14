# ViewModel & Lifecycle Setup

## Overview
ViewModel stores and manages UI-related data in a lifecycle-conscious way, surviving configuration changes like screen rotations.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
lifecycle = "2.8.6"

[libraries]
androidx-lifecycle-viewmodel-ktx = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-ktx", version.ref = "lifecycle" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycle" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
androidx-lifecycle-livedata-ktx = { group = "androidx.lifecycle", name = "lifecycle-livedata-ktx", version.ref = "lifecycle" }

[bundles]
lifecycle = [
    "androidx-lifecycle-viewmodel-ktx",
    "androidx-lifecycle-runtime-ktx",
    "androidx-lifecycle-runtime-compose"
]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.lifecycle)
    
    // For Compose
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    
    // For LiveData (optional)
    implementation(libs.androidx.lifecycle.livedata.ktx)
}
```

## Basic ViewModel

### Simple ViewModel

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
        _count.value++
    }
    
    fun decrement() {
        _count.value--
    }
    
    override fun onCleared() {
        super.onCleared()
        // Cleanup resources
    }
}
```

### ViewModel with Repository

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadUsers()
    }
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            repository.getUsers()
                .onSuccess { users ->
                    _uiState.value = UiState.Success(users)
                }
                .onFailure { error ->
                    _uiState.value = UiState.Error(error.message ?: "Unknown error")
                }
        }
    }
    
    fun refreshUsers() {
        loadUsers()
    }
}

sealed interface UiState {
    data object Loading : UiState
    data class Success(val users: List<User>) : UiState
    data class Error(val message: String) : UiState
}
```

## Using ViewModel

### In Compose

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun UserScreen(
    viewModel: UserViewModel = viewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> {
            LoadingScreen()
        }
        is UiState.Success -> {
            UserList(users = (uiState as UiState.Success).users)
        }
        is UiState.Error -> {
            ErrorScreen(message = (uiState as UiState.Error).message)
        }
    }
}
```

### With Hilt

```kotlin
import androidx.lifecycle.ViewModel
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // ViewModel implementation
}

// In Composable
@Composable
fun UserScreen(
    viewModel: UserViewModel = hiltViewModel()
) {
    // UI implementation
}
```

### In Activity/Fragment

```kotlin
import androidx.activity.viewModels
import androidx.fragment.app.viewModels

class MainActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Collect state
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    // Update UI
                }
            }
        }
    }
}
```

## Lifecycle-Aware Collection

### collectAsStateWithLifecycle

```kotlin
import androidx.lifecycle.compose.collectAsStateWithLifecycle

@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    // Automatically stops collection when screen is in background
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (uiState) {
        // Handle states
    }
}
```

### repeatOnLifecycle in Activity

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    lifecycleScope.launch {
        // Automatically cancels when Activity is destroyed
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.uiState.collect { state ->
                updateUI(state)
            }
        }
    }
}
```

## StateFlow Patterns

### Combining Flows

```kotlin
class DashboardViewModel(
    private val userRepository: UserRepository,
    private val statsRepository: StatsRepository
) : ViewModel() {
    
    val dashboardState = combine(
        userRepository.currentUser,
        statsRepository.stats
    ) { user, stats ->
        DashboardState(user, stats)
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = DashboardState()
    )
}
```

### Transforming Flows

```kotlin
class SearchViewModel(
    private val repository: SearchRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val searchResults = _searchQuery
        .debounce(300)
        .filter { it.length >= 3 }
        .distinctUntilChanged()
        .flatMapLatest { query ->
            repository.search(query)
        }
        .catch { emit(emptyList()) }
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

### One-Shot Events (SharedFlow)

```kotlin
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.SharedFlow

class UserViewModel : ViewModel() {
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events
    
    fun deleteUser(userId: Long) {
        viewModelScope.launch {
            try {
                repository.deleteUser(userId)
                _events.emit(Event.ShowMessage("User deleted"))
            } catch (e: Exception) {
                _events.emit(Event.ShowError(e.message ?: "Error"))
            }
        }
    }
}

sealed interface Event {
    data class ShowMessage(val message: String) : Event
    data class ShowError(val error: String) : Event
}

// Collect in Composable
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is Event.ShowMessage -> {
                    snackbarHostState.showSnackbar(event.message)
                }
                is Event.ShowError -> {
                    snackbarHostState.showSnackbar(event.error)
                }
            }
        }
    }
    
    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) {
        // Content
    }
}
```

## SavedStateHandle

### Save and Restore State

```kotlin
import androidx.lifecycle.SavedStateHandle

@HiltViewModel
class DetailViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: Repository
) : ViewModel() {
    
    // Automatically saved across process death
    var searchQuery: String
        get() = savedStateHandle.get<String>("query") ?: ""
        set(value) {
            savedStateHandle["query"] = value
        }
    
    // StateFlow backed by SavedStateHandle
    val selectedId: StateFlow<Long?> = savedStateHandle.getStateFlow("selected_id", null)
    
    fun selectItem(id: Long) {
        savedStateHandle["selected_id"] = id
    }
}
```

### Navigation Arguments

```kotlin
@HiltViewModel
class DetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // Get navigation argument
    private val userId: Long = savedStateHandle.get<Long>("userId") ?: 0L
    
    val user = repository.getUserById(userId)
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = null
        )
}
```

## Factory Pattern

### Custom ViewModel Factory (without Hilt)

```kotlin
import androidx.lifecycle.ViewModelProvider
import androidx.lifecycle.viewmodel.CreationExtras

class UserViewModelFactory(
    private val repository: UserRepository
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return UserViewModel(repository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}

// Usage
val viewModel: UserViewModel by viewModels {
    UserViewModelFactory(repository)
}
```

## Testing ViewModels

### Unit Test

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.After
import org.junit.Before
import org.junit.Test

@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var viewModel: UserViewModel
    private lateinit var repository: FakeUserRepository
    
    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        repository = FakeUserRepository()
        viewModel = UserViewModel(repository)
    }
    
    @After
    fun teardown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `load users shows success state`() = runTest {
        // Arrange
        val users = listOf(User(1, "John", "john@test.com"))
        repository.setUsers(users)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertTrue(state is UiState.Success)
        assertEquals(users, (state as UiState.Success).users)
    }
    
    @Test
    fun `load users shows error on failure`() = runTest {
        // Arrange
        repository.setShouldFail(true)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertTrue(state is UiState.Error)
    }
}
```

### Test with Turbine

```kotlin
import app.cash.turbine.test

@Test
fun `state emissions are correct`() = runTest {
    viewModel.uiState.test {
        // Initial state
        assertTrue(awaitItem() is UiState.Loading)
        
        // Load users
        viewModel.loadUsers()
        
        // Success state
        val success = awaitItem()
        assertTrue(success is UiState.Success)
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Best Practices

1. ✅ Use StateFlow for UI state
2. ✅ Use SharedFlow for one-shot events
3. ✅ Launch coroutines in viewModelScope
4. ✅ Keep ViewModels free of Android dependencies
5. ✅ Use `collectAsStateWithLifecycle()` in Compose
6. ✅ Use `repeatOnLifecycle()` in Activities/Fragments
7. ✅ Don't pass Context to ViewModel
8. ✅ Use SavedStateHandle for persisting state
9. ✅ Handle loading, success, and error states
10. ✅ Write unit tests for ViewModels

## Common Patterns

### Pagination

```kotlin
class PaginatedViewModel : ViewModel() {
    private val _items = MutableStateFlow<List<Item>>(emptyList())
    val items: StateFlow<List<Item>> = _items.asStateFlow()
    
    private val _isLoading = MutableStateFlow(false)
    val isLoading: StateFlow<Boolean> = _isLoading.asStateFlow()
    
    private var currentPage = 0
    private var hasMore = true
    
    init {
        loadNextPage()
    }
    
    fun loadNextPage() {
        if (_isLoading.value || !hasMore) return
        
        viewModelScope.launch {
            _isLoading.value = true
            try {
                val newItems = repository.getItems(page = currentPage)
                _items.value += newItems
                currentPage++
                hasMore = newItems.isNotEmpty()
            } finally {
                _isLoading.value = false
            }
        }
    }
}
```

### Pull-to-Refresh

```kotlin
class RefreshableViewModel : ViewModel() {
    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()
    
    fun refresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            try {
                // Reload data
                repository.refresh()
            } finally {
                _isRefreshing.value = false
            }
        }
    }
}
```

## Resources

- [ViewModel Overview](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Lifecycle-Aware Components](https://developer.android.com/topic/libraries/architecture/lifecycle)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)

