# Coroutines & Flow Internals

## Overview

Understanding how coroutines and Flow work internally helps you write better asynchronous code and debug issues effectively.

## Coroutine Basics

### What is a Coroutine?

A coroutine is a **suspendable computation** - it can pause execution and resume later without blocking a thread.

```kotlin
suspend fun fetchData(): String {
    delay(1000)  // Suspends, doesn't block
    return "Data"
}

// Under the hood, this becomes a state machine
```

### Suspend Functions

```kotlin
suspend fun example() {
    println("Before")
    delay(1000)  // Suspension point
    println("After")
}

// Compiled to state machine:
// State 0: "Before" → call delay → State 1
// State 1: "After" → return
```

## Coroutine Context

### Context Components

```kotlin
val context = Job() + Dispatchers.Main + CoroutineName("MyCoroutine")

// Job - Controls lifecycle
// Dispatcher - Determines thread
// CoroutineName - For debugging
// CoroutineExceptionHandler - Handles exceptions
```

### Context Hierarchy

```kotlin
val parentContext = Job() + Dispatchers.Main

launch(parentContext) {
    // Inherits parentContext
    println(coroutineContext[Job])  // Parent Job
    println(coroutineContext[CoroutineDispatcher])  // Dispatchers.Main
    
    withContext(Dispatchers.IO) {
        // New context with different dispatcher
        println(coroutineContext[CoroutineDispatcher])  // Dispatchers.IO
        println(coroutineContext[Job])  // Still same Job hierarchy
    }
}
```

## Dispatchers

### How Dispatchers Work

```kotlin
// Dispatchers.Main
// - Uses Android main thread (Looper)
// - UI operations only
// - Created from Handler

// Dispatchers.IO
// - Thread pool (64 threads max)
// - For blocking I/O operations
// - Shared pool for all IO operations

// Dispatchers.Default
// - Thread pool (CPU cores count)
// - For CPU-intensive work
// - Computational tasks

// Dispatchers.Unconfined
// - Starts on caller thread
// - Resumes on whatever thread suspended it
// - Don't use in production
```

### Thread Switching

```kotlin
suspend fun example() {
    // Main thread
    withContext(Dispatchers.IO) {
        // IO thread pool
        val data = fetchFromNetwork()
        
        withContext(Dispatchers.Default) {
            // Default thread pool
            val processed = processData(data)
            
            withContext(Dispatchers.Main) {
                // Back to main thread
                updateUI(processed)
            }
        }
    }
}
```

## Structured Concurrency

### Job Hierarchy

```kotlin
val parentJob = Job()

val job1 = CoroutineScope(parentJob).launch {
    // Child of parentJob
    val job2 = launch {
        // Child of job1, grandchild of parentJob
    }
}

// Cancelling parentJob cancels all children
parentJob.cancel()  // Cancels job1 and job2
```

### Parent-Child Relationship

```
ParentJob
    ├─ ChildJob1
    │   ├─ GrandchildJob1
    │   └─ GrandchildJob2
    └─ ChildJob2

Cancel parent → All children cancelled
Child fails → Parent fails (unless SupervisorJob)
```

### coroutineScope vs supervisorScope

```kotlin
// coroutineScope - One failure cancels all
suspend fun parallelTasks() = coroutineScope {
    val job1 = async { task1() }
    val job2 = async { task2() }  // If task2 fails, task1 cancelled
    
    job1.await() + job2.await()
}

// supervisorScope - Failures are independent
suspend fun independentTasks() = supervisorScope {
    val job1 = async { task1() }
    val job2 = async { task2() }  // If task2 fails, task1 continues
    
    // Must handle failures individually
    val result1 = try { job1.await() } catch (e: Exception) { null }
    val result2 = try { job2.await() } catch (e: Exception) { null }
}
```

## Cancellation

### How Cancellation Works

```kotlin
val job = launch {
    try {
        while (isActive) {  // Check if cancelled
            doWork()
        }
    } finally {
        // Cleanup always runs
        cleanup()
    }
}

delay(1000)
job.cancel()  // Requests cancellation
job.join()    // Waits for completion
```

### Cooperative Cancellation

```kotlin
// BAD - Ignores cancellation
suspend fun ignoreCancellation() {
    while (true) {  // Never checks isActive
        doWork()
    }
}

// GOOD - Checks cancellation
suspend fun cooperativeCancellation() {
    while (isActive) {  // Checks cancellation
        doWork()
    }
}

// GOOD - Uses cancellable suspension
suspend fun withCancellableOps() {
    repeat(100) {
        delay(100)  // Cancellable suspension point
        doWork()
    }
}
```

### Cancellation Propagation

```
Parent cancelled
    ↓
Children notified (CancellationException)
    ↓
Children suspend functions throw CancellationException
    ↓
finally blocks execute
    ↓
Job moves to Cancelled state
```

## Flow Internals

### Cold vs Hot Flows

```kotlin
// Cold Flow - New execution for each collector
fun coldFlow() = flow {
    println("Flow started")  // Printed for each collector
    emit(1)
    emit(2)
}

// Each collection creates new execution
coldFlow().collect { println(it) }  // Prints: Flow started, 1, 2
coldFlow().collect { println(it) }  // Prints: Flow started, 1, 2

// Hot Flow - Shared execution
val hotFlow = MutableSharedFlow<Int>()
hotFlow.emit(1)  // Emitted regardless of collectors

// StateFlow - Hot flow with current value
val stateFlow = MutableStateFlow(0)
stateFlow.value = 1  // Updates regardless of collectors
```

### Flow Operators

```kotlin
// Operators create new flows
val original = flowOf(1, 2, 3)

val mapped = original.map { it * 2 }  // New flow
// Original: 1 → 2 → 3
// Mapped:   2 → 4 → 6

val filtered = original.filter { it > 1 }  // New flow
// Original: 1 → 2 → 3
// Filtered:     2 → 3
```

### Flow Collection

```kotlin
val flow = flow {
    emit(1)
    delay(100)
    emit(2)
}

// collect() is a suspend function
flow.collect { value ->
    // This lambda runs for each emission
    // Suspends until flow completes or is cancelled
    println(value)
}
```

## StateFlow vs SharedFlow

### StateFlow

```kotlin
val stateFlow = MutableStateFlow(0)

// Characteristics:
// 1. Always has a value (current state)
// 2. Conflates values (only latest matters)
// 3. Replay = 1 (new collectors get current value)
// 4. Distinct values only (doesn't emit same value twice)

stateFlow.value = 1  // Updates
stateFlow.value = 1  // Ignored (same value)
stateFlow.value = 2  // Updates

// New collector immediately gets 2
```

### SharedFlow

```kotlin
val sharedFlow = MutableSharedFlow<Int>(
    replay = 0,        // How many previous emissions to replay
    extraBufferCapacity = 64,  // Buffer for slow collectors
    onBufferOverflow = BufferOverflow.DROP_OLDEST
)

// Characteristics:
// 1. No initial value required
// 2. Can emit duplicate values
// 3. Configurable replay
// 4. Supports multiple collectors

sharedFlow.emit(1)
sharedFlow.emit(1)  // Both emissions delivered
```

## Channels

### How Channels Work

```kotlin
val channel = Channel<Int>(capacity = Channel.UNLIMITED)

// Producer
launch {
    channel.send(1)  // Suspends if buffer full
    channel.send(2)
    channel.close()  // Signal completion
}

// Consumer
launch {
    for (value in channel) {  // Receives until closed
        println(value)
    }
}
```

### Channel Types

```kotlin
// Rendezvous (capacity = 0) - Direct handoff
val rendezvous = Channel<Int>()

// Buffered - Fixed buffer
val buffered = Channel<Int>(10)

// Unlimited - No size limit
val unlimited = Channel<Int>(Channel.UNLIMITED)

// Conflated - Only latest value
val conflated = Channel<Int>(Channel.CONFLATED)
```

## Exception Handling

### How Exceptions Propagate

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught: $exception")
}

val scope = CoroutineScope(Job() + handler)

scope.launch {
    throw RuntimeException("Error")  // Caught by handler
}

// Exception propagation:
// 1. Exception thrown
// 2. Propagates to parent
// 3. Cancels siblings
// 4. Caught by CoroutineExceptionHandler
```

### SupervisorJob

```kotlin
// Regular Job - one child fails, all fail
val job = Job()
CoroutineScope(job).launch {
    launch { throw Exception() }  // Fails
    launch { delay(5000) }  // Cancelled!
}

// SupervisorJob - children fail independently
val supervisor = SupervisorJob()
CoroutineScope(supervisor).launch {
    launch { throw Exception() }  // Fails
    launch { delay(5000) }  // Continues!
}
```

## Flow Backpressure

### How Flow Handles Backpressure

```kotlin
// Suspending collection naturally handles backpressure
flow {
    repeat(100) {
        emit(it)  // Suspends until collector ready
    }
}.collect { value ->
    delay(100)  // Slow collector
    // Producer waits for collector
}
```

### Buffer Operator

```kotlin
flow {
    repeat(100) {
        emit(it)
    }
}
.buffer(50)  // Buffer up to 50 items
.collect { value ->
    delay(100)  // Producer doesn't wait
}
```

### Conflate Operator

```kotlin
flow {
    repeat(100) {
        emit(it)
        delay(10)
    }
}
.conflate()  // Skip intermediate values
.collect { value ->
    delay(100)  // Only processes latest
}
```

## Advanced Concepts

### Flow Sharing

```kotlin
val flow = flow {
    emit(fetchData())  // Expensive operation
}

// Without sharing - fetches for each collector
flow.collect { }  // Fetch 1
flow.collect { }  // Fetch 2

// With sharing - single execution
val sharedFlow = flow.shareIn(
    scope = CoroutineScope(Dispatchers.IO),
    started = SharingStarted.WhileSubscribed(),
    replay = 1
)

sharedFlow.collect { }  // Share execution
sharedFlow.collect { }  // Share execution
```

### StateIn vs ShareIn

```kotlin
// stateIn - Converts Flow to StateFlow
val stateFlow: StateFlow<Data> = dataFlow
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyData
    )

// shareIn - Converts Flow to SharedFlow
val sharedFlow: SharedFlow<Data> = dataFlow
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        replay = 0
    )
```

## Common Misconceptions

### ❌ Coroutines are Lightweight Threads
**Reality**: Coroutines run on threads but many coroutines can share one thread

### ❌ suspend Functions Run in Background
**Reality**: They can suspend but run on whatever dispatcher they're launched with

### ❌ Flow is Always Asynchronous
**Reality**: Flow can be synchronous unless you use flowOn() or launch

### ❌ Cancellation is Immediate
**Reality**: Cancellation is cooperative - requires suspension points

## Performance Insights

### Coroutine Overhead

- Creating a coroutine: ~100 bytes
- Suspending: ~10-50 bytes
- Very lightweight compared to threads (MB each)

### Flow Performance

- Cold flows: No overhead when not collected
- Hot flows: Small memory for buffer/replay
- Operators: Minimal overhead (inline functions)

## Key Takeaways

1. **Coroutines suspend, don't block**
   - Can pause and resume
   - Thread can work on other tasks

2. **Structured concurrency**
   - Parent-child relationships
   - Automatic cancellation propagation
   - No leaked coroutines

3. **Dispatchers control execution**
   - Main: UI thread
   - IO: Blocking I/O
   - Default: CPU work

4. **Flow is cold by default**
   - New execution per collector
   - Convert to hot with shareIn/stateIn

5. **Cancellation is cooperative**
   - Check isActive
   - Use cancellable suspension points
   - Clean up in finally blocks

6. **Context is inherited**
   - Children inherit parent context
   - Can override specific elements

## Resources

- [Coroutines Guide](https://kotlinlang.org/docs/coroutines-guide.html)
- [Coroutines Internals](https://kotlinlang.org/docs/coroutines-internals.html)
- [Flow Documentation](https://kotlinlang.org/docs/flow.html)
- [Under the Hood](https://elizarov.medium.com/coroutines-under-the-hood-8062038c06f0)
- [Kotlin Conf Talks](https://www.youtube.com/c/Kotlin/search?query=coroutines)

