# Kotlin Configuration

## Overview
Configure Kotlin for optimal Android development with modern features and best practices.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
kotlin = "2.0.20"
agp = "8.5.2"

[plugins]
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
kotlin-parcelize = { id = "org.jetbrains.kotlin.plugin.parcelize", version.ref = "kotlin" }
```

### 2. Configure in `build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.parcelize) // For Parcelable
}

android {
    kotlinOptions {
        jvmTarget = "17"
        
        // Enable explicit API mode for libraries
        freeCompilerArgs += listOf(
            "-opt-in=kotlin.RequiresOptIn",
            "-Xcontext-receivers"
        )
    }
}
```

### 3. Enable Kotlin Features

**Parcelize for easy Parcelable:**

```kotlin
import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class User(
    val id: Long,
    val name: String,
    val email: String
) : Parcelable
```

## Recommended Kotlin Features

### Data Classes

```kotlin
data class User(
    val id: Long,
    val name: String,
    val email: String
) {
    // Automatically provides: equals(), hashCode(), toString(), copy()
}

val user1 = User(1, "John", "john@example.com")
val user2 = user1.copy(name = "Jane") // Immutable update
```

### Sealed Classes

```kotlin
sealed interface Result<out T> {
    data class Success<T>(val data: T) : Result<T>
    data class Error(val exception: Exception) : Result<Nothing>
    data object Loading : Result<Nothing>
}

fun handleResult(result: Result<String>) {
    when (result) {
        is Result.Success -> println(result.data)
        is Result.Error -> println(result.exception)
        Result.Loading -> println("Loading...")
    }
}
```

### Extension Functions

```kotlin
fun String.isValidEmail(): Boolean {
    return android.util.Patterns.EMAIL_ADDRESS.matcher(this).matches()
}

fun View.show() {
    visibility = View.VISIBLE
}

fun View.hide() {
    visibility = View.GONE
}

// Usage
email.isValidEmail()
textView.show()
```

### Null Safety

```kotlin
var name: String = "John"        // Cannot be null
var email: String? = null        // Can be null

// Safe call
val length = email?.length

// Elvis operator
val displayName = name ?: "Unknown"

// Let function
email?.let { 
    sendEmail(it)
}

// Require
fun setUser(name: String?) {
    requireNotNull(name) { "Name cannot be null" }
}
```

### Scope Functions

```kotlin
// let - transform and return
val result = user?.let {
    "${it.name} - ${it.email}"
}

// apply - configure object
val user = User(1, "John", "john@example.com").apply {
    // this = user
}

// also - side effects
val numbers = mutableListOf(1, 2, 3).also {
    println("Initial list: $it")
}

// with - multiple calls on object
with(user) {
    println(name)
    println(email)
}

// run - execute block and return result
val result = user.run {
    "$name: $email"
}
```

### Destructuring

```kotlin
data class Coordinate(val x: Int, val y: Int)

val (x, y) = Coordinate(10, 20)
println("x: $x, y: $y")

// In loops
for ((key, value) in map) {
    println("$key -> $value")
}

// Lambda parameters
map.forEach { (key, value) ->
    println("$key -> $value")
}
```

### Delegation

```kotlin
// Lazy initialization
val database: AppDatabase by lazy {
    Room.databaseBuilder(context, AppDatabase::class.java, "db").build()
}

// Observable property
var name: String by Delegates.observable("Initial") { prop, old, new ->
    println("$old -> $new")
}

// Delegate to another property
class User {
    var name: String = ""
    var displayName by this::name
}
```

## Best Practices

### Use require/check for validations

```kotlin
fun processAge(age: Int) {
    require(age >= 0) { "Age must be non-negative" }
    check(age < 150) { "Invalid age value" }
    // Process age
}
```

### Prefer val over var

```kotlin
// Good
val immutableList = listOf(1, 2, 3)
val user = User("John")

// Avoid when possible
var mutableCount = 0
```

### Use meaningful names

```kotlin
// Bad
val d = 86400

// Good
val SECONDS_IN_DAY = 86400
```

### Avoid !! operator

```kotlin
// Bad
val length = email!!.length

// Good
val length = email?.length ?: 0
```

### Use type inference

```kotlin
// Redundant
val name: String = "John"

// Better
val name = "John"
```

## Modern Kotlin Features

### Context Receivers (Experimental)

```kotlin
context(Context)
fun showToast(message: String) {
    Toast.makeText(this@Context, message, Toast.LENGTH_SHORT).show()
}

// Usage
with(context) {
    showToast("Hello!")
}
```

### Inline Value Classes

```kotlin
@JvmInline
value class UserId(val id: Long)

@JvmInline
value class Email(val value: String)

// Type-safe without runtime overhead
fun getUser(id: UserId): User { ... }
```

### When with subject

```kotlin
val result = when (val response = apiCall()) {
    is Success -> response.data
    is Error -> null
}
```

## Compiler Options

### Suppress Warnings

```kotlin
@Suppress("UNCHECKED_CAST")
fun <T> getList(): List<T> {
    return listOf() as List<T>
}
```

### Opt-in Requirements

```kotlin
@RequiresOptIn(message = "This API is experimental")
@Retention(AnnotationRetention.BINARY)
annotation class ExperimentalApi

@ExperimentalApi
class ExperimentalFeature

// Usage
@OptIn(ExperimentalApi::class)
fun useExperimental() {
    val feature = ExperimentalFeature()
}
```

## Resources

- [Kotlin Documentation](https://kotlinlang.org/docs/home.html)
- [Kotlin for Android](https://developer.android.com/kotlin)
- [Kotlin Style Guide](https://developer.android.com/kotlin/style-guide)

