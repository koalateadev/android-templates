# Unit Testing Setup (JUnit, Mockk)

## Overview
Set up comprehensive unit testing with JUnit 5, Mockk for mocking, and Coroutines Test for asynchronous code.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
junit = "4.13.2"
junit5 = "5.11.3"
mockk = "1.13.13"
truth = "1.4.4"
coroutines-test = "1.9.0"
turbine = "1.1.0"
robolectric = "4.13"

[libraries]
# JUnit 4 (for Android instrumented tests)
junit = { group = "junit", name = "junit", version.ref = "junit" }

# JUnit 5 (recommended for unit tests)
junit-jupiter-api = { group = "org.junit.jupiter", name = "junit-jupiter-api", version.ref = "junit5" }
junit-jupiter-engine = { group = "org.junit.jupiter", name = "junit-jupiter-engine", version.ref = "junit5" }
junit-jupiter-params = { group = "org.junit.jupiter", name = "junit-jupiter-params", version.ref = "junit5" }

# Mockk
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
mockk-android = { group = "io.mockk", name = "mockk-android", version.ref = "mockk" }

# Assertions
truth = { group = "com.google.truth", name = "truth", version.ref = "truth" }

# Coroutines Test
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines-test" }

# Flow Testing
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }

# Robolectric (Android framework in JVM tests)
robolectric = { group = "org.robolectric", name = "robolectric", version.ref = "robolectric" }

[bundles]
unit-testing = [
    "junit-jupiter-api",
    "mockk",
    "truth",
    "kotlinx-coroutines-test",
    "turbine"
]
```

### 2. Configure Gradle

```kotlin
android {
    testOptions {
        unitTests {
            isIncludeAndroidResources = true // For Robolectric
            isReturnDefaultValues = true
        }
    }
}

dependencies {
    // Unit testing
    testImplementation(libs.bundles.unit.testing)
    testRuntimeOnly(libs.junit.jupiter.engine)
    testImplementation(libs.junit.jupiter.params)
    
    // Robolectric (optional)
    testImplementation(libs.robolectric)
}

tasks.withType<Test> {
    useJUnitPlatform() // Enable JUnit 5
}
```

## Basic Test Structure

### Simple Unit Test

```kotlin
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.AfterEach

class CalculatorTest {
    
    private lateinit var calculator: Calculator
    
    @BeforeEach
    fun setup() {
        calculator = Calculator()
    }
    
    @AfterEach
    fun teardown() {
        // Cleanup if needed
    }
    
    @Test
    fun `add returns sum of two numbers`() {
        val result = calculator.add(2, 3)
        assertEquals(5, result)
    }
    
    @Test
    fun `divide throws exception when dividing by zero`() {
        assertThrows(ArithmeticException::class.java) {
            calculator.divide(10, 0)
        }
    }
}
```

### With Google Truth

```kotlin
import com.google.common.truth.Truth.assertThat

@Test
fun `filter returns only even numbers`() {
    val numbers = listOf(1, 2, 3, 4, 5, 6)
    
    val result = numbers.filter { it % 2 == 0 }
    
    assertThat(result).containsExactly(2, 4, 6)
    assertThat(result).hasSize(3)
    assertThat(result).isNotEmpty()
}
```

## Mocking with Mockk

### Basic Mocking

```kotlin
import io.mockk.*
import org.junit.jupiter.api.Test

class UserRepositoryTest {
    
    private val apiService: ApiService = mockk()
    private val userDao: UserDao = mockk()
    private lateinit var repository: UserRepository
    
    @BeforeEach
    fun setup() {
        repository = UserRepository(apiService, userDao)
    }
    
    @Test
    fun `getUser returns user from API`() = runTest {
        // Arrange
        val expectedUser = User(1, "John", "john@test.com")
        coEvery { apiService.getUser(1) } returns Response.success(expectedUser)
        
        // Act
        val result = repository.getUser(1)
        
        // Assert
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()).isEqualTo(expectedUser)
        
        // Verify
        coVerify(exactly = 1) { apiService.getUser(1) }
    }
    
    @Test
    fun `getUser returns error when API fails`() = runTest {
        // Arrange
        coEvery { apiService.getUser(1) } throws IOException("Network error")
        
        // Act
        val result = repository.getUser(1)
        
        // Assert
        assertThat(result.isFailure).isTrue()
    }
}
```

### Relaxed Mocks

```kotlin
// Relaxed mock - returns default values
val apiService: ApiService = mockk(relaxed = true)

// Regular mock - must specify all returns
val apiService: ApiService = mockk()
every { apiService.someMethod() } returns "value"
```

### Spy (Partial Mocking)

```kotlin
val calculator = spyk(Calculator())

// Use real implementation
val sum = calculator.add(2, 3) // Calls real method

// Override specific method
every { calculator.multiply(any(), any()) } returns 100
val product = calculator.multiply(2, 3) // Returns 100
```

### Capture Arguments

```kotlin
@Test
fun `saveUser calls DAO with correct user`() = runTest {
    val userSlot = slot<User>()
    coEvery { userDao.insertUser(capture(userSlot)) } returns 1L
    
    repository.saveUser("John", "john@test.com")
    
    val capturedUser = userSlot.captured
    assertThat(capturedUser.name).isEqualTo("John")
    assertThat(capturedUser.email).isEqualTo("john@test.com")
}
```

## Testing Coroutines

### Basic Coroutine Test

```kotlin
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.Test

@OptIn(ExperimentalCoroutinesApi::class)
class CoroutineTest {
    
    @Test
    fun `test suspend function`() = runTest {
        val result = suspendFunction()
        assertThat(result).isEqualTo("expected")
    }
}
```

### With Test Dispatcher

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.ExperimentalCoroutinesApi
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.AfterEach
import org.junit.jupiter.api.BeforeEach

@OptIn(ExperimentalCoroutinesApi::class)
class ViewModelTest {
    
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var viewModel: UserViewModel
    
    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        viewModel = UserViewModel(mockRepository)
    }
    
    @AfterEach
    fun teardown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `loadUsers updates state to success`() = runTest {
        // Arrange
        val users = listOf(User(1, "John", "john@test.com"))
        coEvery { mockRepository.getUsers() } returns Result.success(users)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle() // Process all pending coroutines
        
        // Assert
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UiState.Success::class.java)
        assertThat((state as UiState.Success).users).isEqualTo(users)
    }
}
```

### Testing Delays

```kotlin
@Test
fun `test function with delay`() = runTest {
    val startTime = currentTime
    
    functionWithDelay() // Has delay(1000)
    
    advanceTimeBy(1000) // Fast-forward time
    
    val elapsed = currentTime - startTime
    assertThat(elapsed).isEqualTo(1000)
}
```

## Testing Flows

### With Turbine

```kotlin
import app.cash.turbine.test
import org.junit.jupiter.api.Test

@Test
fun `flow emits correct values`() = runTest {
    val flow = repository.getUsers()
    
    flow.test {
        // Loading state
        assertThat(awaitItem()).isInstanceOf(UiState.Loading::class.java)
        
        // Success state
        val success = awaitItem()
        assertThat(success).isInstanceOf(UiState.Success::class.java)
        
        awaitComplete()
    }
}
```

### StateFlow Testing

```kotlin
@Test
fun `stateFlow emits updated values`() = runTest {
    viewModel.uiState.test {
        assertThat(awaitItem()).isInstanceOf(UiState.Loading::class.java)
        
        viewModel.loadData()
        
        val success = awaitItem()
        assertThat(success).isInstanceOf(UiState.Success::class.java)
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Parameterized Tests

```kotlin
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource
import org.junit.jupiter.params.provider.CsvSource

class ParameterizedTest {
    
    @ParameterizedTest
    @ValueSource(ints = [2, 4, 6, 8])
    fun `number is even`(number: Int) {
        assertThat(number % 2).isEqualTo(0)
    }
    
    @ParameterizedTest
    @CsvSource(
        "test@example.com, true",
        "invalid-email, false",
        "user@domain.co, true"
    )
    fun `email validation`(email: String, expected: Boolean) {
        val result = EmailValidator.isValid(email)
        assertThat(result).isEqualTo(expected)
    }
}
```

## Testing ViewModels

### Complete Example

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {
    
    private val testDispatcher = StandardTestDispatcher()
    private val mockRepository: UserRepository = mockk()
    private lateinit var viewModel: UserViewModel
    
    @BeforeEach
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        viewModel = UserViewModel(mockRepository)
    }
    
    @AfterEach
    fun teardown() {
        Dispatchers.resetMain()
    }
    
    @Test
    fun `initial state is loading`() {
        assertThat(viewModel.uiState.value).isInstanceOf(UiState.Loading::class.java)
    }
    
    @Test
    fun `loadUsers updates state to success`() = runTest {
        // Arrange
        val users = listOf(User(1, "John", "john@test.com"))
        coEvery { mockRepository.getUsers() } returns Result.success(users)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UiState.Success::class.java)
        assertThat((state as UiState.Success).users).hasSize(1)
    }
    
    @Test
    fun `loadUsers updates state to error on failure`() = runTest {
        // Arrange
        coEvery { mockRepository.getUsers() } returns Result.failure(Exception("Error"))
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UiState.Error::class.java)
        assertThat((state as UiState.Error).message).isEqualTo("Error")
    }
    
    @Test
    fun `loadUsers calls repository`() = runTest {
        // Arrange
        coEvery { mockRepository.getUsers() } returns Result.success(emptyList())
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        coVerify(exactly = 1) { mockRepository.getUsers() }
    }
}
```

## Testing Repositories

```kotlin
class UserRepositoryTest {
    
    private val mockApiService: ApiService = mockk()
    private val mockUserDao: UserDao = mockk(relaxed = true)
    private lateinit var repository: UserRepository
    
    @BeforeEach
    fun setup() {
        repository = UserRepository(mockApiService, mockUserDao)
    }
    
    @Test
    fun `getUser returns cached data when available`() = runTest {
        // Arrange
        val cachedUser = User(1, "John", "john@test.com")
        coEvery { mockUserDao.getUserById(1) } returns cachedUser
        
        // Act
        val result = repository.getUserCached(1)
        
        // Assert
        assertThat(result).isEqualTo(cachedUser)
        coVerify(exactly = 0) { mockApiService.getUser(any()) } // API not called
    }
    
    @Test
    fun `getUser fetches from API when cache is empty`() = runTest {
        // Arrange
        val apiUser = User(1, "Jane", "jane@test.com")
        coEvery { mockUserDao.getUserById(1) } returns null
        coEvery { mockApiService.getUser(1) } returns Response.success(apiUser)
        
        // Act
        val result = repository.getUserCached(1)
        
        // Assert
        assertThat(result).isEqualTo(apiUser)
        coVerify { mockUserDao.insertUser(apiUser) } // Verify cache update
    }
}
```

## Test Organization

### Nested Tests

```kotlin
import org.junit.jupiter.api.Nested

class UserServiceTest {
    
    @Nested
    inner class GetUserTests {
        @Test
        fun `returns user when found`() { }
        
        @Test
        fun `throws exception when not found`() { }
    }
    
    @Nested
    inner class CreateUserTests {
        @Test
        fun `creates user with valid data`() { }
        
        @Test
        fun `throws exception with invalid email`() { }
    }
}
```

## Best Practices

- Follow AAA pattern (Arrange, Act, Assert)
- Use descriptive test names
- Test one thing per test
- Use mockk for dependencies
- Test edge cases and error scenarios
- Use parameterized tests for multiple inputs
- Keep tests independent
- Use TestDispatcher for coroutines

## Test Coverage

### Generate Coverage Report

```kotlin
android {
    buildTypes {
        debug {
            enableUnitTestCoverage = true
        }
    }
}
```

Run: `./gradlew testDebugUnitTestCoverage`

## Resources

- [JUnit 5 Documentation](https://junit.org/junit5/docs/current/user-guide/)
- [Mockk Documentation](https://mockk.io/)
- [Coroutines Test Guide](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)
- [Android Testing](https://developer.android.com/training/testing)

