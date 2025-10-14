# Testing Patterns

## Table of Contents
- [Test Data Builders](#test-data-builders)
- [Robot Pattern](#robot-pattern)
- [Fake Repositories](#fake-repositories)
- [ViewModel Testing](#viewmodel-testing)
- [Compose UI Testing](#compose-ui-testing)
- [Flow Testing](#flow-testing)
- [Repository Testing](#repository-testing)

## Test Data Builders

### Builder Pattern

```kotlin
class UserTestBuilder {
    private var id: String = "test-id"
    private var name: String = "Test User"
    private var email: String = "test@example.com"
    private var isActive: Boolean = true
    
    fun withId(id: String) = apply { this.id = id }
    fun withName(name: String) = apply { this.name = name }
    fun withEmail(email: String) = apply { this.email = email }
    fun inactive() = apply { this.isActive = false }
    
    fun build() = User(
        id = id,
        name = name,
        email = email,
        isActive = isActive
    )
}

// Usage
val testUser = UserTestBuilder()
    .withName("John Doe")
    .withEmail("john@test.com")
    .build()

val inactiveUser = UserTestBuilder()
    .withId("user-2")
    .inactive()
    .build()
```

### Factory Functions

```kotlin
object TestDataFactory {
    
    fun createUser(
        id: String = "test-id-${Random.nextInt()}",
        name: String = "Test User",
        email: String = "test@example.com",
        isActive: Boolean = true
    ) = User(id, name, email, isActive)
    
    fun createUsers(count: Int = 5): List<User> {
        return List(count) { index ->
            createUser(
                id = "user-$index",
                name = "User $index",
                email = "user$index@test.com"
            )
        }
    }
    
    fun createUserWithPosts(postCount: Int = 3): UserWithPosts {
        val user = createUser()
        val posts = List(postCount) { index ->
            Post(
                id = "post-$index",
                userId = user.id,
                title = "Post $index",
                content = "Content $index"
            )
        }
        return UserWithPosts(user, posts)
    }
}
```

## Robot Pattern

### UI Test Robot

```kotlin
class LoginRobot {
    
    fun enterEmail(email: String) = apply {
        composeTestRule
            .onNodeWithTag("email_input")
            .performTextInput(email)
    }
    
    fun enterPassword(password: String) = apply {
        composeTestRule
            .onNodeWithTag("password_input")
            .performTextInput(password)
    }
    
    fun clickLogin() = apply {
        composeTestRule
            .onNodeWithTag("login_button")
            .performClick()
    }
    
    fun assertLoading() = apply {
        composeTestRule
            .onNodeWithTag("loading_indicator")
            .assertIsDisplayed()
    }
    
    fun assertError(message: String) = apply {
        composeTestRule
            .onNodeWithText(message)
            .assertIsDisplayed()
    }
    
    fun assertSuccess() = apply {
        composeTestRule
            .onNodeWithText("Login successful")
            .assertIsDisplayed()
    }
}

// Usage in tests
@Test
fun testLogin() {
    LoginRobot()
        .enterEmail("test@example.com")
        .enterPassword("password123")
        .clickLogin()
        .assertSuccess()
}

@Test
fun testLoginError() {
    LoginRobot()
        .enterEmail("invalid")
        .enterPassword("wrong")
        .clickLogin()
        .assertError("Invalid credentials")
}
```

### Screen Robot

```kotlin
class UserListRobot(private val composeTestRule: ComposeContentTestRule) {
    
    fun waitForUsers() = apply {
        composeTestRule.waitUntil(timeoutMillis = 5000) {
            composeTestRule
                .onAllNodesWithTag("user_item")
                .fetchSemanticsNodes()
                .isNotEmpty()
        }
    }
    
    fun assertUserDisplayed(userName: String) = apply {
        composeTestRule
            .onNodeWithText(userName)
            .assertIsDisplayed()
    }
    
    fun clickUser(userName: String) = apply {
        composeTestRule
            .onNodeWithText(userName)
            .performClick()
    }
    
    fun pullToRefresh() = apply {
        composeTestRule
            .onNodeWithTag("user_list")
            .performTouchInput {
                swipeDown()
            }
    }
    
    fun deleteUser(userName: String) = apply {
        composeTestRule
            .onNodeWithText(userName)
            .performTouchInput {
                swipeLeft()
            }
    }
}
```

## Fake Repositories

### Fake Repository Implementation

```kotlin
class FakeUserRepository : UserRepository {
    
    private val users = mutableListOf<User>()
    private var shouldReturnError = false
    
    fun setUsers(userList: List<User>) {
        users.clear()
        users.addAll(userList)
    }
    
    fun setShouldReturnError(value: Boolean) {
        shouldReturnError = value
    }
    
    override suspend fun getUsers(): Result<List<User>> {
        return if (shouldReturnError) {
            Result.failure(Exception("Test error"))
        } else {
            Result.success(users.toList())
        }
    }
    
    override suspend fun getUserById(id: String): Result<User> {
        val user = users.find { it.id == id }
        return if (user != null) {
            Result.success(user)
        } else {
            Result.failure(Exception("User not found"))
        }
    }
    
    override suspend fun createUser(user: User): Result<User> {
        users.add(user)
        return Result.success(user)
    }
    
    override suspend fun updateUser(user: User): Result<User> {
        val index = users.indexOfFirst { it.id == user.id }
        if (index != -1) {
            users[index] = user
            return Result.success(user)
        }
        return Result.failure(Exception("User not found"))
    }
    
    override suspend fun deleteUser(id: String): Result<Unit> {
        users.removeIf { it.id == id }
        return Result.success(Unit)
    }
}
```

### Fake with Flow

```kotlin
class FakeFlowRepository : UserRepository {
    
    private val _usersFlow = MutableStateFlow<List<User>>(emptyList())
    
    fun emitUsers(users: List<User>) {
        _usersFlow.value = users
    }
    
    override fun observeUsers(): Flow<List<User>> {
        return _usersFlow.asStateFlow()
    }
}
```

## ViewModel Testing

### Complete ViewModel Test

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserViewModelTest {
    
    private val testDispatcher = StandardTestDispatcher()
    private lateinit var viewModel: UserViewModel
    private lateinit var fakeRepository: FakeUserRepository
    
    @Before
    fun setup() {
        Dispatchers.setMain(testDispatcher)
        fakeRepository = FakeUserRepository()
        viewModel = UserViewModel(fakeRepository)
    }
    
    @After
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
        val testUsers = TestDataFactory.createUsers(3)
        fakeRepository.setUsers(testUsers)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UiState.Success::class.java)
        assertThat((state as UiState.Success).users).hasSize(3)
    }
    
    @Test
    fun `loadUsers updates state to error on failure`() = runTest {
        // Arrange
        fakeRepository.setShouldReturnError(true)
        
        // Act
        viewModel.loadUsers()
        advanceUntilIdle()
        
        // Assert
        val state = viewModel.uiState.value
        assertThat(state).isInstanceOf(UiState.Error::class.java)
    }
    
    @Test
    fun `deleteUser emits success event`() = runTest {
        // Arrange
        val testUser = TestDataFactory.createUser(id = "user-1")
        fakeRepository.setUsers(listOf(testUser))
        
        val effects = mutableListOf<UserEffect>()
        val job = launch {
            viewModel.effect.collect { effects.add(it) }
        }
        
        // Act
        viewModel.deleteUser(testUser.id)
        advanceUntilIdle()
        
        // Assert
        assertThat(effects).contains(UserEffect.ShowSnackbar("User deleted"))
        
        job.cancel()
    }
}
```

### Test with Turbine

```kotlin
import app.cash.turbine.test

@Test
fun `state flow emits correct values`() = runTest {
    fakeRepository.setUsers(TestDataFactory.createUsers(2))
    
    viewModel.uiState.test {
        // Initial loading state
        assertThat(awaitItem()).isInstanceOf(UiState.Loading::class.java)
        
        // Load users
        viewModel.loadUsers()
        
        // Success state
        val success = awaitItem()
        assertThat(success).isInstanceOf(UiState.Success::class.java)
        assertThat((success as UiState.Success).users).hasSize(2)
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Compose UI Testing

### Complete UI Test

```kotlin
@RunWith(AndroidJUnit4::class)
class UserScreenTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    private lateinit var fakeRepository: FakeUserRepository
    private lateinit var viewModel: UserViewModel
    
    @Before
    fun setup() {
        fakeRepository = FakeUserRepository()
        viewModel = UserViewModel(fakeRepository)
    }
    
    @Test
    fun displaysLoadingState() {
        composeTestRule.setContent {
            UserScreen(viewModel)
        }
        
        composeTestRule
            .onNodeWithTag("loading_indicator")
            .assertIsDisplayed()
    }
    
    @Test
    fun displaysUserList() {
        val testUsers = TestDataFactory.createUsers(3)
        fakeRepository.setUsers(testUsers)
        
        composeTestRule.setContent {
            UserScreen(viewModel)
        }
        
        composeTestRule.waitForIdle()
        
        testUsers.forEach { user ->
            composeTestRule
                .onNodeWithText(user.name)
                .assertIsDisplayed()
        }
    }
    
    @Test
    fun displaysErrorState() {
        fakeRepository.setShouldReturnError(true)
        
        composeTestRule.setContent {
            UserScreen(viewModel)
        }
        
        composeTestRule.waitForIdle()
        
        composeTestRule
            .onNodeWithText("Test error")
            .assertIsDisplayed()
        
        composeTestRule
            .onNodeWithText("Retry")
            .assertIsDisplayed()
    }
    
    @Test
    fun retryButtonReloadsData() {
        fakeRepository.setShouldReturnError(true)
        
        composeTestRule.setContent {
            UserScreen(viewModel)
        }
        
        composeTestRule.waitForIdle()
        
        // Fix the error condition
        fakeRepository.setShouldReturnError(false)
        fakeRepository.setUsers(TestDataFactory.createUsers(1))
        
        // Click retry
        composeTestRule
            .onNodeWithText("Retry")
            .performClick()
        
        composeTestRule.waitForIdle()
        
        // Verify success state
        composeTestRule
            .onNodeWithText("Test error")
            .assertDoesNotExist()
    }
}
```

## Flow Testing

### Test StateFlow

```kotlin
@Test
fun `stateFlow emits correct values`() = runTest {
    val viewModel = UserViewModel(fakeRepository)
    
    viewModel.uiState.test {
        // Initial state
        assertThat(awaitItem()).isInstanceOf(UiState.Loading::class.java)
        
        // Trigger load
        viewModel.loadUsers()
        
        // Success state
        val success = awaitItem()
        assertThat(success).isInstanceOf(UiState.Success::class.java)
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

### Test SharedFlow

```kotlin
@Test
fun `effect emits on delete`() = runTest {
    val viewModel = UserViewModel(fakeRepository)
    
    viewModel.effect.test {
        // Perform action
        viewModel.deleteUser("user-1")
        
        // Verify effect
        val effect = awaitItem()
        assertThat(effect).isInstanceOf(UserEffect.ShowSnackbar::class.java)
        assertThat((effect as UserEffect.ShowSnackbar).message)
            .isEqualTo("User deleted")
        
        cancelAndIgnoreRemainingEvents()
    }
}
```

## Repository Testing

### Test with MockWebServer

```kotlin
class UserRepositoryTest {
    
    private lateinit var mockWebServer: MockWebServer
    private lateinit var apiService: ApiService
    private lateinit var repository: UserRepository
    
    @Before
    fun setup() {
        mockWebServer = MockWebServer()
        mockWebServer.start()
        
        val retrofit = Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(Json.asConverterFactory("application/json".toMediaType()))
            .build()
        
        apiService = retrofit.create(ApiService::class.java)
        repository = UserRepositoryImpl(apiService, mockDao)
    }
    
    @After
    fun teardown() {
        mockWebServer.shutdown()
    }
    
    @Test
    fun `getUsers returns success`() = runTest {
        // Arrange
        val mockResponse = MockResponse()
            .setResponseCode(200)
            .setBody("""
                [
                    {"id":"1","name":"John","email":"john@test.com"}
                ]
            """.trimIndent())
        mockWebServer.enqueue(mockResponse)
        
        // Act
        val result = repository.getUsers()
        
        // Assert
        assertThat(result.isSuccess).isTrue()
        assertThat(result.getOrNull()).hasSize(1)
    }
    
    @Test
    fun `getUsers returns error on failure`() = runTest {
        // Arrange
        val mockResponse = MockResponse()
            .setResponseCode(500)
        mockWebServer.enqueue(mockResponse)
        
        // Act
        val result = repository.getUsers()
        
        // Assert
        assertThat(result.isFailure).isTrue()
    }
}
```

## Integration Testing

### End-to-End Test

```kotlin
@HiltAndroidTest
class UserFlowIntegrationTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Before
    fun setup() {
        hiltRule.inject()
    }
    
    @Test
    fun completeUserFlow() {
        // Start at login
        composeTestRule
            .onNodeWithText("Login")
            .assertIsDisplayed()
        
        // Login
        composeTestRule
            .onNodeWithTag("email_input")
            .performTextInput("test@example.com")
        
        composeTestRule
            .onNodeWithTag("password_input")
            .performTextInput("password")
        
        composeTestRule
            .onNodeWithTag("login_button")
            .performClick()
        
        // Wait for navigation
        composeTestRule.waitForIdle()
        
        // Verify home screen
        composeTestRule
            .onNodeWithTag("home_screen")
            .assertIsDisplayed()
        
        // Navigate to profile
        composeTestRule
            .onNodeWithContentDescription("Profile")
            .performClick()
        
        // Verify profile screen
        composeTestRule
            .onNodeWithTag("profile_screen")
            .assertIsDisplayed()
    }
}
```

## Test Helpers

### Compose Test Extensions

```kotlin
fun ComposeContentTestRule.waitForText(
    text: String,
    timeoutMillis: Long = 5000
) {
    waitUntil(timeoutMillis) {
        onAllNodesWithText(text)
            .fetchSemanticsNodes()
            .isNotEmpty()
    }
}

fun ComposeContentTestRule.assertTextExists(text: String) {
    onNodeWithText(text).assertExists()
}

fun ComposeContentTestRule.assertTextDoesNotExist(text: String) {
    onNodeWithText(text).assertDoesNotExist()
}
```

### Test Rules

```kotlin
class TestCoroutineRule(
    val testDispatcher: TestDispatcher = StandardTestDispatcher()
) : TestWatcher() {
    
    override fun starting(description: Description) {
        Dispatchers.setMain(testDispatcher)
    }
    
    override fun finished(description: Description) {
        Dispatchers.resetMain()
    }
}

// Usage
@get:Rule
val coroutineRule = TestCoroutineRule()

@Test
fun test() = runTest(coroutineRule.testDispatcher) {
    // Test code
}
```

## Best Practices

1. ✅ Use test data builders for complex objects
2. ✅ Implement Robot pattern for UI tests
3. ✅ Create fake repositories for testing
4. ✅ Test ViewModels independently from UI
5. ✅ Use Turbine for Flow testing
6. ✅ Mock external dependencies
7. ✅ Write integration tests for critical flows
8. ✅ Use test tags for finding composables
9. ✅ Test loading, success, and error states
10. ✅ Keep tests focused and independent
11. ✅ Use descriptive test names
12. ✅ Follow AAA pattern (Arrange, Act, Assert)
13. ✅ Test edge cases
14. ✅ Aim for high coverage on business logic
15. ✅ Use MockWebServer for API tests

## Test Organization

### Naming Convention

```kotlin
class UserViewModelTest {
    
    @Test
    fun `loadUsers updates state to success when repository returns data`() {
        // Clear test name describes expected behavior
    }
    
    @Test
    fun `loadUsers updates state to error when repository throws exception`() {
        // Describes failure scenario
    }
}
```

### Nested Tests

```kotlin
import org.junit.jupiter.api.Nested

class UserFeatureTest {
    
    @Nested
    inner class LoadingTests {
        @Test
        fun `shows loading indicator`() { }
        
        @Test
        fun `hides loading when data arrives`() { }
    }
    
    @Nested
    inner class ErrorTests {
        @Test
        fun `shows error message`() { }
        
        @Test
        fun `retry button reloads data`() { }
    }
}
```

## Resources

- [Testing in Android](https://developer.android.com/training/testing)
- [Compose Testing](https://developer.android.com/jetpack/compose/testing)
- [Testing Coroutines](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-test/)
- [Turbine](https://github.com/cashapp/turbine)
- [MockWebServer](https://github.com/square/okhttp/tree/master/mockwebserver)

