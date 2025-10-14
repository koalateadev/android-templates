# State Management Patterns in Compose

## Table of Contents
- [State Hoisting](#state-hoisting)
- [Remember Variants](#remember-variants)
- [Side Effects](#side-effects)
- [MVI Architecture](#mvi-architecture)
- [Unidirectional Data Flow](#unidirectional-data-flow)
- [CompositionLocal](#compositionlocal)
- [Derived State](#derived-state)
- [State Restoration](#state-restoration)

## State Hoisting

### Basic State Hoisting

```kotlin
// Bad - State in child
@Composable
fun CounterBad() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}

// Good - State hoisted to parent
@Composable
fun Counter(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}

@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    Counter(
        count = count,
        onIncrement = { count++ }
    )
}
```

### Stateful and Stateless Composables

```kotlin
// Stateful version - for convenience
@Composable
fun SearchBar(
    onSearchQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    var query by remember { mutableStateOf("") }
    
    SearchBarStateless(
        query = query,
        onQueryChange = {
            query = it
            onSearchQueryChange(it)
        },
        modifier = modifier
    )
}

// Stateless version - for flexibility and testing
@Composable
fun SearchBarStateless(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier,
        placeholder = { Text("Search...") }
    )
}
```

## Remember Variants

### remember

```kotlin
@Composable
fun RememberExample() {
    // Survives recomposition but NOT configuration changes
    val counter = remember { mutableStateOf(0) }
    
    Button(onClick = { counter.value++ }) {
        Text("Count: ${counter.value}")
    }
}
```

### rememberSaveable

```kotlin
@Composable
fun RememberSaveableExample() {
    // Survives recomposition AND configuration changes
    var counter by rememberSaveable { mutableStateOf(0) }
    
    Button(onClick = { counter++ }) {
        Text("Count: $counter")
    }
}
```

### rememberUpdatedState

```kotlin
@Composable
fun TimerWithCallback(onTimeout: () -> Unit) {
    // Capture latest onTimeout without restarting effect
    val currentOnTimeout by rememberUpdatedState(onTimeout)
    
    LaunchedEffect(Unit) {
        delay(5000)
        currentOnTimeout()
    }
}
```

### rememberCoroutineScope

```kotlin
@Composable
fun ScrollToTopButton(listState: LazyListState) {
    val scope = rememberCoroutineScope()
    
    FloatingActionButton(
        onClick = {
            scope.launch {
                listState.animateScrollToItem(0)
            }
        }
    ) {
        Icon(Icons.Default.ArrowUpward, contentDescription = "Scroll to top")
    }
}
```

## Side Effects

### LaunchedEffect

```kotlin
@Composable
fun LoadDataOnLaunch(userId: String) {
    val viewModel: UserViewModel = hiltViewModel()
    
    // Runs when userId changes
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }
}
```

### DisposableEffect

```kotlin
@Composable
fun LifecycleAwareScreen() {
    val lifecycleOwner = LocalLifecycleOwner.current
    
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> { /* Screen visible */ }
                Lifecycle.Event.ON_STOP -> { /* Screen hidden */ }
                else -> {}
            }
        }
        
        lifecycleOwner.lifecycle.addObserver(observer)
        
        onDispose {
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
```

### SideEffect

```kotlin
@Composable
fun AnalyticsScreen(screenName: String) {
    // Runs after every successful composition
    SideEffect {
        analytics.logScreenView(screenName)
    }
}
```

### produceState

```kotlin
@Composable
fun loadNetworkData(url: String): State<Result<String>> {
    return produceState<Result<String>>(initialValue = Result.Loading, url) {
        value = try {
            Result.Success(api.fetchData(url))
        } catch (e: Exception) {
            Result.Error(e.message ?: "Unknown error")
        }
    }
}

@Composable
fun NetworkDataScreen(url: String) {
    val dataState by loadNetworkData(url)
    
    when (dataState) {
        is Result.Loading -> LoadingScreen()
        is Result.Success -> SuccessScreen((dataState as Result.Success).data)
        is Result.Error -> ErrorScreen((dataState as Result.Error).message)
    }
}
```

### snapshotFlow

```kotlin
@Composable
fun TrackScrollPosition(listState: LazyListState) {
    LaunchedEffect(listState) {
        snapshotFlow { listState.firstVisibleItemIndex }
            .distinctUntilChanged()
            .collect { index ->
                analytics.logScroll(index)
            }
    }
}
```

## MVI Architecture

### MVI Components

```kotlin
// Intent - User actions
sealed interface UserIntent {
    data object LoadUsers : UserIntent
    data object RefreshUsers : UserIntent
    data class DeleteUser(val userId: String) : UserIntent
    data class SearchUsers(val query: String) : UserIntent
}

// State - UI state
data class UserState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

// Effect - One-time events
sealed interface UserEffect {
    data class ShowSnackbar(val message: String) : UserEffect
    data class NavigateToDetail(val userId: String) : UserEffect
}

// ViewModel
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _state = MutableStateFlow(UserState())
    val state: StateFlow<UserState> = _state.asStateFlow()
    
    private val _effect = MutableSharedFlow<UserEffect>()
    val effect: SharedFlow<UserEffect> = _effect.asSharedFlow()
    
    init {
        processIntent(UserIntent.LoadUsers)
    }
    
    fun processIntent(intent: UserIntent) {
        when (intent) {
            is UserIntent.LoadUsers -> loadUsers()
            is UserIntent.RefreshUsers -> refreshUsers()
            is UserIntent.DeleteUser -> deleteUser(intent.userId)
            is UserIntent.SearchUsers -> searchUsers(intent.query)
        }
    }
    
    private fun loadUsers() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            repository.getUsers()
                .onSuccess { users ->
                    _state.update { it.copy(users = users, isLoading = false) }
                }
                .onFailure { error ->
                    _state.update { it.copy(error = error.message, isLoading = false) }
                }
        }
    }
    
    private fun deleteUser(userId: String) {
        viewModelScope.launch {
            repository.deleteUser(userId)
                .onSuccess {
                    _effect.emit(UserEffect.ShowSnackbar("User deleted"))
                    loadUsers() // Refresh list
                }
                .onFailure { error ->
                    _effect.emit(UserEffect.ShowSnackbar(error.message ?: "Delete failed"))
                }
        }
    }
    
    private fun searchUsers(query: String) {
        _state.update { it.copy(searchQuery = query) }
        // Implement search logic
    }
    
    private fun refreshUsers() {
        loadUsers()
    }
}

// Screen
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val state by viewModel.state.collectAsState()
    val snackbarHostState = remember { SnackbarHostState() }
    
    // Handle one-time effects
    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is UserEffect.ShowSnackbar -> {
                    snackbarHostState.showSnackbar(effect.message)
                }
                is UserEffect.NavigateToDetail -> {
                    // Handle navigation
                }
            }
        }
    }
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        UserContent(
            state = state,
            onIntent = viewModel::processIntent,
            modifier = Modifier.padding(paddingValues)
        )
    }
}

@Composable
fun UserContent(
    state: UserState,
    onIntent: (UserIntent) -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier.fillMaxSize()) {
        when {
            state.isLoading && state.users.isEmpty() -> {
                LoadingScreen()
            }
            state.error != null && state.users.isEmpty() -> {
                ErrorScreen(
                    message = state.error,
                    onRetry = { onIntent(UserIntent.RefreshUsers) }
                )
            }
            else -> {
                LazyColumn {
                    items(state.users) { user ->
                        UserItem(
                            user = user,
                            onDelete = { onIntent(UserIntent.DeleteUser(user.id)) }
                        )
                    }
                }
            }
        }
    }
}
```

## Unidirectional Data Flow

### Complete UDF Pattern

```kotlin
// Screen State
data class ScreenState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val selectedItem: Item? = null,
    val filterOption: FilterOption = FilterOption.ALL
)

// User Events
sealed interface ScreenEvent {
    data object LoadItems : ScreenEvent
    data class SelectItem(val item: Item) : ScreenEvent
    data class FilterItems(val filter: FilterOption) : ScreenEvent
    data object ClearSelection : ScreenEvent
}

// One-Time Effects
sealed interface ScreenEffect {
    data class ShowToast(val message: String) : ScreenEffect
    data class NavigateToDetails(val itemId: String) : ScreenEffect
}

// ViewModel with UDF
class ScreenViewModel : ViewModel() {
    private val _state = MutableStateFlow(ScreenState())
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    private val _effect = MutableSharedFlow<ScreenEffect>()
    val effect: SharedFlow<ScreenEffect> = _effect.asSharedFlow()
    
    fun onEvent(event: ScreenEvent) {
        when (event) {
            is ScreenEvent.LoadItems -> loadItems()
            is ScreenEvent.SelectItem -> selectItem(event.item)
            is ScreenEvent.FilterItems -> filterItems(event.filter)
            is ScreenEvent.ClearSelection -> clearSelection()
        }
    }
    
    private fun loadItems() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            // Load items
            _state.update { it.copy(isLoading = false, items = loadedItems) }
        }
    }
    
    private fun selectItem(item: Item) {
        _state.update { it.copy(selectedItem = item) }
        viewModelScope.launch {
            _effect.emit(ScreenEffect.NavigateToDetails(item.id))
        }
    }
    
    private fun filterItems(filter: FilterOption) {
        _state.update { it.copy(filterOption = filter) }
    }
    
    private fun clearSelection() {
        _state.update { it.copy(selectedItem = null) }
    }
}
```

## CompositionLocal

### Define CompositionLocal

```kotlin
// Define
val LocalAppSettings = compositionLocalOf<AppSettings> {
    error("No AppSettings provided")
}

data class AppSettings(
    val isDarkMode: Boolean,
    val language: String,
    val fontSize: Float
)

// Provide
@Composable
fun App() {
    val appSettings = remember {
        AppSettings(
            isDarkMode = false,
            language = "en",
            fontSize = 16f
        )
    }
    
    CompositionLocalProvider(LocalAppSettings provides appSettings) {
        MainScreen()
    }
}

// Consume
@Composable
fun SettingsDisplay() {
    val settings = LocalAppSettings.current
    
    Text("Language: ${settings.language}")
    Text("Font Size: ${settings.fontSize}")
}
```

### Custom Theme with CompositionLocal

```kotlin
data class CustomColors(
    val success: Color,
    val warning: Color,
    val info: Color
)

val LocalCustomColors = staticCompositionLocalOf {
    CustomColors(
        success = Color.Green,
        warning = Color.Yellow,
        info = Color.Blue
    )
}

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val customColors = if (darkTheme) {
        CustomColors(
            success = Color(0xFF4CAF50),
            warning = Color(0xFFFFC107),
            info = Color(0xFF2196F3)
        )
    } else {
        CustomColors(
            success = Color(0xFF81C784),
            warning = Color(0xFFFFD54F),
            info = Color(0xFF64B5F6)
        )
    }
    
    CompositionLocalProvider(LocalCustomColors provides customColors) {
        MaterialTheme {
            content()
        }
    }
}

// Extension property for easy access
val MaterialTheme.customColors: CustomColors
    @Composable
    @ReadOnlyComposable
    get() = LocalCustomColors.current

// Usage
@Composable
fun StatusCard() {
    Card(
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.customColors.success
        )
    ) {
        Text("Success!")
    }
}
```

## Derived State

### Basic derivedStateOf

```kotlin
@Composable
fun UserList(users: List<User>) {
    var searchQuery by remember { mutableStateOf("") }
    
    // Only recalculates when users or searchQuery changes
    val filteredUsers by remember {
        derivedStateOf {
            if (searchQuery.isEmpty()) {
                users
            } else {
                users.filter { it.name.contains(searchQuery, ignoreCase = true) }
            }
        }
    }
    
    Column {
        SearchBar(
            query = searchQuery,
            onQueryChange = { searchQuery = it }
        )
        
        LazyColumn {
            items(filteredUsers) { user ->
                UserItem(user)
            }
        }
    }
}
```

### Complex Derived State

```kotlin
@Composable
fun ShoppingCart(
    items: List<CartItem>,
    taxRate: Float
) {
    // Derived calculations
    val subtotal by remember {
        derivedStateOf {
            items.sumOf { it.price * it.quantity }
        }
    }
    
    val tax by remember {
        derivedStateOf {
            subtotal * taxRate
        }
    }
    
    val total by remember {
        derivedStateOf {
            subtotal + tax
        }
    }
    
    Column {
        LazyColumn {
            items(items) { item ->
                CartItemView(item)
            }
        }
        
        Divider()
        
        Text("Subtotal: $${"%.2f".format(subtotal)}")
        Text("Tax: $${"%.2f".format(tax)}")
        Text(
            "Total: $${"%.2f".format(total)}",
            style = MaterialTheme.typography.titleLarge
        )
    }
}
```

## State Restoration

### Custom Saver

```kotlin
data class CustomState(
    val id: String,
    val name: String,
    val count: Int
)

val CustomStateSaver = Saver<CustomState, List<Any>>(
    save = { state ->
        listOf(state.id, state.name, state.count)
    },
    restore = { list ->
        CustomState(
            id = list[0] as String,
            name = list[1] as String,
            count = list[2] as Int
        )
    }
)

@Composable
fun CustomStateExample() {
    var state by rememberSaveable(stateSaver = CustomStateSaver) {
        mutableStateOf(CustomState("1", "Default", 0))
    }
    
    // Use state
}
```

### List Saver

```kotlin
data class ShoppingList(
    val items: List<String>,
    val isChecked: Map<String, Boolean>
)

val ShoppingListSaver = listSaver<ShoppingList, Any>(
    save = { list ->
        buildList {
            add(list.items)
            add(list.isChecked)
        }
    },
    restore = { saved ->
        ShoppingList(
            items = saved[0] as List<String>,
            isChecked = saved[1] as Map<String, Boolean>
        )
    }
)
```

## State Best Practices

### 1. Single Source of Truth

```kotlin
// Bad - Multiple sources of truth
@Composable
fun BadExample() {
    var name by remember { mutableStateOf("") }
    var displayName by remember { mutableStateOf("") }
    
    // Two states for same data
}

// Good - Single source
@Composable
fun GoodExample() {
    var name by remember { mutableStateOf("") }
    val displayName = name.ifEmpty { "Unknown" }
    
    // Derived from single source
}
```

### 2. Avoid Redundant State

```kotlin
// Bad
@Composable
fun BadList(items: List<Item>) {
    var itemCount by remember(items) { mutableStateOf(items.size) }
    // Redundant state
}

// Good
@Composable
fun GoodList(items: List<Item>) {
    val itemCount = items.size  // Derived directly
}
```

### 3. Minimize State Scope

```kotlin
// Bad - State at top level when not needed
@Composable
fun BadScreen() {
    var expandedItemId by remember { mutableStateOf<String?>(null) }
    
    LazyColumn {
        items(items) { item ->
            ItemCard(item, expandedItemId == item.id)
        }
    }
}

// Good - State closer to usage
@Composable
fun ItemCard(item: Item) {
    var isExpanded by remember { mutableStateOf(false) }
    
    Card(
        onClick = { isExpanded = !isExpanded }
    ) {
        // Card content
    }
}
```

## ViewModel State Patterns

### Loading, Success, Error Pattern

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

class DataViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState<List<Item>>>(UiState.Loading)
    val uiState: StateFlow<UiState<List<Item>>> = _uiState.asStateFlow()
    
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            
            repository.getData()
                .onSuccess { data ->
                    _uiState.value = UiState.Success(data)
                }
                .onFailure { error ->
                    _uiState.value = UiState.Error(error.message ?: "Unknown error")
                }
        }
    }
}
```

### Separate Loading States

```kotlin
data class ScreenState(
    val items: List<Item> = emptyList(),
    val isInitialLoading: Boolean = false,
    val isRefreshing: Boolean = false,
    val isPaginating: Boolean = false,
    val error: String? = null
)

class MultiLoadingViewModel : ViewModel() {
    private val _state = MutableStateFlow(ScreenState())
    val state: StateFlow<ScreenState> = _state.asStateFlow()
    
    fun initialLoad() {
        viewModelScope.launch {
            _state.update { it.copy(isInitialLoading = true) }
            // Load data
            _state.update { it.copy(isInitialLoading = false, items = data) }
        }
    }
    
    fun refresh() {
        viewModelScope.launch {
            _state.update { it.copy(isRefreshing = true) }
            // Refresh data
            _state.update { it.copy(isRefreshing = false, items = data) }
        }
    }
    
    fun loadMore() {
        viewModelScope.launch {
            _state.update { it.copy(isPaginating = true) }
            // Load more
            _state.update { it.copy(isPaginating = false, items = it.items + newData) }
        }
    }
}
```

## Complex State Management

### Grouped State Updates

```kotlin
class FormViewModel : ViewModel() {
    private val _state = MutableStateFlow(FormState())
    val state: StateFlow<FormState> = _state.asStateFlow()
    
    fun updateField(update: FormState.() -> FormState) {
        _state.update(update)
    }
    
    // Usage
    fun onNameChange(name: String) {
        updateField { copy(name = name, nameError = validateName(name)) }
    }
    
    fun onEmailChange(email: String) {
        updateField { copy(email = email, emailError = validateEmail(email)) }
    }
}
```

### State with Undo/Redo

```kotlin
class UndoRedoViewModel<T>(initialState: T) : ViewModel() {
    private val history = mutableListOf(initialState)
    private var historyIndex = 0
    
    private val _state = MutableStateFlow(initialState)
    val state: StateFlow<T> = _state.asStateFlow()
    
    val canUndo: Boolean get() = historyIndex > 0
    val canRedo: Boolean get() = historyIndex < history.lastIndex
    
    fun updateState(newState: T) {
        // Remove any redo history
        while (history.size > historyIndex + 1) {
            history.removeLast()
        }
        
        history.add(newState)
        historyIndex = history.lastIndex
        _state.value = newState
    }
    
    fun undo() {
        if (canUndo) {
            historyIndex--
            _state.value = history[historyIndex]
        }
    }
    
    fun redo() {
        if (canRedo) {
            historyIndex++
            _state.value = history[historyIndex]
        }
    }
}
```

## Best Practices

1. ✅ **Hoist state** to the lowest common ancestor
2. ✅ **Use StateFlow** in ViewModels, not MutableState
3. ✅ **Separate state and events** clearly
4. ✅ **Use sealed interfaces** for type-safe states
5. ✅ **Minimize state** - derive when possible
6. ✅ **Use appropriate remember** variants
7. ✅ **Handle side effects** correctly
8. ✅ **Make state immutable** (use data classes with copy)
9. ✅ **Single source of truth** for each piece of state
10. ✅ **Test state changes** independently from UI

## Common Mistakes

### ❌ Creating State in Unstable Composables

```kotlin
// Bad
@Composable
fun BadComposable() {
    val state = mutableStateOf(0)  // Creates new state every recomposition
}

// Good
@Composable
fun GoodComposable() {
    val state = remember { mutableStateOf(0) }  // Survives recomposition
}
```

### ❌ Not Using Keys in Lists

```kotlin
// Bad - Items may get confused on recomposition
LazyColumn {
    items(users) { user ->
        UserItem(user)
    }
}

// Good - Stable keys prevent issues
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }
    ) { user ->
        UserItem(user)
    }
}
```

### ❌ Reading State During Composition

```kotlin
// Bad - Can cause infinite recomposition
@Composable
fun BadStateRead(viewModel: MyViewModel) {
    viewModel.updateState()  // Side effect during composition
}

// Good - Use LaunchedEffect
@Composable
fun GoodStateRead(viewModel: MyViewModel) {
    LaunchedEffect(Unit) {
        viewModel.updateState()
    }
}
```

## Resources

- [State and Jetpack Compose](https://developer.android.com/jetpack/compose/state)
- [Side Effects in Compose](https://developer.android.com/jetpack/compose/side-effects)
- [Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model)
- [State Hoisting](https://developer.android.com/jetpack/compose/state-hoisting)

