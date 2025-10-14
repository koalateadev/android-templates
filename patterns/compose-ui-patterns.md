# Common Compose UI Patterns

## Table of Contents
- [Loading States](#loading-states)
- [Error Handling](#error-handling)
- [Empty States](#empty-states)
- [Pull to Refresh](#pull-to-refresh)
- [Infinite Scroll](#infinite-scroll)
- [Search](#search)
- [Dialogs](#dialogs)
- [Bottom Sheets](#bottom-sheets)
- [Swipe to Delete](#swipe-to-delete)
- [Form Validation](#form-validation)
- [Image Pickers](#image-pickers)
- [Tabs](#tabs)
- [Toasts](#toasts)
- [Snackbars](#snackbars)
- [Advanced Forms](#advanced-forms)
- [More Dialogs](#more-dialogs)
- [Filters and Sort](#filters-and-sort)

## Loading States

### Basic Loading

```kotlin
@Composable
fun LoadingScreen() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        CircularProgressIndicator()
    }
}
```

### Loading with Message

```kotlin
@Composable
fun LoadingWithMessage(message: String = "Loading...") {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.spacedBy(16.dp)
        ) {
            CircularProgressIndicator()
            Text(
                text = message,
                style = MaterialTheme.typography.bodyMedium
            )
        }
    }
}
```

### Content with Loading Overlay

```kotlin
@Composable
fun ContentWithLoading(
    isLoading: Boolean,
    content: @Composable () -> Unit
) {
    Box(modifier = Modifier.fillMaxSize()) {
        content()
        
        if (isLoading) {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(Color.Black.copy(alpha = 0.5f))
                    .clickable(enabled = false) { },
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator(color = Color.White)
            }
        }
    }
}
```

## Error Handling

### Error Screen

```kotlin
@Composable
fun ErrorScreen(
    message: String,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.Error,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.error
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Text(
            text = message,
            style = MaterialTheme.typography.bodyLarge,
            textAlign = TextAlign.Center
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Button(onClick = onRetry) {
            Text("Retry")
        }
    }
}
```

### Inline Error

```kotlin
@Composable
fun InlineError(
    message: String,
    onDismiss: () -> Unit
) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        colors = CardDefaults.cardColors(
            containerColor = MaterialTheme.colorScheme.errorContainer
        )
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Icon(
                imageVector = Icons.Default.Warning,
                contentDescription = null,
                tint = MaterialTheme.colorScheme.error
            )
            
            Spacer(modifier = Modifier.width(8.dp))
            
            Text(
                text = message,
                modifier = Modifier.weight(1f),
                style = MaterialTheme.typography.bodyMedium
            )
            
            IconButton(onClick = onDismiss) {
                Icon(Icons.Default.Close, contentDescription = "Dismiss")
            }
        }
    }
}
```

## Empty States

### Empty List

```kotlin
@Composable
fun EmptyState(
    message: String,
    actionText: String? = null,
    onAction: (() -> Unit)? = null
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.Inbox,
            contentDescription = null,
            modifier = Modifier.size(120.dp),
            tint = MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.5f)
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Text(
            text = message,
            style = MaterialTheme.typography.titleMedium,
            textAlign = TextAlign.Center,
            color = MaterialTheme.colorScheme.onSurfaceVariant
        )
        
        if (actionText != null && onAction != null) {
            Spacer(modifier = Modifier.height(16.dp))
            
            TextButton(onClick = onAction) {
                Text(actionText)
            }
        }
    }
}
```

## Pull to Refresh

```kotlin
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.pulltorefresh.PullToRefreshBox

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun PullToRefreshList(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    content: @Composable () -> Unit
) {
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh
    ) {
        content()
    }
}

// Usage
@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    val isRefreshing = uiState is UiState.Loading
    
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { viewModel.refresh() }
    ) {
        LazyColumn {
            items(users) { user ->
                UserItem(user)
            }
        }
    }
}
```

## Infinite Scroll

```kotlin
@Composable
fun LazyListPagination(
    items: List<Item>,
    isLoading: Boolean,
    canLoadMore: Boolean,
    onLoadMore: () -> Unit
) {
    val listState = rememberLazyListState()
    
    LazyColumn(state = listState) {
        items(items) { item ->
            ItemView(item)
        }
        
        if (isLoading) {
            item {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
        }
    }
    
    // Detect when scrolled to bottom
    LaunchedEffect(listState) {
        snapshotFlow { listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index }
            .collect { lastVisibleIndex ->
                if (lastVisibleIndex != null &&
                    lastVisibleIndex >= items.size - 3 &&
                    canLoadMore &&
                    !isLoading
                ) {
                    onLoadMore()
                }
            }
    }
}
```

## Search

### Search Bar with Debounce

```kotlin
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        placeholder = { Text("Search...") },
        leadingIcon = {
            Icon(Icons.Default.Search, contentDescription = null)
        },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { onQueryChange("") }) {
                    Icon(Icons.Default.Close, contentDescription = "Clear")
                }
            }
        },
        singleLine = true
    )
}

// ViewModel with debounce
class SearchViewModel : ViewModel() {
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
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

## Dialogs

### Confirmation Dialog

```kotlin
@Composable
fun ConfirmationDialog(
    title: String,
    message: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit,
    confirmText: String = "Confirm",
    dismissText: String = "Cancel"
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = { Text(message) },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text(confirmText)
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text(dismissText)
            }
        }
    )
}

// Usage
var showDialog by remember { mutableStateOf(false) }

Button(onClick = { showDialog = true }) {
    Text("Delete")
}

if (showDialog) {
    ConfirmationDialog(
        title = "Delete Item",
        message = "Are you sure you want to delete this item?",
        onConfirm = {
            viewModel.deleteItem()
            showDialog = false
        },
        onDismiss = { showDialog = false }
    )
}
```

### Input Dialog

```kotlin
@Composable
fun InputDialog(
    title: String,
    label: String,
    initialValue: String = "",
    onConfirm: (String) -> Unit,
    onDismiss: () -> Unit
) {
    var text by remember { mutableStateOf(initialValue) }
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = {
            OutlinedTextField(
                value = text,
                onValueChange = { text = it },
                label = { Text(label) },
                singleLine = true
            )
        },
        confirmButton = {
            TextButton(
                onClick = { onConfirm(text) },
                enabled = text.isNotBlank()
            ) {
                Text("Save")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        }
    )
}
```

## Bottom Sheets

### Modal Bottom Sheet

```kotlin
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.ModalBottomSheet
import androidx.compose.material3.rememberModalBottomSheetState

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OptionsBottomSheet(
    onDismiss: () -> Unit,
    onEdit: () -> Unit,
    onDelete: () -> Unit,
    onShare: () -> Unit
) {
    val sheetState = rememberModalBottomSheetState()
    
    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(bottom = 32.dp)
        ) {
            ListItem(
                headlineContent = { Text("Edit") },
                leadingContent = { Icon(Icons.Default.Edit, null) },
                modifier = Modifier.clickable {
                    onEdit()
                    onDismiss()
                }
            )
            
            ListItem(
                headlineContent = { Text("Share") },
                leadingContent = { Icon(Icons.Default.Share, null) },
                modifier = Modifier.clickable {
                    onShare()
                    onDismiss()
                }
            )
            
            ListItem(
                headlineContent = { Text("Delete") },
                leadingContent = {
                    Icon(
                        Icons.Default.Delete,
                        null,
                        tint = MaterialTheme.colorScheme.error
                    )
                },
                modifier = Modifier.clickable {
                    onDelete()
                    onDismiss()
                }
            )
        }
    }
}
```

## Swipe to Delete

```kotlin
import androidx.compose.foundation.background
import androidx.compose.material3.SwipeToDismissBox
import androidx.compose.material3.SwipeToDismissBoxValue
import androidx.compose.material3.rememberSwipeToDismissBoxState

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeToDeleteItem(
    item: Item,
    onDelete: () -> Unit,
    content: @Composable () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { value ->
            if (value == SwipeToDismissBoxValue.EndToStart) {
                onDelete()
                true
            } else {
                false
            }
        }
    )
    
    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(MaterialTheme.colorScheme.error)
                    .padding(16.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = "Delete",
                    tint = Color.White
                )
            }
        },
        enableDismissFromStartToEnd = false
    ) {
        content()
    }
}
```

## Form Validation

```kotlin
data class FormState(
    val name: String = "",
    val email: String = "",
    val password: String = "",
    val nameError: String? = null,
    val emailError: String? = null,
    val passwordError: String? = null
) {
    val isValid: Boolean
        get() = name.isNotBlank() &&
                email.isNotBlank() &&
                password.isNotBlank() &&
                nameError == null &&
                emailError == null &&
                passwordError == null
}

class FormViewModel : ViewModel() {
    var formState by mutableStateOf(FormState())
        private set
    
    fun updateName(name: String) {
        formState = formState.copy(
            name = name,
            nameError = validateName(name)
        )
    }
    
    fun updateEmail(email: String) {
        formState = formState.copy(
            email = email,
            emailError = validateEmail(email)
        )
    }
    
    fun updatePassword(password: String) {
        formState = formState.copy(
            password = password,
            passwordError = validatePassword(password)
        )
    }
    
    private fun validateName(name: String): String? {
        return when {
            name.isBlank() -> "Name is required"
            name.length < 2 -> "Name must be at least 2 characters"
            else -> null
        }
    }
    
    private fun validateEmail(email: String): String? {
        return when {
            email.isBlank() -> "Email is required"
            !android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches() ->
                "Invalid email format"
            else -> null
        }
    }
    
    private fun validatePassword(password: String): String? {
        return when {
            password.isBlank() -> "Password is required"
            password.length < 8 -> "Password must be at least 8 characters"
            else -> null
        }
    }
}

@Composable
fun RegistrationForm(viewModel: FormViewModel = viewModel()) {
    val formState = viewModel.formState
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        OutlinedTextField(
            value = formState.name,
            onValueChange = viewModel::updateName,
            label = { Text("Name") },
            isError = formState.nameError != null,
            supportingText = formState.nameError?.let { { Text(it) } },
            modifier = Modifier.fillMaxWidth()
        )
        
        OutlinedTextField(
            value = formState.email,
            onValueChange = viewModel::updateEmail,
            label = { Text("Email") },
            isError = formState.emailError != null,
            supportingText = formState.emailError?.let { { Text(it) } },
            modifier = Modifier.fillMaxWidth(),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email)
        )
        
        OutlinedTextField(
            value = formState.password,
            onValueChange = viewModel::updatePassword,
            label = { Text("Password") },
            isError = formState.passwordError != null,
            supportingText = formState.passwordError?.let { { Text(it) } },
            modifier = Modifier.fillMaxWidth(),
            visualTransformation = PasswordVisualTransformation(),
            keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Password)
        )
        
        Button(
            onClick = { /* Submit */ },
            modifier = Modifier.fillMaxWidth(),
            enabled = formState.isValid
        ) {
            Text("Register")
        }
    }
}
```

## Image Pickers

See the separate permissions and intents pattern files for complete image picker implementations.

## Tabs

### Tab Layout

```kotlin
@Composable
fun TabScreen() {
    var selectedTabIndex by remember { mutableStateOf(0) }
    val tabs = listOf("Tab 1", "Tab 2", "Tab 3")
    
    Column(modifier = Modifier.fillMaxSize()) {
        TabRow(selectedTabIndex = selectedTabIndex) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = selectedTabIndex == index,
                    onClick = { selectedTabIndex = index },
                    text = { Text(title) }
                )
            }
        }
        
        when (selectedTabIndex) {
            0 -> Tab1Content()
            1 -> Tab2Content()
            2 -> Tab3Content()
        }
    }
}
```

### Scrollable Tabs

```kotlin
@Composable
fun ScrollableTabsScreen() {
    var selectedTabIndex by remember { mutableStateOf(0) }
    val tabs = List(10) { "Tab ${it + 1}" }
    
    Column(modifier = Modifier.fillMaxSize()) {
        ScrollableTabRow(selectedTabIndex = selectedTabIndex) {
            tabs.forEachIndexed { index, title ->
                Tab(
                    selected = selectedTabIndex == index,
                    onClick = { selectedTabIndex = index },
                    text = { Text(title) }
                )
            }
        }
        
        HorizontalPager(
            count = tabs.size,
            state = rememberPagerState(),
            modifier = Modifier.fillMaxSize()
        ) { page ->
            TabContent(page)
        }
    }
}
```

## Toasts

### Simple Toast

```kotlin
@Composable
fun ShowToastButton(message: String) {
    val context = LocalContext.current
    
    Button(onClick = {
        Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
    }) {
        Text("Show Toast")
    }
}
```

### Toast Extension

```kotlin
fun Context.showToast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}

// Usage
val context = LocalContext.current
Button(onClick = { context.showToast("Hello!") }) {
    Text("Show Toast")
}
```

### Custom Toast

```kotlin
@Composable
fun CustomToast(
    message: String,
    icon: ImageVector? = null,
    backgroundColor: Color = MaterialTheme.colorScheme.surface,
    textColor: Color = MaterialTheme.colorScheme.onSurface
) {
    val context = LocalContext.current
    
    Button(onClick = {
        val toast = Toast(context)
        toast.duration = Toast.LENGTH_LONG
        toast.view = ComposeView(context).apply {
            setContent {
                MaterialTheme {
                    Surface(
                        shape = RoundedCornerShape(8.dp),
                        color = backgroundColor,
                        shadowElevation = 4.dp,
                        modifier = Modifier.padding(16.dp)
                    ) {
                        Row(
                            modifier = Modifier.padding(12.dp),
                            verticalAlignment = Alignment.CenterVertically
                        ) {
                            icon?.let {
                                Icon(
                                    imageVector = it,
                                    contentDescription = null,
                                    tint = textColor
                                )
                                Spacer(modifier = Modifier.width(8.dp))
                            }
                            Text(text = message, color = textColor)
                        }
                    }
                }
            }
        }
        toast.show()
    }) {
        Text("Show Custom Toast")
    }
}
```

## Snackbars

### Basic Snackbar

```kotlin
@Composable
fun SnackbarExample() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        Column(modifier = Modifier.padding(paddingValues)) {
            Button(onClick = {
                scope.launch {
                    snackbarHostState.showSnackbar("This is a snackbar")
                }
            }) {
                Text("Show Snackbar")
            }
        }
    }
}
```

### Snackbar with Action

```kotlin
@Composable
fun SnackbarWithAction() {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        Column(modifier = Modifier.padding(paddingValues)) {
            Button(onClick = {
                scope.launch {
                    val result = snackbarHostState.showSnackbar(
                        message = "Item deleted",
                        actionLabel = "Undo",
                        duration = SnackbarDuration.Long
                    )
                    when (result) {
                        SnackbarResult.ActionPerformed -> {
                            // Handle undo
                        }
                        SnackbarResult.Dismissed -> {
                            // Handle dismiss
                        }
                    }
                }
            }) {
                Text("Delete Item")
            }
        }
    }
}
```

### Custom Snackbar

```kotlin
@Composable
fun CustomSnackbar(
    snackbarData: SnackbarData,
    backgroundColor: Color = MaterialTheme.colorScheme.inverseSurface,
    contentColor: Color = MaterialTheme.colorScheme.inverseOnSurface,
    actionColor: Color = MaterialTheme.colorScheme.inversePrimary
) {
    Snackbar(
        modifier = Modifier.padding(16.dp),
        action = {
            snackbarData.visuals.actionLabel?.let { actionLabel ->
                TextButton(
                    onClick = { snackbarData.performAction() },
                    colors = ButtonDefaults.textButtonColors(
                        contentColor = actionColor
                    )
                ) {
                    Text(actionLabel)
                }
            }
        },
        dismissAction = {
            IconButton(onClick = { snackbarData.dismiss() }) {
                Icon(Icons.Default.Close, contentDescription = "Dismiss")
            }
        },
        containerColor = backgroundColor,
        contentColor = contentColor,
        shape = RoundedCornerShape(8.dp)
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            Icon(
                imageVector = Icons.Default.Info,
                contentDescription = null,
                modifier = Modifier.size(20.dp)
            )
            Spacer(modifier = Modifier.width(8.dp))
            Text(snackbarData.visuals.message)
        }
    }
}

// Usage
Scaffold(
    snackbarHost = {
        SnackbarHost(snackbarHostState) { data ->
            CustomSnackbar(data)
        }
    }
) { /* Content */ }
```

### Snackbar Manager

```kotlin
class SnackbarManager {
    private val _messages = MutableSharedFlow<String>()
    val messages: SharedFlow<String> = _messages.asSharedFlow()
    
    suspend fun showMessage(message: String) {
        _messages.emit(message)
    }
}

@Composable
fun SnackbarManagerExample(snackbarManager: SnackbarManager) {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()
    
    LaunchedEffect(Unit) {
        snackbarManager.messages.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }
    
    Scaffold(
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        // Content
    }
}
```

## Advanced Forms

### Multi-Step Form

```kotlin
data class FormStep(
    val title: String,
    val isComplete: Boolean = false
)

@Composable
fun MultiStepForm() {
    var currentStep by remember { mutableStateOf(0) }
    val steps = listOf(
        FormStep("Personal Info"),
        FormStep("Contact Details"),
        FormStep("Preferences")
    )
    
    Column(modifier = Modifier.fillMaxSize()) {
        // Progress Indicator
        LinearProgressIndicator(
            progress = { (currentStep + 1) / steps.size.toFloat() },
            modifier = Modifier.fillMaxWidth()
        )
        
        // Step Indicator
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            steps.forEachIndexed { index, step ->
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Box(
                        modifier = Modifier
                            .size(32.dp)
                            .background(
                                if (index <= currentStep)
                                    MaterialTheme.colorScheme.primary
                                else
                                    MaterialTheme.colorScheme.surfaceVariant,
                                CircleShape
                            ),
                        contentAlignment = Alignment.Center
                    ) {
                        Text(
                            text = "${index + 1}",
                            color = if (index <= currentStep)
                                MaterialTheme.colorScheme.onPrimary
                            else
                                MaterialTheme.colorScheme.onSurfaceVariant
                        )
                    }
                    Spacer(modifier = Modifier.height(4.dp))
                    Text(
                        text = step.title,
                        style = MaterialTheme.typography.labelSmall,
                        maxLines = 1
                    )
                }
            }
        }
        
        // Form Content
        Box(
            modifier = Modifier
                .weight(1f)
                .padding(16.dp)
        ) {
            when (currentStep) {
                0 -> PersonalInfoForm()
                1 -> ContactDetailsForm()
                2 -> PreferencesForm()
            }
        }
        
        // Navigation Buttons
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            horizontalArrangement = Arrangement.SpaceBetween
        ) {
            if (currentStep > 0) {
                OutlinedButton(onClick = { currentStep-- }) {
                    Icon(Icons.Default.ArrowBack, contentDescription = null)
                    Spacer(modifier = Modifier.width(4.dp))
                    Text("Back")
                }
            } else {
                Spacer(modifier = Modifier.width(1.dp))
            }
            
            Button(onClick = {
                if (currentStep < steps.size - 1) {
                    currentStep++
                } else {
                    // Submit form
                }
            }) {
                Text(if (currentStep < steps.size - 1) "Next" else "Submit")
                Spacer(modifier = Modifier.width(4.dp))
                Icon(Icons.Default.ArrowForward, contentDescription = null)
            }
        }
    }
}
```

### Dynamic Form Fields

```kotlin
data class FormField(
    val id: String,
    val type: FieldType,
    val label: String,
    val value: String = "",
    val required: Boolean = false,
    val options: List<String> = emptyList()
)

enum class FieldType {
    TEXT, EMAIL, NUMBER, DROPDOWN, CHECKBOX, DATE
}

@Composable
fun DynamicForm(fields: List<FormField>) {
    var formData by remember {
        mutableStateOf(fields.associate { it.id to it.value })
    }
    
    LazyColumn(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        items(fields) { field ->
            when (field.type) {
                FieldType.TEXT -> {
                    OutlinedTextField(
                        value = formData[field.id] ?: "",
                        onValueChange = { formData = formData + (field.id to it) },
                        label = { Text(field.label + if (field.required) " *" else "") },
                        modifier = Modifier.fillMaxWidth()
                    )
                }
                
                FieldType.EMAIL -> {
                    OutlinedTextField(
                        value = formData[field.id] ?: "",
                        onValueChange = { formData = formData + (field.id to it) },
                        label = { Text(field.label + if (field.required) " *" else "") },
                        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
                        modifier = Modifier.fillMaxWidth()
                    )
                }
                
                FieldType.NUMBER -> {
                    OutlinedTextField(
                        value = formData[field.id] ?: "",
                        onValueChange = { formData = formData + (field.id to it) },
                        label = { Text(field.label + if (field.required) " *" else "") },
                        keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Number),
                        modifier = Modifier.fillMaxWidth()
                    )
                }
                
                FieldType.DROPDOWN -> {
                    var expanded by remember { mutableStateOf(false) }
                    ExposedDropdownMenuBox(
                        expanded = expanded,
                        onExpandedChange = { expanded = it }
                    ) {
                        OutlinedTextField(
                            value = formData[field.id] ?: "",
                            onValueChange = {},
                            readOnly = true,
                            label = { Text(field.label) },
                            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded) },
                            modifier = Modifier
                                .fillMaxWidth()
                                .menuAnchor()
                        )
                        
                        ExposedDropdownMenu(
                            expanded = expanded,
                            onDismissRequest = { expanded = false }
                        ) {
                            field.options.forEach { option ->
                                DropdownMenuItem(
                                    text = { Text(option) },
                                    onClick = {
                                        formData = formData + (field.id to option)
                                        expanded = false
                                    }
                                )
                            }
                        }
                    }
                }
                
                FieldType.CHECKBOX -> {
                    Row(
                        verticalAlignment = Alignment.CenterVertically,
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Checkbox(
                            checked = formData[field.id] == "true",
                            onCheckedChange = {
                                formData = formData + (field.id to it.toString())
                            }
                        )
                        Spacer(modifier = Modifier.width(8.dp))
                        Text(field.label)
                    }
                }
                
                FieldType.DATE -> {
                    // Implement date picker
                    OutlinedTextField(
                        value = formData[field.id] ?: "",
                        onValueChange = {},
                        label = { Text(field.label) },
                        readOnly = true,
                        modifier = Modifier.fillMaxWidth()
                    )
                }
            }
        }
        
        item {
            Button(
                onClick = { /* Submit form */ },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Submit")
            }
        }
    }
}
```

## More Dialogs

### List Selection Dialog

```kotlin
@Composable
fun ListSelectionDialog(
    title: String,
    items: List<String>,
    onItemSelected: (String) -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = {
            LazyColumn {
                items(items) { item ->
                    TextButton(
                        onClick = {
                            onItemSelected(item)
                            onDismiss()
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Text(
                            text = item,
                            modifier = Modifier.fillMaxWidth(),
                            textAlign = TextAlign.Start
                        )
                    }
                }
            }
        },
        confirmButton = {}
    )
}
```

### Multi-Select Dialog

```kotlin
@Composable
fun MultiSelectDialog(
    title: String,
    items: List<String>,
    selectedItems: Set<String>,
    onConfirm: (Set<String>) -> Unit,
    onDismiss: () -> Unit
) {
    var selected by remember { mutableStateOf(selectedItems) }
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text(title) },
        text = {
            LazyColumn {
                items(items) { item ->
                    Row(
                        modifier = Modifier
                            .fillMaxWidth()
                            .clickable {
                                selected = if (item in selected) {
                                    selected - item
                                } else {
                                    selected + item
                                }
                            }
                            .padding(vertical = 8.dp),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Checkbox(
                            checked = item in selected,
                            onCheckedChange = null
                        )
                        Spacer(modifier = Modifier.width(8.dp))
                        Text(item)
                    }
                }
            }
        },
        confirmButton = {
            TextButton(onClick = {
                onConfirm(selected)
                onDismiss()
            }) {
                Text("Confirm")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        }
    )
}
```

### Date Picker Dialog

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DatePickerDialog(
    onDateSelected: (Long?) -> Unit,
    onDismiss: () -> Unit
) {
    val datePickerState = rememberDatePickerState()
    
    DatePickerDialog(
        onDismissRequest = onDismiss,
        confirmButton = {
            TextButton(onClick = {
                onDateSelected(datePickerState.selectedDateMillis)
                onDismiss()
            }) {
                Text("OK")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        }
    ) {
        DatePicker(state = datePickerState)
    }
}
```

### Time Picker Dialog

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TimePickerDialog(
    onTimeSelected: (Int, Int) -> Unit,
    onDismiss: () -> Unit
) {
    val timePickerState = rememberTimePickerState()
    
    AlertDialog(
        onDismissRequest = onDismiss,
        title = { Text("Select Time") },
        text = {
            TimePicker(state = timePickerState)
        },
        confirmButton = {
            TextButton(onClick = {
                onTimeSelected(timePickerState.hour, timePickerState.minute)
                onDismiss()
            }) {
                Text("OK")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        }
    )
}
```

### Loading Dialog

```kotlin
@Composable
fun LoadingDialog(
    message: String = "Loading...",
    onDismiss: () -> Unit = {}
) {
    Dialog(
        onDismissRequest = onDismiss,
        properties = DialogProperties(dismissOnClickOutside = false)
    ) {
        Card(
            shape = RoundedCornerShape(8.dp),
            modifier = Modifier.padding(16.dp)
        ) {
            Row(
                modifier = Modifier.padding(24.dp),
                verticalAlignment = Alignment.CenterVertically
            ) {
                CircularProgressIndicator()
                Spacer(modifier = Modifier.width(16.dp))
                Text(message)
            }
        }
    }
}
```

## Filters and Sort

### Filter Chips

```kotlin
@Composable
fun FilterChips(
    filters: List<String>,
    selectedFilters: Set<String>,
    onFilterToggle: (String) -> Unit
) {
    LazyRow(
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(horizontal = 16.dp)
    ) {
        items(filters) { filter ->
            FilterChip(
                selected = filter in selectedFilters,
                onClick = { onFilterToggle(filter) },
                label = { Text(filter) },
                leadingIcon = if (filter in selectedFilters) {
                    {
                        Icon(
                            imageVector = Icons.Default.Check,
                            contentDescription = null,
                            modifier = Modifier.size(FilterChipDefaults.IconSize)
                        )
                    }
                } else {
                    null
                }
            )
        }
    }
}
```

### Filter Bottom Sheet

```kotlin
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FilterBottomSheet(
    filters: Map<String, List<String>>,
    selectedFilters: Map<String, Set<String>>,
    onFiltersApply: (Map<String, Set<String>>) -> Unit,
    onDismiss: () -> Unit
) {
    var currentFilters by remember { mutableStateOf(selectedFilters) }
    val sheetState = rememberModalBottomSheetState()
    
    ModalBottomSheet(
        onDismissRequest = onDismiss,
        sheetState = sheetState
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp)
        ) {
            Text(
                text = "Filters",
                style = MaterialTheme.typography.titleLarge,
                modifier = Modifier.padding(bottom = 16.dp)
            )
            
            filters.forEach { (category, options) ->
                Text(
                    text = category,
                    style = MaterialTheme.typography.titleMedium,
                    modifier = Modifier.padding(vertical = 8.dp)
                )
                
                options.forEach { option ->
                    Row(
                        modifier = Modifier
                            .fillMaxWidth()
                            .clickable {
                                val current = currentFilters[category] ?: emptySet()
                                currentFilters = currentFilters + (category to
                                        if (option in current) current - option
                                        else current + option
                                        )
                            }
                            .padding(vertical = 8.dp),
                        verticalAlignment = Alignment.CenterVertically
                    ) {
                        Checkbox(
                            checked = option in (currentFilters[category] ?: emptySet()),
                            onCheckedChange = null
                        )
                        Spacer(modifier = Modifier.width(8.dp))
                        Text(option)
                    }
                }
                
                Divider(modifier = Modifier.padding(vertical = 8.dp))
            }
            
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                OutlinedButton(
                    onClick = {
                        currentFilters = emptyMap()
                    },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Clear All")
                }
                
                Button(
                    onClick = {
                        onFiltersApply(currentFilters)
                        onDismiss()
                    },
                    modifier = Modifier.weight(1f)
                ) {
                    Text("Apply")
                }
            }
            
            Spacer(modifier = Modifier.height(16.dp))
        }
    }
}
```

### Sort Menu

```kotlin
enum class SortOption(val label: String) {
    NAME_ASC("Name (A-Z)"),
    NAME_DESC("Name (Z-A)"),
    DATE_NEW("Newest First"),
    DATE_OLD("Oldest First"),
    PRICE_LOW("Price: Low to High"),
    PRICE_HIGH("Price: High to Low")
}

@Composable
fun SortMenu(
    currentSort: SortOption,
    onSortSelected: (SortOption) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }
    
    Box {
        OutlinedButton(onClick = { expanded = true }) {
            Icon(Icons.Default.Sort, contentDescription = null)
            Spacer(modifier = Modifier.width(4.dp))
            Text("Sort")
        }
        
        DropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            SortOption.entries.forEach { option ->
                DropdownMenuItem(
                    text = {
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            if (option == currentSort) {
                                Icon(
                                    Icons.Default.Check,
                                    contentDescription = null,
                                    modifier = Modifier.size(20.dp)
                                )
                                Spacer(modifier = Modifier.width(8.dp))
                            } else {
                                Spacer(modifier = Modifier.width(28.dp))
                            }
                            Text(option.label)
                        }
                    },
                    onClick = {
                        onSortSelected(option)
                        expanded = false
                    }
                )
            }
        }
    }
}
```

### Combined Filter and Sort Bar

```kotlin
@Composable
fun FilterSortBar(
    filterCount: Int,
    currentSort: SortOption,
    onFilterClick: () -> Unit,
    onSortClick: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        horizontalArrangement = Arrangement.SpaceBetween
    ) {
        // Filter Button
        OutlinedButton(onClick = onFilterClick) {
            Icon(Icons.Default.FilterList, contentDescription = null)
            Spacer(modifier = Modifier.width(4.dp))
            Text("Filters")
            if (filterCount > 0) {
                Spacer(modifier = Modifier.width(4.dp))
                Badge {
                    Text(filterCount.toString())
                }
            }
        }
        
        // Sort Button
        OutlinedButton(onClick = onSortClick) {
            Icon(Icons.Default.Sort, contentDescription = null)
            Spacer(modifier = Modifier.width(4.dp))
            Text("Sort")
        }
    }
}
```

### Search with Filters

```kotlin
@Composable
fun SearchWithFilters() {
    var searchQuery by remember { mutableStateOf("") }
    var selectedFilters by remember { mutableStateOf(setOf<String>()) }
    var sortOption by remember { mutableStateOf(SortOption.NAME_ASC) }
    var showFilters by remember { mutableStateOf(false) }
    
    Column {
        // Search Bar
        OutlinedTextField(
            value = searchQuery,
            onValueChange = { searchQuery = it },
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            placeholder = { Text("Search...") },
            leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) },
            trailingIcon = {
                if (searchQuery.isNotEmpty()) {
                    IconButton(onClick = { searchQuery = "" }) {
                        Icon(Icons.Default.Close, contentDescription = "Clear")
                    }
                }
            },
            singleLine = true
        )
        
        // Filter Chips
        FilterChips(
            filters = listOf("Category 1", "Category 2", "Category 3"),
            selectedFilters = selectedFilters,
            onFilterToggle = { filter ->
                selectedFilters = if (filter in selectedFilters) {
                    selectedFilters - filter
                } else {
                    selectedFilters + filter
                }
            }
        )
        
        // Filter and Sort Bar
        FilterSortBar(
            filterCount = selectedFilters.size,
            currentSort = sortOption,
            onFilterClick = { showFilters = true },
            onSortClick = { /* Show sort menu */ }
        )
        
        // Content
        LazyColumn {
            items(100) { index ->
                Text(
                    text = "Item $index",
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp)
                )
            }
        }
    }
}
```

## Resources

- [Material3 Components](https://m3.material.io/components)
- [Compose Samples](https://github.com/android/compose-samples)
- [Compose Cookbook](https://github.com/Gurupreet/ComposeCookBook)

