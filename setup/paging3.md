# Paging3 Setup

## Overview
Paging 3 library helps you load and display pages of data from a larger dataset, with built-in support for loading states, error handling, and efficient memory usage.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
paging = "3.3.4"

[libraries]
androidx-paging-runtime = { group = "androidx.paging", name = "paging-runtime", version.ref = "paging" }
androidx-paging-compose = { group = "androidx.paging", name = "paging-compose", version.ref = "paging" }
androidx-paging-testing = { group = "androidx.paging", name = "paging-testing", version.ref = "paging" }

[bundles]
paging = ["androidx-paging-runtime", "androidx-paging-compose"]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.paging)
    
    // For testing
    testImplementation(libs.androidx.paging.testing)
}
```

## Basic Implementation

### 1. Define Data Source

```kotlin
import androidx.paging.PagingSource
import androidx.paging.PagingState
import retrofit2.HttpException
import java.io.IOException

class UserPagingSource(
    private val apiService: ApiService
) : PagingSource<Int, User>() {
    
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        return try {
            val page = params.key ?: 1
            val response = apiService.getUsers(
                page = page,
                pageSize = params.loadSize
            )
            
            LoadResult.Page(
                data = response.users,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (response.users.isEmpty()) null else page + 1
            )
        } catch (e: IOException) {
            LoadResult.Error(e)
        } catch (e: HttpException) {
            LoadResult.Error(e)
        }
    }
    
    override fun getRefreshKey(state: PagingState<Int, User>): Int? {
        return state.anchorPosition?.let { anchorPosition ->
            state.closestPageToPosition(anchorPosition)?.prevKey?.plus(1)
                ?: state.closestPageToPosition(anchorPosition)?.nextKey?.minus(1)
        }
    }
}
```

### 2. Create Repository

```kotlin
import androidx.paging.Pager
import androidx.paging.PagingConfig
import androidx.paging.PagingData
import kotlinx.coroutines.flow.Flow

class UserRepository(
    private val apiService: ApiService
) {
    fun getUsers(): Flow<PagingData<User>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                enablePlaceholders = false,
                prefetchDistance = 3,
                initialLoadSize = 20
            ),
            pagingSourceFactory = { UserPagingSource(apiService) }
        ).flow
    }
}
```

### 3. Setup ViewModel

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.paging.PagingData
import androidx.paging.cachedIn
import kotlinx.coroutines.flow.Flow

class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    val users: Flow<PagingData<User>> = repository.getUsers()
        .cachedIn(viewModelScope) // Cache for config changes
}
```

### 4. Display in Compose

```kotlin
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.runtime.Composable
import androidx.paging.compose.LazyPagingItems
import androidx.paging.compose.collectAsLazyPagingItems
import androidx.paging.compose.itemKey

@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users: LazyPagingItems<User> = viewModel.users.collectAsLazyPagingItems()
    
    LazyColumn {
        items(
            count = users.itemCount,
            key = users.itemKey { it.id }
        ) { index ->
            val user = users[index]
            user?.let {
                UserItem(user = it)
            }
        }
    }
}
```

## Handling Loading States

### In Compose

```kotlin
import androidx.paging.LoadState
import androidx.compose.foundation.layout.*

@Composable
fun UserListScreen(viewModel: UserViewModel = hiltViewModel()) {
    val users = viewModel.users.collectAsLazyPagingItems()
    
    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn {
            items(
                count = users.itemCount,
                key = users.itemKey { it.id }
            ) { index ->
                users[index]?.let { user ->
                    UserItem(user)
                }
            }
            
            // Loading indicator at bottom
            when {
                users.loadState.append is LoadState.Loading -> {
                    item {
                        CircularProgressIndicator(
                            modifier = Modifier
                                .fillMaxWidth()
                                .padding(16.dp)
                                .wrapContentWidth(Alignment.CenterHorizontally)
                        )
                    }
                }
                
                users.loadState.append is LoadState.Error -> {
                    item {
                        ErrorItem(
                            message = "Failed to load more items",
                            onRetry = { users.retry() }
                        )
                    }
                }
            }
        }
        
        // Initial loading
        if (users.loadState.refresh is LoadState.Loading) {
            CircularProgressIndicator(
                modifier = Modifier.align(Alignment.Center)
            )
        }
        
        // Initial error
        if (users.loadState.refresh is LoadState.Error) {
            ErrorScreen(
                message = "Failed to load users",
                onRetry = { users.refresh() },
                modifier = Modifier.align(Alignment.Center)
            )
        }
    }
}
```

### Detailed Load States

```kotlin
@Composable
fun UserListWithStates(users: LazyPagingItems<User>) {
    val loadState = users.loadState
    
    when {
        // Initial loading
        loadState.refresh is LoadState.Loading && users.itemCount == 0 -> {
            LoadingScreen()
        }
        
        // Initial error
        loadState.refresh is LoadState.Error && users.itemCount == 0 -> {
            val error = (loadState.refresh as LoadState.Error).error
            ErrorScreen(error.message, onRetry = { users.retry() })
        }
        
        // Empty state
        loadState.refresh is LoadState.NotLoading && users.itemCount == 0 -> {
            EmptyScreen()
        }
        
        // Success with data
        else -> {
            LazyColumn {
                items(
                    count = users.itemCount,
                    key = users.itemKey { it.id }
                ) { index ->
                    users[index]?.let { UserItem(it) }
                }
                
                // Append loading
                if (loadState.append is LoadState.Loading) {
                    item { LoadingItem() }
                }
                
                // Append error
                if (loadState.append is LoadState.Error) {
                    val error = (loadState.append as LoadState.Error).error
                    item {
                        ErrorItem(
                            message = error.message ?: "Error",
                            onRetry = { users.retry() }
                        )
                    }
                }
            }
        }
    }
}
```

## RemoteMediator (Network + Database)

### Setup Room Database

```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val email: String
)

@Entity(tableName = "remote_keys")
data class RemoteKey(
    @PrimaryKey val userId: Long,
    val prevKey: Int?,
    val nextKey: Int?
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): PagingSource<Int, UserEntity>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUsers(users: List<UserEntity>)
    
    @Query("DELETE FROM users")
    suspend fun clearUsers()
}

@Dao
interface RemoteKeyDao {
    @Query("SELECT * FROM remote_keys WHERE userId = :userId")
    suspend fun getRemoteKey(userId: Long): RemoteKey?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertKeys(keys: List<RemoteKey>)
    
    @Query("DELETE FROM remote_keys")
    suspend fun clearKeys()
}
```

### Create RemoteMediator

```kotlin
import androidx.paging.ExperimentalPagingApi
import androidx.paging.LoadType
import androidx.paging.PagingState
import androidx.paging.RemoteMediator
import androidx.room.withTransaction
import retrofit2.HttpException
import java.io.IOException

@OptIn(ExperimentalPagingApi::class)
class UserRemoteMediator(
    private val database: AppDatabase,
    private val apiService: ApiService
) : RemoteMediator<Int, UserEntity>() {
    
    private val userDao = database.userDao()
    private val remoteKeyDao = database.remoteKeyDao()
    
    override suspend fun load(
        loadType: LoadType,
        state: PagingState<Int, UserEntity>
    ): MediatorResult {
        return try {
            val page = when (loadType) {
                LoadType.REFRESH -> {
                    val remoteKey = getRemoteKeyClosestToCurrentPosition(state)
                    remoteKey?.nextKey?.minus(1) ?: 1
                }
                LoadType.PREPEND -> {
                    val remoteKey = getRemoteKeyForFirstItem(state)
                    remoteKey?.prevKey
                        ?: return MediatorResult.Success(endOfPaginationReached = true)
                }
                LoadType.APPEND -> {
                    val remoteKey = getRemoteKeyForLastItem(state)
                    remoteKey?.nextKey
                        ?: return MediatorResult.Success(endOfPaginationReached = true)
                }
            }
            
            val response = apiService.getUsers(
                page = page,
                pageSize = state.config.pageSize
            )
            
            val users = response.users
            val endOfPaginationReached = users.isEmpty()
            
            database.withTransaction {
                if (loadType == LoadType.REFRESH) {
                    remoteKeyDao.clearKeys()
                    userDao.clearUsers()
                }
                
                val prevKey = if (page == 1) null else page - 1
                val nextKey = if (endOfPaginationReached) null else page + 1
                val keys = users.map {
                    RemoteKey(userId = it.id, prevKey = prevKey, nextKey = nextKey)
                }
                
                remoteKeyDao.insertKeys(keys)
                userDao.insertUsers(users.map { it.toEntity() })
            }
            
            MediatorResult.Success(endOfPaginationReached = endOfPaginationReached)
        } catch (e: IOException) {
            MediatorResult.Error(e)
        } catch (e: HttpException) {
            MediatorResult.Error(e)
        }
    }
    
    private suspend fun getRemoteKeyForLastItem(
        state: PagingState<Int, UserEntity>
    ): RemoteKey? {
        return state.pages.lastOrNull { it.data.isNotEmpty() }?.data?.lastOrNull()
            ?.let { user -> remoteKeyDao.getRemoteKey(user.id) }
    }
    
    private suspend fun getRemoteKeyForFirstItem(
        state: PagingState<Int, UserEntity>
    ): RemoteKey? {
        return state.pages.firstOrNull { it.data.isNotEmpty() }?.data?.firstOrNull()
            ?.let { user -> remoteKeyDao.getRemoteKey(user.id) }
    }
    
    private suspend fun getRemoteKeyClosestToCurrentPosition(
        state: PagingState<Int, UserEntity>
    ): RemoteKey? {
        return state.anchorPosition?.let { position ->
            state.closestItemToPosition(position)?.id?.let { userId ->
                remoteKeyDao.getRemoteKey(userId)
            }
        }
    }
}
```

### Use RemoteMediator

```kotlin
@OptIn(ExperimentalPagingApi::class)
class UserRepository(
    private val database: AppDatabase,
    private val apiService: ApiService
) {
    fun getUsers(): Flow<PagingData<UserEntity>> {
        return Pager(
            config = PagingConfig(
                pageSize = 20,
                enablePlaceholders = false
            ),
            remoteMediator = UserRemoteMediator(database, apiService),
            pagingSourceFactory = { database.userDao().getAllUsers() }
        ).flow
    }
}
```

## Transformations

### Map Items

```kotlin
import androidx.paging.map

val uiUsers: Flow<PagingData<UiUser>> = repository.getUsers()
    .map { pagingData ->
        pagingData.map { user ->
            user.toUiModel()
        }
    }
    .cachedIn(viewModelScope)
```

### Filter Items

```kotlin
import androidx.paging.filter

val activeUsers: Flow<PagingData<User>> = repository.getUsers()
    .map { pagingData ->
        pagingData.filter { user ->
            user.isActive
        }
    }
    .cachedIn(viewModelScope)
```

### Insert Separators

```kotlin
import androidx.paging.insertSeparators

sealed class UiModel {
    data class UserItem(val user: User) : UiModel()
    data class SeparatorItem(val letter: Char) : UiModel()
}

val usersWithSeparators: Flow<PagingData<UiModel>> = repository.getUsers()
    .map { pagingData ->
        pagingData.map { UiModel.UserItem(it) }
            .insertSeparators { before, after ->
                if (before == null) {
                    // First item
                    return@insertSeparators UiModel.SeparatorItem(
                        after?.user?.name?.first() ?: 'A'
                    )
                }
                
                if (after == null) {
                    // Last item
                    return@insertSeparators null
                }
                
                val beforeLetter = before.user.name.first()
                val afterLetter = after.user.name.first()
                
                if (beforeLetter != afterLetter) {
                    UiModel.SeparatorItem(afterLetter)
                } else {
                    null
                }
            }
    }
    .cachedIn(viewModelScope)
```

## Pull to Refresh

```kotlin
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.pulltorefresh.PullToRefreshBox

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun UserListWithRefresh(viewModel: UserViewModel = hiltViewModel()) {
    val users = viewModel.users.collectAsLazyPagingItems()
    val isRefreshing = users.loadState.refresh is LoadState.Loading
    
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = { users.refresh() }
    ) {
        LazyColumn {
            items(
                count = users.itemCount,
                key = users.itemKey { it.id }
            ) { index ->
                users[index]?.let { UserItem(it) }
            }
        }
    }
}
```

## Search with Paging

```kotlin
class SearchViewModel(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery.asStateFlow()
    
    val searchResults: Flow<PagingData<User>> = searchQuery
        .debounce(300)
        .flatMapLatest { query ->
            if (query.isEmpty()) {
                flowOf(PagingData.empty())
            } else {
                repository.searchUsers(query)
            }
        }
        .cachedIn(viewModelScope)
    
    fun updateQuery(query: String) {
        _searchQuery.value = query
    }
}
```

## Testing

### Test PagingSource

```kotlin
import androidx.paging.PagingSource
import kotlinx.coroutines.test.runTest
import org.junit.Test

class UserPagingSourceTest {
    
    @Test
    fun `load returns page when successful`() = runTest {
        val mockApi = mockk<ApiService>()
        coEvery { mockApi.getUsers(any(), any()) } returns UsersResponse(
            users = listOf(testUser1, testUser2)
        )
        
        val pagingSource = UserPagingSource(mockApi)
        
        val result = pagingSource.load(
            PagingSource.LoadParams.Refresh(
                key = null,
                loadSize = 20,
                placeholdersEnabled = false
            )
        )
        
        assertTrue(result is PagingSource.LoadResult.Page)
        val page = result as PagingSource.LoadResult.Page
        assertEquals(2, page.data.size)
        assertEquals(null, page.prevKey)
        assertEquals(2, page.nextKey)
    }
    
    @Test
    fun `load returns error on exception`() = runTest {
        val mockApi = mockk<ApiService>()
        coEvery { mockApi.getUsers(any(), any()) } throws IOException()
        
        val pagingSource = UserPagingSource(mockApi)
        
        val result = pagingSource.load(
            PagingSource.LoadParams.Refresh(
                key = null,
                loadSize = 20,
                placeholdersEnabled = false
            )
        )
        
        assertTrue(result is PagingSource.LoadResult.Error)
    }
}
```

## Best Practices

1. ✅ Use `cachedIn(viewModelScope)` to cache data across config changes
2. ✅ Handle all load states (refresh, append, prepend)
3. ✅ Provide retry functionality for errors
4. ✅ Use RemoteMediator for offline support
5. ✅ Set appropriate page size (20-50 items)
6. ✅ Use unique keys for items
7. ✅ Show loading indicators for better UX
8. ✅ Handle empty states
9. ✅ Test PagingSource thoroughly
10. ✅ Consider using placeholders for consistent item positions

## Common Issues

### Items Jumping

Use stable keys:
```kotlin
items(
    count = users.itemCount,
    key = users.itemKey { it.id } // Stable key
) { ... }
```

### Duplicate Items

Check your `getRefreshKey` implementation and ensure proper page key calculation.

### Memory Leaks

Always use `cachedIn(viewModelScope)` to tie the cache to ViewModel lifecycle.

## Resources

- [Official Documentation](https://developer.android.com/topic/libraries/architecture/paging/v3-overview)
- [Paging Codelab](https://developer.android.com/codelabs/android-paging)
- [Paging Samples](https://github.com/android/architecture-components-samples/tree/main/PagingSample)

