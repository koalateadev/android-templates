# Performance Optimization Patterns

## Table of Contents
- [Recomposition Optimization](#recomposition-optimization)
- [LazyColumn Optimization](#lazycolumn-optimization)
- [Image Loading](#image-loading)
- [Memory Management](#memory-management)
- [State Optimization](#state-optimization)
- [Stability](#stability)
- [Profiling](#profiling)

## Recomposition Optimization

### Use derivedStateOf

```kotlin
// Bad - Recalculates on every recomposition
@Composable
fun BadList(items: List<Item>) {
    val filteredItems = items.filter { it.isActive }  // Runs every recomposition
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemView(item)
        }
    }
}

// Good - Only recalculates when items change
@Composable
fun GoodList(items: List<Item>) {
    val filteredItems by remember {
        derivedStateOf {
            items.filter { it.isActive }
        }
    }
    
    LazyColumn {
        items(filteredItems) { item ->
            ItemView(item)
        }
    }
}
```

### Skip Unnecessary Recompositions

```kotlin
// Bad - All parameters trigger recomposition
@Composable
fun ExpensiveItem(item: Item, onClick: () -> Unit) {
    // Heavy computation
    Text(item.name)
}

// Good - Stable parameters
@Composable
fun OptimizedItem(item: Item, onClick: () -> Unit) {
    // Wrap unstable lambda
    val stableClick = rememberUpdatedState(onClick)
    
    Text(
        item.name,
        modifier = Modifier.clickable { stableClick.value() }
    )
}
```

### Use remember for Expensive Operations

```kotlin
@Composable
fun ExpensiveCalculation(data: Data) {
    // Bad - Recalculates every recomposition
    val result = performHeavyCalculation(data)
    
    // Good - Only recalculates when data changes
    val result = remember(data) {
        performHeavyCalculation(data)
    }
    
    Text(result)
}
```

### Avoid Reading State During Composition

```kotlin
// Bad - Can cause infinite recomposition
@Composable
fun BadComposable(viewModel: MyViewModel) {
    if (viewModel.shouldLoad) {
        viewModel.loadData()  // Side effect during composition
    }
}

// Good - Use side effect
@Composable
fun GoodComposable(viewModel: MyViewModel) {
    LaunchedEffect(viewModel.shouldLoad) {
        if (viewModel.shouldLoad) {
            viewModel.loadData()
        }
    }
}
```

## LazyColumn Optimization

### Use Keys

```kotlin
// Bad - No keys
LazyColumn {
    items(users) { user ->
        UserItem(user)
    }
}

// Good - Stable keys
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }
    ) { user ->
        UserItem(user)
    }
}
```

### Use contentType

```kotlin
sealed class ListItem {
    data class Header(val title: String) : ListItem()
    data class User(val user: User) : ListItem()
    data class Ad(val ad: Ad) : ListItem()
}

@Composable
fun OptimizedList(items: List<ListItem>) {
    LazyColumn {
        items(
            items = items,
            key = { item ->
                when (item) {
                    is ListItem.Header -> "header-${item.title}"
                    is ListItem.User -> "user-${item.user.id}"
                    is ListItem.Ad -> "ad-${item.ad.id}"
                }
            },
            contentType = { item ->
                when (item) {
                    is ListItem.Header -> "header"
                    is ListItem.User -> "user"
                    is ListItem.Ad -> "ad"
                }
            }
        ) { item ->
            when (item) {
                is ListItem.Header -> HeaderView(item.title)
                is ListItem.User -> UserItem(item.user)
                is ListItem.Ad -> AdView(item.ad)
            }
        }
    }
}
```

### Prefetch Items

```kotlin
@Composable
fun PrefetchedList(items: List<Item>) {
    val listState = rememberLazyListState()
    
    LazyColumn(
        state = listState,
        flingBehavior = ScrollableDefaults.flingBehavior()
    ) {
        items(
            items = items,
            key = { it.id }
        ) { item ->
            ItemView(item)
        }
    }
    
    // Prefetch logic happens automatically
    // Adjust prefetch distance if needed
}
```

### Avoid Heavy Operations in Items

```kotlin
// Bad
@Composable
fun HeavyListItem(item: Item) {
    val processedData = processData(item)  // Heavy operation every recomposition
    Text(processedData)
}

// Good
@Composable
fun OptimizedListItem(item: Item) {
    val processedData = remember(item) {
        processData(item)  // Only process when item changes
    }
    Text(processedData)
}
```

## Image Loading

### Coil with Caching

```kotlin
@Composable
fun OptimizedImageList(items: List<Item>) {
    val imageLoader = ImageLoader.Builder(LocalContext.current)
        .memoryCache {
            MemoryCache.Builder(LocalContext.current)
                .maxSizePercent(0.25)  // Use 25% of app memory
                .build()
        }
        .diskCache {
            DiskCache.Builder()
                .directory(LocalContext.current.cacheDir.resolve("image_cache"))
                .maxSizeBytes(50 * 1024 * 1024)  // 50 MB
                .build()
        }
        .build()
    
    LazyColumn {
        items(items) { item ->
            AsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(item.imageUrl)
                    .crossfade(true)
                    .size(400, 400)  // Request specific size
                    .build(),
                contentDescription = null,
                imageLoader = imageLoader
            )
        }
    }
}
```

### Thumbnail Loading

```kotlin
@Composable
fun ImageWithThumbnail(
    thumbnailUrl: String,
    fullImageUrl: String
) {
    var loadFullImage by remember { mutableStateOf(false) }
    
    Box {
        // Load thumbnail first
        AsyncImage(
            model = thumbnailUrl,
            contentDescription = null,
            modifier = Modifier.fillMaxSize()
        )
        
        // Load full image when visible
        if (loadFullImage) {
            AsyncImage(
                model = fullImageUrl,
                contentDescription = null,
                modifier = Modifier.fillMaxSize()
            )
        }
    }
    
    LaunchedEffect(Unit) {
        delay(100)  // Small delay
        loadFullImage = true
    }
}
```

## Memory Management

### Dispose Resources

```kotlin
@Composable
fun ResourceUsingComposable() {
    val resource = remember { HeavyResource() }
    
    DisposableEffect(Unit) {
        onDispose {
            resource.release()
        }
    }
    
    // Use resource
}
```

### Clean Up Flows

```kotlin
@Composable
fun FlowCollector() {
    val viewModel: MyViewModel = hiltViewModel()
    
    LaunchedEffect(Unit) {
        viewModel.dataFlow.collect { data ->
            // Process data
        }
    }
    
    // Flow collection automatically cancelled when composable leaves composition
}
```

### Avoid Memory Leaks

```kotlin
// Bad - Can leak Activity
class BadViewModel(private val context: Context) : ViewModel() {
    // Activity context can leak
}

// Good - Use Application context
class GoodViewModel(
    @ApplicationContext private val context: Context
) : ViewModel() {
    // Application context won't leak
}
```

## State Optimization

### Use Immutable Collections

```kotlin
// Bad - Mutable list triggers recomposition on every change
@Composable
fun BadList() {
    val items = remember { mutableStateListOf<Item>() }
    // Changes to items trigger recomposition of entire list
}

// Good - Immutable list with proper state
@Composable
fun GoodList() {
    var items by remember { mutableStateOf<List<Item>>(emptyList()) }
    // Only recomposes when entire list changes
}
```

### Minimize State Scope

```kotlin
// Bad - Top-level state when not needed
@Composable
fun BadScreen() {
    var selectedId by remember { mutableStateOf<String?>(null) }
    
    LazyColumn {
        items(100) { index ->
            ItemCard(
                item = items[index],
                isSelected = selectedId == items[index].id
            )
        }
    }
}

// Good - State in item
@Composable
fun GoodScreen() {
    LazyColumn {
        items(100) { index ->
            var isSelected by remember { mutableStateOf(false) }
            ItemCard(
                item = items[index],
                isSelected = isSelected,
                onSelect = { isSelected = true }
            )
        }
    }
}
```

## Stability

### Mark Stable Classes

```kotlin
import androidx.compose.runtime.Stable
import androidx.compose.runtime.Immutable

@Stable
data class StableUser(
    val id: String,
    val name: String
)

@Immutable
data class ImmutableConfig(
    val apiUrl: String,
    val timeout: Int
)
```

### Stable Collections

```kotlin
import kotlinx.collections.immutable.ImmutableList
import kotlinx.collections.immutable.persistentListOf

// Add dependency
implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.7")

@Composable
fun StableList(items: ImmutableList<Item>) {
    // ImmutableList is stable - better performance
    LazyColumn {
        items(items.size) { index ->
            ItemView(items[index])
        }
    }
}
```

## Profiling

### Composition Tracing

```kotlin
@Composable
fun TracedComposable() {
    // Enable composition tracing in build.gradle.kts
    // android.enableComposeCompilerMetrics = true
    // android.enableComposeCompilerReports = true
    
    TraceComposition("MyComposable") {
        // Your composable content
    }
}

@Composable
fun TraceComposition(name: String, content: @Composable () -> Unit) {
    androidx.compose.runtime.SideEffect {
        android.os.Trace.beginSection(name)
    }
    
    content()
    
    androidx.compose.runtime.DisposableEffect(Unit) {
        onDispose {
            android.os.Trace.endSection()
        }
    }
}
```

### Recomposition Counter

```kotlin
@Composable
fun RecompositionCounter() {
    val recompositions = remember { mutableStateOf(0) }
    
    SideEffect {
        recompositions.value++
    }
    
    Text("Recompositions: ${recompositions.value}")
}
```

## Advanced Optimizations

### Defer Heavy Operations

```kotlin
@Composable
fun DeferredHeavyContent() {
    var shouldLoad by remember { mutableStateOf(false) }
    
    LaunchedEffect(Unit) {
        delay(100)  // Defer until UI is rendered
        shouldLoad = true
    }
    
    if (shouldLoad) {
        HeavyComposable()
    } else {
        Placeholder()
    }
}
```

### Virtualized Lists

```kotlin
// Use LazyColumn/LazyRow instead of Column/Row with scroll
// Bad
Column(modifier = Modifier.verticalScroll(rememberScrollState())) {
    items.forEach { item ->
        ItemView(item)  // All items created at once
    }
}

// Good
LazyColumn {
    items(items) { item ->
        ItemView(item)  // Only visible items created
    }
}
```

### Avoid Unnecessary Boxing

```kotlin
// Bad - Creates new lambda on every recomposition
@Composable
fun BadClickable(onClick: () -> Unit) {
    Box(modifier = Modifier.clickable { onClick() })
}

// Good - Use remember for stable reference
@Composable
fun GoodClickable(onClick: () -> Unit) {
    val click = remember { onClick }
    Box(modifier = Modifier.clickable(onClick = click))
}
```

## Best Practices

1. ✅ Use `derivedStateOf` for computed values
2. ✅ Add `key` to LazyColumn items
3. ✅ Use `contentType` for mixed lists
4. ✅ Mark data classes as `@Stable` or `@Immutable`
5. ✅ Use `remember` for expensive operations
6. ✅ Avoid reading state during composition
7. ✅ Use `graphicsLayer` for transforms
8. ✅ Defer heavy operations
9. ✅ Use proper image caching
10. ✅ Profile with Composition Tracing
11. ✅ Minimize state scope
12. ✅ Use immutable collections
13. ✅ Dispose resources properly
14. ✅ Avoid creating new objects in composition
15. ✅ Use virtualized lists (LazyColumn)

## Performance Checklist

- [ ] Added keys to all LazyColumn items
- [ ] Used contentType for mixed item types
- [ ] Marked data classes as Stable/Immutable
- [ ] Used derivedStateOf for computed values
- [ ] Remembered expensive calculations
- [ ] Configured image loading cache
- [ ] Disposed resources in DisposableEffect
- [ ] Avoided creating lambdas in composition
- [ ] Used graphicsLayer for transforms
- [ ] Profiled with Composition Tracing
- [ ] Minimized state scope
- [ ] Tested on low-end devices
- [ ] Monitored memory usage
- [ ] Optimized image sizes
- [ ] Deferred heavy operations

## Resources

- [Performance Best Practices](https://developer.android.com/jetpack/compose/performance)
- [Composition Tracing](https://developer.android.com/jetpack/compose/performance/tracing)
- [Stability](https://developer.android.com/jetpack/compose/performance/stability)
- [Performance FAQ](https://developer.android.com/jetpack/compose/performance/faq)

