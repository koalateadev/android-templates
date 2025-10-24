# Compose Rendering Deep Dive

## Overview

Jetpack Compose rendering happens in three distinct phases: Composition, Layout, and Drawing. Understanding these phases is crucial for writing performant Compose apps.

## The Three Phases

```
Composition → Layout → Drawing
     ↓          ↓         ↓
  What?      Where?     How?
```

### 1. Composition Phase

**What**: Decides WHAT to show

```kotlin
@Composable
fun Example() {
    Column {  // ← Composition decides to show Column
        Text("Hello")  // ← and Text
        Button(onClick = {}) {  // ← and Button
            Text("Click")  // ← and nested Text
        }
    }
}
```

**What Happens**:
- Executes `@Composable` functions
- Builds UI tree (node tree)
- Determines which composables to display
- Reads state values

**Output**: UI tree structure

### 2. Layout Phase

**Where**: Decides WHERE to place elements

```kotlin
@Composable
fun LayoutExample() {
    Column(  // ← Layout: stack children vertically
        modifier = Modifier
            .fillMaxWidth()  // ← width: match parent
            .padding(16.dp)  // ← add padding
    ) {
        Text("Top")  // ← positioned at top
        Spacer(modifier = Modifier.height(8.dp))  // ← 8dp space
        Text("Bottom")  // ← positioned below spacer
    }
}
```

**What Happens**:
- Measures each composable
- Determines size and position
- Calculates constraints
- Places children

**Output**: Size and position of each element

### 3. Drawing Phase

**How**: Decides HOW to render

```kotlin
@Composable
fun DrawingExample() {
    Canvas(modifier = Modifier.size(100.dp)) {  // ← Drawing phase
        drawCircle(color = Color.Red)  // ← draws red circle
        drawLine(/* ... */)  // ← draws line
    }
    
    Text(
        "Styled",
        color = Color.Blue,  // ← draws blue text
        fontSize = 20.sp
    )
}
```

**What Happens**:
- Renders pixels
- Draws shapes, text, images
- Applies colors, shadows, borders
- Renders to canvas

**Output**: Pixels on screen

## Recomposition

### What is Recomposition?

Recomposition is when Compose re-executes composables that may have changed, then updates the UI tree to reflect any changes.

### Recomposition Triggers

```kotlin
@Composable
fun CounterExample() {
    var count by remember { mutableStateOf(0) }
    
    // When count changes:
    // 1. Text recomposes (reads count)
    // 2. Button does NOT recompose (doesn't read count)
    
    Column {
        Text("Count: $count")  // ← Recomposes when count changes
        
        Button(onClick = { count++ }) {  // ← Never recomposes
            Text("Increment")  // ← Never recomposes
        }
    }
}
```

### Smart Recomposition

Compose only recomposes:
- Functions that read changed state
- Their parent composables (up to the nearest restart scope)

```kotlin
@Composable
fun SmartRecomposition() {
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    
    Column {
        Text("Name: $name")  // ← Only recomposes when name changes
        Text("Email: $email")  // ← Only recomposes when email changes
        
        TextField(
            value = name,
            onValueChange = { name = it }
        )
        
        TextField(
            value = email,
            onValueChange = { email = it }
        )
    }
}
```

## Skipping

### What is Skipping?

If inputs haven't changed, Compose can skip recomposition entirely.

### Requirements for Skipping

```kotlin
// Stable inputs → Can skip
@Composable
fun Skippable(
    name: String,  // Stable (primitive)
    count: Int     // Stable (primitive)
) {
    Text("$name: $count")
}

// Unstable inputs → Always recomposes
@Composable
fun NotSkippable(
    user: MutableUser  // Unstable (mutable)
) {
    Text(user.name)
}
```

### Make Classes Stable

```kotlin
import androidx.compose.runtime.Stable
import androidx.compose.runtime.Immutable

// Immutable - no properties can change
@Immutable
data class User(
    val id: String,
    val name: String
)

// Stable - properties may change but Compose will be notified
@Stable
data class MutableUser(
    var name: String
) {
    // Moshi notify Compose of changes if using mutableStateOf
}

// Unstable - mutable and Compose won't know about changes
data class UnstableUser(
    var name: String  // var without State
)
```

## Phase Optimization

### Skipping Layout

```kotlin
// Good - Only drawing phase needed
@Composable
fun ColorChange() {
    var color by remember { mutableStateOf(Color.Red) }
    
    Box(
        modifier = Modifier
            .size(100.dp)
            .background(color)  // Only drawing changes
    )
    
    // Changing color skips Layout phase
}
```

### Skipping Composition

```kotlin
// Good - Skips Composition if inputs unchanged
@Composable
fun UserItem(user: User) {  // Stable input
    // If user hasn't changed, entire function skipped
    Text(user.name)
}
```

## Modifier Order Matters

### Modifier Pipeline

```kotlin
// Different results based on order
@Composable
fun ModifierOrder() {
    // Background then padding
    Box(
        Modifier
            .background(Color.Red)
            .padding(16.dp)  // Red background extends to padding
    )
    
    // Padding then background
    Box(
        Modifier
            .padding(16.dp)
            .background(Color.Red)  // Red background inside padding
    )
}
```

### Performance Modifiers

```kotlin
// Use graphicsLayer for better performance
@Composable
fun OptimizedTransform() {
    var rotation by remember { mutableStateOf(0f) }
    
    // Good - Uses graphicsLayer (skips Layout)
    Box(
        Modifier.graphicsLayer {
            rotationZ = rotation
            scaleX = 1.5f
            scaleY = 1.5f
        }
    )
    
    // Avoid - Triggers Layout
    Box(Modifier.rotate(rotation))
}
```

## Composition Local

### How It Works

```kotlin
val LocalUser = compositionLocalOf<User?> { null }

@Composable
fun App() {
    val user = remember { User("1", "John") }
    
    // Provides value down the tree
    CompositionLocalProvider(LocalUser provides user) {
        ChildComposable()  // Can access user
    }
}

@Composable
fun ChildComposable() {
    val user = LocalUser.current  // Reads from nearest provider
    Text(user?.name ?: "Unknown")
}
```

**How it Works**:
- Values stored in composition tree
- Inherited by child composables
- Changes trigger recomposition of readers
- Efficient way to pass data deep

## Remember Mechanics

### remember vs rememberSaveable

```kotlin
@Composable
fun RememberExplained() {
    // Survives recomposition
    // Lost on configuration change
    val state1 = remember { mutableStateOf(0) }
    
    // Survives recomposition
    // Survives configuration changes
    val state2 = rememberSaveable { mutableStateOf(0) }
}
```

**How remember works**:
1. First composition: Calculates and stores value
2. Recomposition: Returns stored value
3. Leaves composition: Value discarded

**How rememberSaveable works**:
1. Stores value in Bundle
2. Survives process death
3. Automatically restored

## Side Effects Lifecycle

### LaunchedEffect

```kotlin
@Composable
fun LaunchedEffectLifecycle(userId: String) {
    LaunchedEffect(userId) {
        // Launched when composable enters composition
        // Cancelled and relaunched when userId changes
        // Cancelled when composable leaves composition
        
        println("Effect started for user: $userId")
        
        try {
            loadUserData(userId)
        } finally {
            println("Effect cancelled")
        }
    }
}
```

### DisposableEffect

```kotlin
@Composable
fun DisposableEffectLifecycle() {
    DisposableEffect(Unit) {
        println("Setup")
        val listener = setupListener()
        
        onDispose {
            println("Cleanup")
            listener.remove()
        }
    }
}
```

## Recomposition Scope

### Restart Scope

```kotlin
@Composable
fun RestartScope() {
    var count by remember { mutableStateOf(0) }
    
    // This composable is a restart scope
    Text("Count: $count")  // ← When count changes, only this Text recomposes
    
    Button(onClick = { count++ }) {
        Text("Increment")  // ← This never recomposes
    }
}
```

### Lambda Restart Scopes

```kotlin
@Composable
fun LambdaScopes() {
    var count by remember { mutableStateOf(0) }
    
    LazyColumn {
        items(10) { index ->
            // This lambda is a restart scope
            Text("Item $index: $count")  // ← All items recompose when count changes
        }
    }
}
```

## Performance Implications

### Expensive Recomposition

```kotlin
// BAD - Expensive operation on every recomposition
@Composable
fun Expensive() {
    val result = expensiveCalculation()  // Runs every recomposition
    Text(result)
}

// GOOD - Cached calculation
@Composable
fun Optimized(data: Data) {
    val result = remember(data) {
        expensiveCalculation(data)  // Only when data changes
    }
    Text(result)
}
```

### Composition vs Recomposition

```kotlin
@Composable
fun Example() {
    println("Composition/Recomposition")  // Runs every time
    
    SideEffect {
        println("After composition")  // Runs after successful composition
    }
    
    DisposableEffect(Unit) {
        println("Setup")  // Runs once
        onDispose {
            println("Cleanup")  // Runs when leaving
        }
    }
}
```

## Key Concepts

### 1. Composables are Functions
Not objects - called every recomposition

### 2. Composition Creates a Tree
UI structure stored in memory

### 3. Recomposition is Optimized
Only changed parts re-execute

### 4. State Drives UI
UI = f(state)

### 5. Side Effects Have Lifecycle
Use proper effect handlers

## Debugging Recomposition

### Count Recompositions

```kotlin
@Composable
fun RecompositionCounter() {
    val count = remember { mutableStateOf(0) }
    
    SideEffect {
        count.value++
    }
    
    Text("Recompositions: ${count.value}")
}
```

### Enable Composition Tracing

In `gradle.properties`:
```properties
# Enable composition tracking
android.enableComposeCompilerMetrics=true
android.enableComposeCompilerReports=true
```

Then check:
- `app/build/compose_metrics/` - Stability reports
- Look for "unstable" classes

## Resources

- [Compose Phases](https://developer.android.com/jetpack/compose/phases)
- [Lifecycle of Composables](https://developer.android.com/jetpack/compose/lifecycle)
- [Performance](https://developer.android.com/jetpack/compose/performance)
- [Thinking in Compose](https://developer.android.com/jetpack/compose/mental-model)
- [Stability](https://developer.android.com/jetpack/compose/performance/stability)

