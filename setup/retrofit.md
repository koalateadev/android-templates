# Retrofit Setup

## Overview
Retrofit is a type-safe HTTP client for Android and Java that turns your HTTP API into a Java interface.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
retrofit = "2.11.0"
kotlinx-serialization = "1.7.3"

[libraries]
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-kotlinx-serialization = { group = "com.squareup.retrofit2", name = "converter-kotlinx-serialization", version.ref = "retrofit" }
retrofit-converter-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }

[bundles]
retrofit = ["retrofit", "retrofit-converter-kotlinx-serialization"]
```

### 2. Add Dependencies and Plugin

In app `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(libs.bundles.retrofit)
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.okhttp) // From OkHttp setup
    implementation(libs.okhttp.logging.interceptor)
}
```

### 3. Define Data Models

Using Kotlinx Serialization:

```kotlin
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val id: Long,
    val name: String,
    val email: String,
    @SerialName("created_at")
    val createdAt: String
)

@Serializable
data class ApiResponse<T>(
    val success: Boolean,
    val data: T?,
    val message: String?
)

@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String
)
```

### 4. Create API Interface

```kotlin
import retrofit2.Response
import retrofit2.http.*

interface ApiService {
    
    @GET("users")
    suspend fun getUsers(): Response<List<User>>
    
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") userId: Long): Response<User>
    
    @POST("users")
    suspend fun createUser(@Body request: CreateUserRequest): Response<User>
    
    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") userId: Long,
        @Body request: CreateUserRequest
    ): Response<User>
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") userId: Long): Response<Unit>
    
    @GET("search")
    suspend fun searchUsers(
        @Query("q") query: String,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 20
    ): Response<List<User>>
    
    @GET("users/{id}")
    suspend fun getUserWithHeaders(
        @Path("id") userId: Long,
        @Header("Authorization") token: String
    ): Response<User>
    
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part("description") description: RequestBody,
        @Part file: MultipartBody.Part
    ): Response<Unit>
}
```

### 5. Create Retrofit Instance

```kotlin
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import retrofit2.Retrofit

object RetrofitClient {
    private const val BASE_URL = "https://api.example.com/"
    
    private val json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
        encodeDefaults = true
    }
    
    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        })
        .build()
    
    private val retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
    
    val apiService: ApiService = retrofit.create(ApiService::class.java)
}
```

### 6. With Hilt Dependency Injection

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        coerceInputValues = true
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        json: Json
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(
                json.asConverterFactory("application/json".toMediaType())
            )
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

### 7. Create Repository

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class UserRepository(private val apiService: ApiService) {
    
    suspend fun getUsers(): Result<List<User>> {
        return try {
            val response = apiService.getUsers()
            if (response.isSuccessful && response.body() != null) {
                Result.success(response.body()!!)
            } else {
                Result.failure(Exception("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    fun getUsersFlow(): Flow<Result<List<User>>> = flow {
        emit(getUsers())
    }
    
    suspend fun getUserById(id: Long): Result<User> {
        return try {
            val response = apiService.getUserById(id)
            if (response.isSuccessful && response.body() != null) {
                Result.success(response.body()!!)
            } else {
                Result.failure(Exception("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    suspend fun createUser(name: String, email: String): Result<User> {
        return try {
            val request = CreateUserRequest(name, email)
            val response = apiService.createUser(request)
            if (response.isSuccessful && response.body() != null) {
                Result.success(response.body()!!)
            } else {
                Result.failure(Exception("Error: ${response.code()}"))
            }
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
}
```

### 8. Usage in ViewModel

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState
    
    init {
        loadUsers()
    }
    
    fun loadUsers() {
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
    
    fun createUser(name: String, email: String) {
        viewModelScope.launch {
            repository.createUser(name, email)
                .onSuccess {
                    loadUsers() // Reload list
                }
                .onFailure { error ->
                    _uiState.value = UiState.Error(error.message ?: "Failed to create user")
                }
        }
    }
}

sealed interface UiState {
    data object Loading : UiState
    data class Success(val users: List<User>) : UiState
    data class Error(val message: String) : UiState
}
```

## Advanced Features

### Generic API Response Handler

```kotlin
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val code: Int, val message: String?) : ApiResult<Nothing>()
    data object Loading : ApiResult<Nothing>()
}

suspend fun <T> safeApiCall(apiCall: suspend () -> Response<T>): ApiResult<T> {
    return try {
        val response = apiCall()
        if (response.isSuccessful) {
            ApiResult.Success(response.body()!!)
        } else {
            ApiResult.Error(response.code(), response.message())
        }
    } catch (e: Exception) {
        ApiResult.Error(-1, e.message)
    }
}

// Usage
val result = safeApiCall { apiService.getUsers() }
```

### Custom Interceptors

```kotlin
class AuthInterceptor(private val tokenProvider: TokenProvider) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val token = tokenProvider.getToken()
        
        val newRequest = originalRequest.newBuilder()
            .header("Authorization", "Bearer $token")
            .build()
        
        val response = chain.proceed(newRequest)
        
        // Handle 401 unauthorized
        if (response.code == 401) {
            response.close()
            // Refresh token logic here
            val refreshedToken = tokenProvider.refreshToken()
            val retryRequest = originalRequest.newBuilder()
                .header("Authorization", "Bearer $refreshedToken")
                .build()
            return chain.proceed(retryRequest)
        }
        
        return response
    }
}
```

### File Upload

```kotlin
suspend fun uploadFile(file: File, description: String): Result<Unit> {
    return try {
        val requestBody = file.asRequestBody("image/*".toMediaTypeOrNull())
        val filePart = MultipartBody.Part.createFormData(
            "file",
            file.name,
            requestBody
        )
        val descriptionBody = description.toRequestBody("text/plain".toMediaType())
        
        val response = apiService.uploadFile(descriptionBody, filePart)
        if (response.isSuccessful) {
            Result.success(Unit)
        } else {
            Result.failure(Exception("Upload failed"))
        }
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### Streaming Response

```kotlin
@Streaming
@GET("download")
suspend fun downloadFile(@Url fileUrl: String): Response<ResponseBody>

// Usage
suspend fun downloadFile(url: String, destinationFile: File) {
    val response = apiService.downloadFile(url)
    if (response.isSuccessful) {
        response.body()?.let { body ->
            destinationFile.outputStream().use { output ->
                body.byteStream().use { input ->
                    input.copyTo(output)
                }
            }
        }
    }
}
```

## Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class ApiServiceTest {
    private lateinit var mockWebServer: MockWebServer
    private lateinit var apiService: ApiService
    
    @Before
    fun setup() {
        mockWebServer = MockWebServer()
        mockWebServer.start()
        
        val retrofit = Retrofit.Builder()
            .baseUrl(mockWebServer.url("/"))
            .addConverterFactory(
                Json.asConverterFactory("application/json".toMediaType())
            )
            .build()
        
        apiService = retrofit.create(ApiService::class.java)
    }
    
    @After
    fun teardown() {
        mockWebServer.shutdown()
    }
    
    @Test
    fun testGetUsers() = runTest {
        // Arrange
        val mockResponse = MockResponse()
            .setResponseCode(200)
            .setBody("""[{"id":1,"name":"John","email":"john@example.com","created_at":"2025-01-01"}]""")
        mockWebServer.enqueue(mockResponse)
        
        // Act
        val response = apiService.getUsers()
        
        // Assert
        assertThat(response.isSuccessful).isTrue()
        assertThat(response.body()).hasSize(1)
        assertThat(response.body()?.first()?.name).isEqualTo("John")
    }
}
```

## Best Practices

- Use suspend functions with Coroutines
- Handle errors with sealed classes
- Use Repository pattern for data access
- Implement interceptors for cross-cutting concerns
- Set appropriate timeouts
- Cache responses when appropriate
- Keep API interface focused

## Resources

- [Official Documentation](https://square.github.io/retrofit/)
- [GitHub Repository](https://github.com/square/retrofit)
- [Kotlinx Serialization](https://github.com/Kotlin/kotlinx.serialization)

