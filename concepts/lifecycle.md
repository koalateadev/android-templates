# Lifecycle Deep Dive

## Overview

Understanding lifecycle is fundamental to Android development. This guide explains how lifecycles work across different Android components.

## Activity Lifecycle

### Lifecycle States

```
Created → Started → Resumed → Paused → Stopped → Destroyed
```

### Lifecycle Methods

```kotlin
class MyActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Called when activity is first created
        // Initialize views, ViewModels
        // Runs once per activity instance
    }
    
    override fun onStart() {
        super.onStart()
        // Called when activity becomes visible
        // Start animations, register listeners
        // May be called multiple times
    }
    
    override fun onResume() {
        super.onResume()
        // Called when activity is ready for user interaction
        // Start camera, location updates, etc.
        // Activity is in foreground
    }
    
    override fun onPause() {
        super.onPause()
        // Called when activity is partially obscured
        // Stop animations, pause video
        // Save draft data
        // Should be fast (< few hundred ms)
    }
    
    override fun onStop() {
        super.onStop()
        // Called when activity is no longer visible
        // Release heavy resources
        // Stop location updates
        // Activity may be killed from this state
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // Called before activity is destroyed
        // Final cleanup
        // May not always be called (process killed)
    }
}
```

### What Triggers Lifecycle Changes

| Trigger | Lifecycle Change |
|---------|------------------|
| App launched | onCreate() → onStart() → onResume() |
| Screen off | onPause() → onStop() |
| Screen on | onRestart() → onStart() → onResume() |
| Home button | onPause() → onStop() |
| Back button | onPause() → onStop() → onDestroy() |
| Rotation | onPause() → onStop() → onDestroy() → onCreate() → onStart() → onResume() |
| Dialog shown | No change (activity still visible) |
| Another activity | onPause() → onStop() |

## Fragment Lifecycle

### Fragment Lifecycle Methods

```kotlin
class MyFragment : Fragment() {
    
    override fun onAttach(context: Context) {
        super.onAttach(context)
        // Fragment attached to activity
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Fragment created
        // Initialize non-view resources
    }
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        // Create fragment's view hierarchy
        return inflater.inflate(R.layout.fragment_my, container, false)
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // View is created
        // Safe to access views
        // Initialize UI components
    }
    
    override fun onStart() {
        super.onStart()
        // Fragment visible
    }
    
    override fun onResume() {
        super.onResume()
        // Fragment ready for interaction
    }
    
    override fun onPause() {
        super.onPause()
        // Fragment partially obscured
    }
    
    override fun onStop() {
        super.onStop()
        // Fragment no longer visible
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        // View hierarchy destroyed
        // Clean up view references
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // Fragment destroyed
    }
    
    override fun onDetach() {
        super.onDetach()
        // Fragment detached from activity
    }
}
```

### Fragment vs Activity Lifecycle

Fragment has additional states between onCreate and onDestroy:
- `onCreateView()` - Create view
- `onViewCreated()` - View ready
- `onDestroyView()` - View destroyed (but Fragment may survive)

## ViewModel Lifecycle

### ViewModel Scope

```kotlin
class MyViewModel : ViewModel() {
    
    init {
        // Called when ViewModel is first created
        loadData()
    }
    
    fun loadData() {
        viewModelScope.launch {
            // Coroutine automatically cancelled when ViewModel cleared
            repository.getData()
        }
    }
    
    override fun onCleared() {
        super.onCleared()
        // Called when ViewModel is about to be destroyed
        // Clean up resources
        // Don't reference UI here
    }
}
```

### ViewModel Survives Configuration Changes

```
Activity: onCreate → onDestroy (rotation)
           ↓              ↓
ViewModel: created -------- survives -------- still alive
           ↓                                   ↓
Activity: onCreate (new) -------------------- can access ViewModel
```

### ViewModel Lifecycle Scope

```kotlin
class TimerViewModel : ViewModel() {
    private val timer = Timer()
    
    init {
        // Start timer when ViewModel created
        timer.scheduleAtFixedRate(object : TimerTask() {
            override fun run() {
                updateTime()
            }
        }, 0, 1000)
    }
    
    override fun onCleared() {
        // Cancel timer when ViewModel destroyed
        timer.cancel()
        super.onCleared()
    }
}
```

## Composable Lifecycle

### Composition Lifecycle

```
Enters Composition → Recomposes (0+ times) → Leaves Composition
```

### Lifecycle Events

```kotlin
@Composable
fun LifecycleAwareComposable() {
    // Runs when composable enters composition
    DisposableEffect(Unit) {
        println("Composable entered composition")
        
        // Cleanup when composable leaves composition
        onDispose {
            println("Composable left composition")
        }
    }
    
    // Runs after every successful composition
    SideEffect {
        println("Composable committed")
    }
    
    // Runs when key changes
    LaunchedEffect(someKey) {
        println("Key changed, running effect")
    }
}
```

### Recomposition

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    // Initial composition: Creates UI tree
    // Recomposition: Updates only changed parts
    
    Text("Count: $count")  // Recomposes when count changes
    Button(onClick = { count++ }) {  // Button doesn't recompose
        Text("Increment")  // Text inside doesn't recompose
    }
}
```

## Lifecycle-Aware Components

### LifecycleObserver

```kotlin
class MyObserver : DefaultLifecycleObserver {
    
    override fun onCreate(owner: LifecycleOwner) {
        // Activity/Fragment onCreate
    }
    
    override fun onStart(owner: LifecycleOwner) {
        // Activity/Fragment onStart
    }
    
    override fun onResume(owner: LifecycleOwner) {
        // Activity/Fragment onResume
    }
    
    override fun onPause(owner: LifecycleOwner) {
        // Activity/Fragment onPause
    }
    
    override fun onStop(owner: LifecycleOwner) {
        // Activity/Fragment onStop
    }
    
    override fun onDestroy(owner: LifecycleOwner) {
        // Activity/Fragment onDestroy
    }
}

// Register observer
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(MyObserver())
    }
}
```

### In Compose

```kotlin
@Composable
fun LifecycleAwareScreen() {
    val lifecycleOwner = LocalLifecycleOwner.current
    
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_CREATE -> println("Screen created")
                Lifecycle.Event.ON_START -> println("Screen started")
                Lifecycle.Event.ON_RESUME -> println("Screen resumed")
                Lifecycle.Event.ON_PAUSE -> println("Screen paused")
                Lifecycle.Event.ON_STOP -> println("Screen stopped")
                Lifecycle.Event.ON_DESTROY -> println("Screen destroyed")
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

## Process Lifecycle

### Process States

1. **Foreground** - User is interacting
2. **Visible** - User can see but not interacting
3. **Service** - Background service running
4. **Cached** - Not needed, can be killed
5. **Empty** - No components, killed first

### Process Death and Restoration

```kotlin
class MyActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Restore state if process was killed
        savedInstanceState?.let { bundle ->
            val savedData = bundle.getString("key")
            // Restore UI state
        }
    }
    
    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        // Save state before potential process death
        outState.putString("key", "value")
    }
}
```

## Lifecycle Best Practices

### What to Do in Each Method

| Method | What to Do | What NOT to Do |
|--------|------------|----------------|
| `onCreate()` | Initialize, set content view | Long operations, network calls |
| `onStart()` | Register listeners | Heavy processing |
| `onResume()` | Start camera, location | Database operations |
| `onPause()` | Save drafts, pause media | Long operations (< 300ms) |
| `onStop()` | Release resources | Depend on this being called |
| `onDestroy()` | Final cleanup | Count on this being called |

### Lifecycle-Aware Data Collection

```kotlin
// Activity/Fragment
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}

// Compose
@Composable
fun Screen(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // Automatically stops collection when screen is in background
}
```

## Common Lifecycle Issues

### Memory Leaks

```kotlin
// BAD - Leaks Activity
class LeakyViewModel(private val activity: Activity) : ViewModel() {
    // Activity reference prevents garbage collection
}

// GOOD - Use Application context
class ProperViewModel(
    @ApplicationContext private val context: Context
) : ViewModel() {
    // Application context is safe
}
```

### Unregistered Listeners

```kotlin
// BAD - Listener never unregistered
class BadActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        someService.registerListener(listener)
        // Memory leak!
    }
}

// GOOD - Properly managed
class GoodActivity : AppCompatActivity() {
    private val observer = MyObserver()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycle.addObserver(observer)
        // Automatically managed
    }
}
```

### Configuration Changes

```kotlin
// Handle rotation properly
class MyActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    
    // ViewModel survives rotation automatically
    // No need for onRetainNonConfigurationInstance
}
```

## Summary

**Activity/Fragment:**
- onCreate → onStart → onResume → onPause → onStop → onDestroy
- onDestroy may be skipped if process killed
- Keep onPause fast

**ViewModel:**
- Survives configuration changes
- Cleared when activity finished

**Composable:**
- Enters → Recomposes → Leaves
- Use DisposableEffect for cleanup

**Process:**
- Can be killed anytime
- Save state in onSaveInstanceState

## Resources

- [Activity Lifecycle](https://developer.android.com/guide/components/activities/activity-lifecycle)
- [Fragment Lifecycle](https://developer.android.com/guide/fragments/lifecycle)
- [ViewModel Lifecycle](https://developer.android.com/topic/libraries/architecture/viewmodel)
- [Lifecycle-Aware Components](https://developer.android.com/topic/libraries/architecture/lifecycle)
- [Compose Lifecycle](https://developer.android.com/jetpack/compose/lifecycle)

