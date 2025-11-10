# Apollo GraphQL Setup

## Overview
Apollo Android is a GraphQL client that generates Java/Kotlin models from GraphQL queries at compile time.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
apollo = "4.1.0"

[libraries]
apollo-runtime = { group = "com.apollographql.apollo", name = "apollo-runtime", version.ref = "apollo" }
apollo-normalized-cache = { group = "com.apollographql.apollo", name = "apollo-normalized-cache", version.ref = "apollo" }
apollo-adapters = { group = "com.apollographql.apollo", name = "apollo-adapters", version.ref = "apollo" }
apollo-mockserver = { group = "com.apollographql.apollo", name = "apollo-mockserver", version.ref = "apollo" }

[plugins]
apollo = { id = "com.apollographql.apollo", version.ref = "apollo" }
```

### 2. Add Plugin and Dependencies

In `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.apollo)
}

dependencies {
    implementation(libs.apollo.runtime)
    
    // Optional: Normalized cache
    implementation(libs.apollo.normalized.cache)
    
    // Optional: Date/UUID adapters
    implementation(libs.apollo.adapters)
    
    // Testing
    testImplementation(libs.apollo.mockserver)
}

apollo {
    service("service") {
        packageName.set("com.example.app")
        
        // GraphQL schema location
        schemaFile.set(file("src/main/graphql/schema.graphqls"))
        
        // Or download from server
        // introspection {
        //     endpointUrl.set("https://api.example.com/graphql")
        //     schemaFile.set(file("src/main/graphql/schema.graphqls"))
        // }
    }
}
```

## Basic Setup

### 1. Add GraphQL Schema

Create `src/main/graphql/schema.graphqls`:

```graphql
type Query {
  users: [User!]!
  user(id: ID!): User
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

input CreateUserInput {
  name: String!
  email: String!
}

input UpdateUserInput {
  name: String
  email: String
}
```

### 2. Write Queries

Create `src/main/graphql/GetUsers.graphql`:

```graphql
query GetUsers {
  users {
    id
    name
    email
  }
}
```

Create `src/main/graphql/GetUser.graphql`:

```graphql
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    posts {
      id
      title
      content
    }
  }
}
```

### 3. Write Mutations

Create `src/main/graphql/CreateUser.graphql`:

```graphql
mutation CreateUser($input: CreateUserInput!) {
  createUser(input: $input) {
    id
    name
    email
  }
}
```

### 4. Create Apollo Client

```kotlin
import com.apollographql.apollo.ApolloClient
import com.apollographql.apollo.network.okHttpClient
import okhttp3.OkHttpClient

val apolloClient = ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .okHttpClient(okHttpClient)
    .build()
```

### 5. Execute Queries

```kotlin
import com.apollographql.apollo.exception.ApolloException

class UserRepository(private val apolloClient: ApolloClient) {
    
    suspend fun getUsers(): Result<List<GetUsersQuery.User>> {
        return try {
            val response = apolloClient.query(GetUsersQuery()).execute()
            
            if (response.hasErrors()) {
                Result.failure(Exception(response.errors?.firstOrNull()?.message))
            } else {
                Result.success(response.data?.users ?: emptyList())
            }
        } catch (e: ApolloException) {
            Result.failure(e)
        }
    }
    
    suspend fun getUser(id: String): Result<GetUserQuery.User?> {
        return try {
            val response = apolloClient.query(GetUserQuery(id)).execute()
            
            if (response.hasErrors()) {
                Result.failure(Exception(response.errors?.firstOrNull()?.message))
            } else {
                Result.success(response.data?.user)
            }
        } catch (e: ApolloException) {
            Result.failure(e)
        }
    }
}
```

### 6. Execute Mutations

```kotlin
suspend fun createUser(name: String, email: String): Result<CreateUserMutation.CreateUser> {
    return try {
        val input = CreateUserInput(name, email)
        val response = apolloClient.mutation(CreateUserMutation(input)).execute()
        
        if (response.hasErrors()) {
            Result.failure(Exception(response.errors?.firstOrNull()?.message))
        } else {
            Result.success(response.data?.createUser!!)
        }
    } catch (e: ApolloException) {
        Result.failure(e)
    }
}
```

## With Hilt

### Setup Module

```kotlin
import com.apollographql.apollo.ApolloClient
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object ApolloModule {
    
    @Provides
    @Singleton
    fun provideApolloClient(
        okHttpClient: OkHttpClient
    ): ApolloClient {
        return ApolloClient.Builder()
            .serverUrl("https://api.example.com/graphql")
            .okHttpClient(okHttpClient)
            .build()
    }
}
```

## Authentication

### Add Auth Header

```kotlin
import com.apollographql.apollo.api.http.HttpRequest
import com.apollographql.apollo.api.http.HttpResponse
import com.apollographql.apollo.network.http.HttpInterceptor
import com.apollographql.apollo.network.http.HttpInterceptorChain

class AuthorizationInterceptor(
    private val tokenProvider: TokenProvider
) : HttpInterceptor {
    
    override suspend fun intercept(
        request: HttpRequest,
        chain: HttpInterceptorChain
    ): HttpResponse {
        val token = tokenProvider.getToken()
        
        val newRequest = request.newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()
        
        return chain.proceed(newRequest)
    }
}

// Add to Apollo Client
val apolloClient = ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .addHttpInterceptor(AuthorizationInterceptor(tokenProvider))
    .build()
```

## Caching

### Setup Normalized Cache

```kotlin
import com.apollographql.apollo.cache.normalized.api.MemoryCacheFactory
import com.apollographql.apollo.cache.normalized.normalizedCache

val cacheFactory = MemoryCacheFactory(maxSizeBytes = 10 * 1024 * 1024)

val apolloClient = ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .normalizedCache(cacheFactory)
    .build()
```

### Cache Policies

```kotlin
import com.apollographql.apollo.api.CachePolicy

// Cache first, then network
val response = apolloClient
    .query(GetUsersQuery())
    .cachePolicy(CachePolicy.CacheFirst)
    .execute()

// Network only
val response = apolloClient
    .query(GetUsersQuery())
    .cachePolicy(CachePolicy.NetworkOnly)
    .execute()

// Cache only
val response = apolloClient
    .query(GetUsersQuery())
    .cachePolicy(CachePolicy.CacheOnly)
    .execute()
```

### Watch Queries (Observable)

```kotlin
import com.apollographql.apollo.api.ApolloResponse
import kotlinx.coroutines.flow.Flow

fun watchUsers(): Flow<ApolloResponse<GetUsersQuery.Data>> {
    return apolloClient
        .query(GetUsersQuery())
        .cachePolicy(CachePolicy.CacheAndNetwork)
        .watch()
}

// In ViewModel
val users: StateFlow<List<User>> = repository.watchUsers()
    .map { response ->
        response.data?.users ?: emptyList()
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
```

## Subscriptions

### Define Subscription

Create `src/main/graphql/UserAdded.graphql`:

```graphql
subscription UserAdded {
  userAdded {
    id
    name
    email
  }
}
```

### Subscribe

```kotlin
import com.apollographql.apollo.api.ApolloResponse
import kotlinx.coroutines.flow.Flow

fun subscribeToUserAdded(): Flow<ApolloResponse<UserAddedSubscription.Data>> {
    return apolloClient
        .subscription(UserAddedSubscription())
        .toFlow()
}

// Collect in ViewModel
viewModelScope.launch {
    repository.subscribeToUserAdded()
        .collect { response ->
            response.data?.userAdded?.let { user ->
                // Handle new user
            }
        }
}
```

## File Upload

### Configure

```kotlin
apollo {
    service("service") {
        packageName.set("com.example.app")
        generateKotlinModels.set(true)
        
        // Enable file upload
        generateFragmentImplementations.set(true)
    }
}
```

### Upload Mutation

```graphql
mutation UploadFile($file: Upload!) {
  uploadFile(file: $file) {
    id
    url
  }
}
```

### Execute Upload

```kotlin
import com.apollographql.apollo.api.Upload

suspend fun uploadFile(file: File): Result<String> {
    return try {
        val upload = file.asUpload("image/jpeg")
        val response = apolloClient
            .mutation(UploadFileMutation(upload))
            .execute()
        
        if (response.hasErrors()) {
            Result.failure(Exception(response.errors?.firstOrNull()?.message))
        } else {
            Result.success(response.data?.uploadFile?.url!!)
        }
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

## Fragments

### Define Fragment

Create `src/main/graphql/UserFragment.graphql`:

```graphql
fragment UserDetails on User {
  id
  name
  email
}
```

### Use Fragment

```graphql
query GetUsers {
  users {
    ...UserDetails
  }
}

query GetUser($id: ID!) {
  user(id: $id) {
    ...UserDetails
    posts {
      id
      title
    }
  }
}
```

## Custom Scalars

### Define Custom Scalar

In `build.gradle.kts`:

```kotlin
apollo {
    service("service") {
        packageName.set("com.example.app")
        
        // Map custom scalars
        mapScalar("DateTime", "java.time.Instant")
        mapScalar("URL", "java.net.URL")
    }
}
```

### Custom Adapter

```kotlin
import com.apollographql.apollo.api.Adapter
import com.apollographql.apollo.api.CustomScalarAdapters
import com.apollographql.apollo.api.json.JsonReader
import com.apollographql.apollo.api.json.JsonWriter
import java.time.Instant

object InstantAdapter : Adapter<Instant> {
    override fun fromJson(reader: JsonReader, customScalarAdapters: CustomScalarAdapters): Instant {
        return Instant.parse(reader.nextString()!!)
    }
    
    override fun toJson(writer: JsonWriter, customScalarAdapters: CustomScalarAdapters, value: Instant) {
        writer.value(value.toString())
    }
}

// Register adapter
val apolloClient = ApolloClient.Builder()
    .serverUrl("https://api.example.com/graphql")
    .addCustomScalarAdapter(DateTime.type, InstantAdapter)
    .build()
```

## Error Handling

### Handle Errors

```kotlin
suspend fun getUsers(): Result<List<User>> {
    return try {
        val response = apolloClient.query(GetUsersQuery()).execute()
        
        when {
            response.hasErrors() -> {
                val errors = response.errors?.joinToString { it.message }
                Result.failure(Exception(errors))
            }
            response.data == null -> {
                Result.failure(Exception("No data"))
            }
            else -> {
                Result.success(response.data!!.users)
            }
        }
    } catch (e: ApolloException) {
        when (e) {
            is ApolloNetworkException -> {
                Result.failure(Exception("Network error"))
            }
            is ApolloHttpException -> {
                Result.failure(Exception("HTTP ${e.statusCode}"))
            }
            else -> {
                Result.failure(e)
            }
        }
    }
}
```

## Testing

### Mock Server

```kotlin
import com.apollographql.apollo.mockserver.MockServer
import com.apollographql.apollo.mockserver.enqueueString
import kotlinx.coroutines.test.runTest
import org.junit.Test

class ApolloRepositoryTest {
    
    @Test
    fun testGetUsers() = runTest {
        val mockServer = MockServer()
        
        mockServer.enqueueString("""
            {
              "data": {
                "users": [
                  {"id": "1", "name": "John", "email": "john@test.com"}
                ]
              }
            }
        """)
        
        val apolloClient = ApolloClient.Builder()
            .serverUrl(mockServer.url())
            .build()
        
        val repository = UserRepository(apolloClient)
        val result = repository.getUsers()
        
        assertTrue(result.isSuccess)
        assertEquals(1, result.getOrNull()?.size)
        
        mockServer.stop()
    }
}
```

## Best Practices

- Use fragments for reusable fields
- Implement proper error handling
- Use normalized cache for performance
- Add authentication interceptors
- Handle network errors gracefully
- Test with MockServer
- Organize queries in separate files
- Use proper cache policies

## Resources

- [Official Documentation](https://www.apollographql.com/docs/kotlin/)
- [Apollo Kotlin GitHub](https://github.com/apollographql/apollo-kotlin)
- [Tutorial](https://www.apollographql.com/docs/kotlin/tutorial/00-introduction)

