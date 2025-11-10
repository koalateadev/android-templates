# Architecture Patterns

## Table of Contents
- [Repository Pattern](#repository-pattern)
- [Use Case Pattern](#use-case-pattern)
- [Mapper Pattern](#mapper-pattern)
- [MVVM Implementation](#mvvm-implementation)
- [Clean Architecture](#clean-architecture)
- [Multi-Module Architecture](#multi-module-architecture)

## Repository Pattern

### Basic Repository

```kotlin
interface UserRepository {
    suspend fun getUsers(): Result<List<User>>
    suspend fun getUserById(id: String): Result<User>
    suspend fun createUser(user: User): Result<User>
    suspend fun updateUser(user: User): Result<User>
    suspend fun deleteUser(id: String): Result<Unit>
}

class UserRepositoryImpl(
    private val apiService: ApiService,
    private val userDao: UserDao,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO
) : UserRepository {
    
    override suspend fun getUsers(): Result<List<User>> = withContext(ioDispatcher) {
        try {
            val response = apiService.getUsers()
            if (response.isSuccessful && response.body() != null) {
                val users = response.body()!!
                userDao.insertAll(users)
                Result.success(users)
            } else {
                Result.failure(Exception("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            // Return cached data on error
            val cachedUsers = userDao.getAllUsersOnce()
            if (cachedUsers.isNotEmpty()) {
                Result.success(cachedUsers)
            } else {
                Result.failure(e)
            }
        }
    }
    
    override suspend fun getUserById(id: String): Result<User> = withContext(ioDispatcher) {
        try {
            val response = apiService.getUserById(id)
            if (response.isSuccessful && response.body() != null) {
                val user = response.body()!!
                userDao.insert(user)
                Result.success(user)
            } else {
                Result.failure(Exception("User not found"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun createUser(user: User): Result<User> = withContext(ioDispatcher) {
        try {
            val response = apiService.createUser(user)
            if (response.isSuccessful && response.body() != null) {
                val createdUser = response.body()!!
                userDao.insert(createdUser)
                Result.success(createdUser)
            } else {
                Result.failure(Exception("Failed to create user"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun updateUser(user: User): Result<User> = withContext(ioDispatcher) {
        try {
            val response = apiService.updateUser(user.id, user)
            if (response.isSuccessful && response.body() != null) {
                val updatedUser = response.body()!!
                userDao.update(updatedUser)
                Result.success(updatedUser)
            } else {
                Result.failure(Exception("Failed to update user"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    override suspend fun deleteUser(id: String): Result<Unit> = withContext(ioDispatcher) {
        try {
            val response = apiService.deleteUser(id)
            if (response.isSuccessful) {
                userDao.deleteById(id)
                Result.success(Unit)
            } else {
                Result.failure(Exception("Failed to delete user"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

## Use Case Pattern

### Basic Use Case

```kotlin
interface UseCase<in P, out R> {
    suspend operator fun invoke(params: P): R
}

class GetUserUseCase(
    private val repository: UserRepository
) : UseCase<String, Result<User>> {
    
    override suspend fun invoke(userId: String): Result<User> {
        return repository.getUserById(userId)
    }
}

class GetUsersUseCase(
    private val repository: UserRepository,
    private val sortPreferences: SortPreferences
) : UseCase<Unit, Result<List<User>>> {
    
    override suspend fun invoke(params: Unit): Result<List<User>> {
        return repository.getUsers().map { users ->
            when (sortPreferences.sortOption) {
                SortOption.NAME_ASC -> users.sortedBy { it.name }
                SortOption.NAME_DESC -> users.sortedByDescending { it.name }
                SortOption.DATE_NEW -> users.sortedByDescending { it.createdAt }
                SortOption.DATE_OLD -> users.sortedBy { it.createdAt }
            }
        }
    }
}

// Usage in ViewModel
class UserViewModel(
    private val getUsersUseCase: GetUsersUseCase,
    private val deleteUserUseCase: DeleteUserUseCase
) : ViewModel() {
    
    fun loadUsers() {
        viewModelScope.launch {
            getUsersUseCase(Unit)
                .onSuccess { users ->
                    _uiState.update { it.copy(users = users) }
                }
                .onFailure { error ->
                    _uiState.update { it.copy(error = error.message) }
                }
        }
    }
}
```

### Flow Use Case

```kotlin
abstract class FlowUseCase<in P, out R> {
    operator fun invoke(params: P): Flow<R> = execute(params)
        .catch { e -> emit(handleError(e)) }
    
    protected abstract fun execute(params: P): Flow<R>
    
    protected open fun handleError(throwable: Throwable): R {
        throw throwable
    }
}

class ObserveUsersUseCase(
    private val repository: UserRepository
) : FlowUseCase<Unit, List<User>>() {
    
    override fun execute(params: Unit): Flow<List<User>> {
        return repository.observeUsers()
    }
}
```

## Mapper Pattern

### Entity to Domain to UI Mappers

```kotlin
// Data layer (Entity)
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?,
    val createdAt: Long
)

// Domain layer (Model)
data class User(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?,
    val createdAt: Long
)

// UI layer (UI Model)
data class UiUser(
    val id: String,
    val name: String,
    val email: String,
    val avatarUrl: String?,
    val formattedDate: String,
    val initials: String
)

// Mappers
fun UserEntity.toDomain(): User = User(
    id = id,
    name = name,
    email = email,
    avatarUrl = avatarUrl,
    createdAt = createdAt
)

fun User.toUi(): UiUser = UiUser(
    id = id,
    name = name,
    email = email,
    avatarUrl = avatarUrl,
    formattedDate = formatDate(createdAt),
    initials = name.split(" ")
        .mapNotNull { it.firstOrNull()?.uppercase() }
        .take(2)
        .joinToString("")
)

private fun formatDate(timestamp: Long): String {
    val sdf = SimpleDateFormat("MMM dd, yyyy", Locale.getDefault())
    return sdf.format(Date(timestamp))
}
```

### Bi-directional Mapping

```kotlin
interface Mapper<E, D> {
    fun mapToDomain(entity: E): D
    fun mapToEntity(domain: D): E
}

class UserMapper : Mapper<UserEntity, User> {
    override fun mapToDomain(entity: UserEntity): User {
        return User(
            id = entity.id,
            name = entity.name,
            email = entity.email,
            avatarUrl = entity.avatarUrl,
            createdAt = entity.createdAt
        )
    }
    
    override fun mapToEntity(domain: User): UserEntity {
        return UserEntity(
            id = domain.id,
            name = domain.name,
            email = domain.email,
            avatarUrl = domain.avatarUrl,
            createdAt = domain.createdAt
        )
    }
}
```

## MVVM Implementation

### Complete MVVM Example

```kotlin
// Model (Data Layer)
data class User(val id: String, val name: String, val email: String)

interface UserRepository {
    suspend fun getUsers(): Result<List<User>>
}

// ViewModel
sealed interface UiState {
    data object Loading : UiState
    data class Success(val users: List<User>) : UiState
    data class Error(val message: String) : UiState
}

sealed interface UiEvent {
    data object LoadUsers : UiEvent
    data object RefreshUsers : UiEvent
    data class DeleteUser(val userId: String) : UiEvent
}

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        onEvent(UiEvent.LoadUsers)
    }
    
    fun onEvent(event: UiEvent) {
        when (event) {
            is UiEvent.LoadUsers -> loadUsers()
            is UiEvent.RefreshUsers -> refreshUsers()
            is UiEvent.DeleteUser -> deleteUser(event.userId)
        }
    }
    
    private fun loadUsers() {
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
    
    private fun refreshUsers() {
        loadUsers()
    }
    
    private fun deleteUser(userId: String) {
        // Implementation
    }
}

// View (Composable)
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    UserContent(
        uiState = uiState,
        onEvent = viewModel::onEvent
    )
}

@Composable
fun UserContent(
    uiState: UiState,
    onEvent: (UiEvent) -> Unit
) {
    when (uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> UserList(
            users = uiState.users,
            onRefresh = { onEvent(UiEvent.RefreshUsers) },
            onDelete = { userId -> onEvent(UiEvent.DeleteUser(userId)) }
        )
        is UiState.Error -> ErrorScreen(
            message = uiState.message,
            onRetry = { onEvent(UiEvent.LoadUsers) }
        )
    }
}
```

## Clean Architecture

### Layer Structure

```kotlin
// Domain Layer
data class User(
    val id: String,
    val name: String,
    val email: String
)

interface UserRepository {
    suspend fun getUsers(): Result<List<User>>
}

class GetUsersUseCase(private val repository: UserRepository) {
    suspend operator fun invoke(): Result<List<User>> {
        return repository.getUsers()
    }
}

// Data Layer
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    suspend fun getAllUsers(): List<UserEntity>
}

@Serializable
data class UserDto(
    val id: String,
    val name: String,
    val email: String
)

interface ApiService {
    @GET("users")
    suspend fun getUsers(): Response<List<UserDto>>
}

class UserRepositoryImpl(
    private val apiService: ApiService,
    private val userDao: UserDao
) : UserRepository {
    
    override suspend fun getUsers(): Result<List<User>> {
        return try {
            val response = apiService.getUsers()
            if (response.isSuccessful) {
                val users = response.body()!!.map { it.toDomain() }
                userDao.insertAll(users.map { it.toEntity() })
                Result.success(users)
            } else {
                Result.failure(Exception("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}

// Mappers
fun UserDto.toDomain() = User(id, name, email)
fun User.toEntity() = UserEntity(id, name, email)
fun UserEntity.toDomain() = User(id, name, email)

// Presentation Layer
data class UiUser(
    val id: String,
    val displayName: String,
    val emailFormatted: String
)

fun User.toUi() = UiUser(
    id = id,
    displayName = name,
    emailFormatted = email.lowercase()
)

class UserViewModel(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            getUsersUseCase()
                .onSuccess { users ->
                    _uiState.value = UiState.Success(users.map { it.toUi() })
                }
                .onFailure { error ->
                    _uiState.value = UiState.Error(error.message ?: "Error")
                }
        }
    }
}
```

## Multi-Module Architecture

### Module Structure

```
:app - Main application module
:feature:home - Home feature
:feature:profile - Profile feature
:core:ui - Shared UI components
:core:data - Data layer
:core:domain - Domain models and use cases
:core:network - Network client
:core:database - Database
```

### Feature Module

```kotlin
// :feature:home module

// Public API
interface HomeEntry {
    @Composable
    fun HomeScreen(onNavigateToProfile: (String) -> Unit)
}

// Internal implementation
internal class HomeEntryImpl(
    private val viewModelFactory: ViewModelProvider.Factory
) : HomeEntry {
    
    @Composable
    override fun HomeScreen(onNavigateToProfile: (String) -> Unit) {
        val viewModel: HomeViewModel = viewModel(factory = viewModelFactory)
        HomeScreenContent(viewModel, onNavigateToProfile)
    }
}

// Dependency Injection
@Module
@InstallIn(SingletonComponent::class)
object HomeModule {
    
    @Provides
    fun provideHomeEntry(
        factory: ViewModelProvider.Factory
    ): HomeEntry {
        return HomeEntryImpl(factory)
    }
}
```

## Best Practices

- Separate concerns (UI, Domain, Data)
- Depend on abstractions not implementations
- Use interfaces for repositories
- Keep ViewModels framework-independent
- Use mappers between layers
- Single responsibility per class
- Use dependency injection for testability
- Keep domain layer pure Kotlin
- Use use cases for complex business logic

## Resources

- [Guide to App Architecture](https://developer.android.com/topic/architecture)
- [Architecture Samples](https://github.com/android/architecture-samples)
- [Now in Android](https://github.com/android/nowinandroid)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

