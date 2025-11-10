# Moshi Setup

## Overview
Moshi is a modern JSON library for Android and Java that makes it easy to parse JSON into Java/Kotlin objects. It's created by Square and is a lighter alternative to Gson with better Kotlin support.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
moshi = "1.15.1"

[libraries]
moshi = { group = "com.squareup.moshi", name = "moshi", version.ref = "moshi" }
moshi-kotlin = { group = "com.squareup.moshi", name = "moshi-kotlin", version.ref = "moshi" }
moshi-adapters = { group = "com.squareup.moshi", name = "moshi-adapters", version.ref = "moshi" }
moshi-kotlin-codegen = { group = "com.squareup.moshi", name = "moshi-kotlin-codegen", version.ref = "moshi" }
retrofit-converter-moshi = { group = "com.squareup.retrofit2", name = "converter-moshi", version.ref = "retrofit" }

[bundles]
moshi = ["moshi", "moshi-kotlin", "moshi-adapters"]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.moshi)
    
    // For code generation (recommended)
    ksp(libs.moshi.kotlin.codegen)
    
    // For Retrofit integration
    implementation(libs.retrofit.converter.moshi)
}
```

## Basic Usage

### Define Data Classes

```kotlin
import com.squareup.moshi.Json
import com.squareup.moshi.JsonClass

@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String,
    val email: String,
    @Json(name = "created_at")
    val createdAt: String
)

@JsonClass(generateAdapter = true)
data class Post(
    val id: Long,
    val title: String,
    val content: String,
    @Json(name = "user_id")
    val userId: Long,
    @Json(name = "published_date")
    val publishedDate: String?
)
```

### Parse JSON

```kotlin
import com.squareup.moshi.Moshi
import com.squareup.moshi.kotlin.reflect.KotlinJsonAdapterFactory

val moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory())
    .build()

// Parse JSON string to object
val json = """{"id":1,"name":"John","email":"john@example.com","created_at":"2025-01-01"}"""
val adapter = moshi.adapter(User::class.java)
val user = adapter.fromJson(json)

// Convert object to JSON
val userJson = adapter.toJson(user)
```

### Parse Lists

```kotlin
import com.squareup.moshi.Types

val listType = Types.newParameterizedType(List::class.java, User::class.java)
val adapter = moshi.adapter<List<User>>(listType)

val json = """[{"id":1,"name":"John","email":"john@example.com","created_at":"2025-01-01"}]"""
val users = adapter.fromJson(json)
```

## Retrofit Integration

### Setup Moshi with Retrofit

```kotlin
import com.squareup.moshi.Moshi
import com.squareup.moshi.kotlin.reflect.KotlinJsonAdapterFactory
import retrofit2.Retrofit
import retrofit2.converter.moshi.MoshiConverterFactory

val moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory())
    .build()

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(MoshiConverterFactory.create(moshi))
    .build()

val apiService = retrofit.create(ApiService::class.java)
```

### API Interface

```kotlin
import retrofit2.Response
import retrofit2.http.*

interface ApiService {
    
    @GET("users")
    suspend fun getUsers(): Response<List<User>>
    
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") userId: Long): Response<User>
    
    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): Response<User>
}

@JsonClass(generateAdapter = true)
data class CreateUserRequest(
    val name: String,
    val email: String
)
```

## With Hilt

### Provide Moshi Instance

```kotlin
import com.squareup.moshi.Moshi
import com.squareup.moshi.kotlin.reflect.KotlinJsonAdapterFactory
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import retrofit2.Retrofit
import retrofit2.converter.moshi.MoshiConverterFactory
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideMoshi(): Moshi {
        return Moshi.Builder()
            .add(KotlinJsonAdapterFactory())
            .add(DateAdapter())  // Custom adapters
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(
        moshi: Moshi,
        okHttpClient: OkHttpClient
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(MoshiConverterFactory.create(moshi))
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

## Custom Adapters

### Date Adapter

```kotlin
import com.squareup.moshi.FromJson
import com.squareup.moshi.ToJson
import java.text.SimpleDateFormat
import java.util.*

class DateAdapter {
    private val dateFormat = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'", Locale.US)
    
    @FromJson
    fun fromJson(string: String): Date {
        return dateFormat.parse(string) ?: Date()
    }
    
    @ToJson
    fun toJson(date: Date): String {
        return dateFormat.format(date)
    }
}

// Register adapter
val moshi = Moshi.Builder()
    .add(DateAdapter())
    .build()
```

### UUID Adapter

```kotlin
import java.util.UUID

class UUIDAdapter {
    @FromJson
    fun fromJson(string: String): UUID {
        return UUID.fromString(string)
    }
    
    @ToJson
    fun toJson(uuid: UUID): String {
        return uuid.toString()
    }
}
```

### Enum Adapter

```kotlin
enum class UserType {
    ADMIN, REGULAR, GUEST
}

// Moshi handles enums automatically, but for custom mapping:
class UserTypeAdapter {
    @FromJson
    fun fromJson(string: String): UserType {
        return when (string.uppercase()) {
            "ADMIN" -> UserType.ADMIN
            "REGULAR" -> UserType.REGULAR
            "GUEST" -> UserType.GUEST
            else -> UserType.GUEST
        }
    }
    
    @ToJson
    fun toJson(type: UserType): String {
        return type.name.lowercase()
    }
}
```

### Custom Object Adapter

```kotlin
@JsonClass(generateAdapter = true)
data class Address(
    val street: String,
    val city: String,
    val zipCode: String
)

class AddressAdapter {
    @FromJson
    fun fromJson(json: Map<String, Any?>): Address {
        return Address(
            street = json["street"] as? String ?: "",
            city = json["city"] as? String ?: "",
            zipCode = json["zip_code"] as? String ?: ""
        )
    }
    
    @ToJson
    fun toJson(address: Address): Map<String, String> {
        return mapOf(
            "street" to address.street,
            "city" to address.city,
            "zip_code" to address.zipCode
        )
    }
}
```

## Handling Null and Default Values

### Nullable Fields

```kotlin
@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String,
    val email: String?,  // Can be null
    val avatarUrl: String? = null,  // Optional with default
    @Json(name = "phone_number")
    val phoneNumber: String? = null
)
```

### Required vs Optional

```kotlin
@JsonClass(generateAdapter = true)
data class UserResponse(
    val id: Long,  // Required
    val name: String,  // Required
    val email: String? = null,  // Optional
    @Json(name = "avatar_url")
    val avatarUrl: String? = null  // Optional with custom name
)
```

## Advanced Features

### Polymorphic Types

```kotlin
import com.squareup.moshi.adapters.PolymorphicJsonAdapterFactory

sealed class Animal {
    abstract val name: String
}

@JsonClass(generateAdapter = true)
data class Dog(
    override val name: String,
    val breed: String
) : Animal()

@JsonClass(generateAdapter = true)
data class Cat(
    override val name: String,
    val color: String
) : Animal()

val moshi = Moshi.Builder()
    .add(
        PolymorphicJsonAdapterFactory.of(Animal::class.java, "type")
            .withSubtype(Dog::class.java, "dog")
            .withSubtype(Cat::class.java, "cat")
    )
    .build()

// JSON format:
// {"type":"dog","name":"Rex","breed":"Labrador"}
// {"type":"cat","name":"Whiskers","color":"orange"}
```

### Generic Response Wrapper

```kotlin
@JsonClass(generateAdapter = true)
data class ApiResponse<T>(
    val success: Boolean,
    val data: T?,
    val message: String?,
    @Json(name = "error_code")
    val errorCode: String?
)

// Usage
val type = Types.newParameterizedType(
    ApiResponse::class.java,
    User::class.java
)
val adapter = moshi.adapter<ApiResponse<User>>(type)
val response = adapter.fromJson(json)
```

### Ignore Unknown Fields

```kotlin
// Moshi ignores unknown fields by default!
// No configuration needed

@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String
    // Any extra fields in JSON are ignored
)
```

### Fallback Values

```kotlin
import com.squareup.moshi.JsonAdapter
import com.squareup.moshi.JsonReader
import com.squareup.moshi.JsonWriter

class FallbackAdapter<T>(
    private val delegate: JsonAdapter<T>,
    private val fallback: T
) : JsonAdapter<T>() {
    
    override fun fromJson(reader: JsonReader): T {
        return try {
            delegate.fromJson(reader) ?: fallback
        } catch (e: Exception) {
            fallback
        }
    }
    
    override fun toJson(writer: JsonWriter, value: T?) {
        delegate.toJson(writer, value)
    }
}
```

## Reflection vs Codegen

### Reflection (Simpler but Slower)

```kotlin
// Add KotlinJsonAdapterFactory
val moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory())
    .build()

// No @JsonClass annotation needed
data class User(
    val id: Long,
    val name: String,
    @Json(name = "created_at")
    val createdAt: String
)
```

### Codegen (Recommended - Faster)

```kotlin
// Requires KSP and @JsonClass annotation
@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String,
    @Json(name = "created_at")
    val createdAt: String
)

val moshi = Moshi.Builder()
    .build()  // No need for KotlinJsonAdapterFactory

// Generates UserJsonAdapter at compile time
```

## Error Handling

### Handle Parse Errors

```kotlin
fun parseUser(json: String): User? {
    return try {
        val adapter = moshi.adapter(User::class.java)
        adapter.fromJson(json)
    } catch (e: Exception) {
        Timber.e(e, "Failed to parse user JSON")
        null
    }
}
```

### Lenient Parsing

```kotlin
val moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory())
    .build()

val adapter = moshi.adapter(User::class.java).lenient()
val user = adapter.fromJson(malformedJson)  // More forgiving
```

## Testing

### Test JSON Parsing

```kotlin
import com.squareup.moshi.Moshi
import com.squareup.moshi.kotlin.reflect.KotlinJsonAdapterFactory
import org.junit.Test

class UserAdapterTest {
    
    private val moshi = Moshi.Builder()
        .add(KotlinJsonAdapterFactory())
        .build()
    
    private val adapter = moshi.adapter(User::class.java)
    
    @Test
    fun `parse user from JSON`() {
        val json = """
            {
                "id": 1,
                "name": "John Doe",
                "email": "john@example.com",
                "created_at": "2025-01-01"
            }
        """.trimIndent()
        
        val user = adapter.fromJson(json)
        
        assertThat(user).isNotNull()
        assertThat(user?.id).isEqualTo(1)
        assertThat(user?.name).isEqualTo("John Doe")
        assertThat(user?.email).isEqualTo("john@example.com")
    }
    
    @Test
    fun `serialize user to JSON`() {
        val user = User(
            id = 1,
            name = "John Doe",
            email = "john@example.com",
            createdAt = "2025-01-01"
        )
        
        val json = adapter.toJson(user)
        
        assertThat(json).contains("\"id\":1")
        assertThat(json).contains("\"name\":\"John Doe\"")
        assertThat(json).contains("\"created_at\":\"2025-01-01\"")
    }
    
    @Test
    fun `handle null values`() {
        val json = """{"id":1,"name":"John","email":null,"created_at":"2025-01-01"}"""
        
        val user = adapter.fromJson(json)
        
        assertThat(user?.email).isNull()
    }
}
```

## Comparison with Other Libraries

### Moshi vs Gson vs Kotlinx Serialization

| Feature | Moshi | Gson | Kotlinx Serialization |
|---------|-------|------|----------------------|
| Kotlin Support | ✅ Excellent | ⚠️ Basic | ✅ Excellent |
| Performance | ✅ Fast | ⚠️ Slower | ✅ Fastest |
| Code Gen | ✅ Yes (KSP) | ❌ No | ✅ Yes |
| Reflection | ⚠️ Optional | ✅ Yes | ❌ No |
| Library Size | ✅ Small | ⚠️ Larger | ✅ Smallest |
| Type Safety | ✅ Good | ⚠️ Basic | ✅ Excellent |
| Null Safety | ✅ Good | ⚠️ Limited | ✅ Excellent |
| Learning Curve | ✅ Easy | ✅ Easy | ⚠️ Moderate |

## Migration from Gson

### Gson Code

```kotlin
import com.google.gson.Gson
import com.google.gson.annotations.SerializedName

data class User(
    val id: Long,
    val name: String,
    @SerializedName("created_at")
    val createdAt: String
)

val gson = Gson()
val user = gson.fromJson(json, User::class.java)
val jsonString = gson.toJson(user)
```

### Moshi Equivalent

```kotlin
import com.squareup.moshi.Json
import com.squareup.moshi.JsonClass
import com.squareup.moshi.Moshi

@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String,
    @Json(name = "created_at")
    val createdAt: String
)

val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(User::class.java)
val user = adapter.fromJson(json)
val jsonString = adapter.toJson(user)
```

## Best Practices

- Use @JsonClass(generateAdapter = true) for code generation
- Use KSP instead of reflection
- Create custom adapters for complex types
- Use @Json(name = "...") for field mapping
- Handle null values explicitly
- Reuse Moshi instance
- Test JSON parsing thoroughly

## Common Patterns

### Date Handling with Built-in Adapter

```kotlin
import com.squareup.moshi.adapters.Rfc3339DateJsonAdapter
import java.util.Date

val moshi = Moshi.Builder()
    .add(Date::class.java, Rfc3339DateJsonAdapter().nullSafe())
    .build()

@JsonClass(generateAdapter = true)
data class Event(
    val id: Long,
    val name: String,
    val date: Date  // Automatically parsed from RFC 3339 format
)
```

### Nested Objects

```kotlin
@JsonClass(generateAdapter = true)
data class UserProfile(
    val id: Long,
    val name: String,
    val address: Address,  // Nested object
    val contacts: List<Contact>  // Nested list
)

@JsonClass(generateAdapter = true)
data class Address(
    val street: String,
    val city: String,
    val country: String
)

@JsonClass(generateAdapter = true)
data class Contact(
    val type: String,
    val value: String
)
```

### Sealed Classes (Polymorphism)

```kotlin
import com.squareup.moshi.adapters.PolymorphicJsonAdapterFactory

sealed class Result {
    @JsonClass(generateAdapter = true)
    data class Success(val data: String) : Result()
    
    @JsonClass(generateAdapter = true)
    data class Error(val message: String) : Result()
}

val moshi = Moshi.Builder()
    .add(
        PolymorphicJsonAdapterFactory.of(Result::class.java, "status")
            .withSubtype(Result.Success::class.java, "success")
            .withSubtype(Result.Error::class.java, "error")
    )
    .build()

// JSON:
// {"status":"success","data":"Hello"}
// {"status":"error","message":"Failed"}
```

### Default Values

```kotlin
@JsonClass(generateAdapter = true)
data class Config(
    val apiUrl: String,
    val timeout: Int = 30,  // Default value
    val retries: Int = 3,   // Default value
    val debug: Boolean = false  // Default value
)

// If JSON doesn't include these fields, defaults are used
```

## ProGuard Rules

```proguard
# Moshi
-keepclasseswithmembers class * {
    @com.squareup.moshi.* <methods>;
}
-keep @com.squareup.moshi.JsonQualifier @interface *

# Keep generated JsonAdapters
-keep class **JsonAdapter {
    <init>(...);
    <fields>;
}
-keepnames @com.squareup.moshi.JsonClass class *

# Keep fields
-keepclassmembers class * {
    @com.squareup.moshi.FromJson <methods>;
    @com.squareup.moshi.ToJson <methods>;
}

# Keep your data model classes
-keep class com.example.app.data.model.** { *; }
```

## Common Issues

### Missing Adapter

**Error**: "Platform class java.util.Date requires explicit JsonAdapter"

**Solution**: Add adapter for custom types
```kotlin
val moshi = Moshi.Builder()
    .add(Date::class.java, Rfc3339DateJsonAdapter())
    .build()
```

### KSP Not Generating Adapters

**Problem**: Adapters not generated

**Solution**: 
1. Ensure KSP plugin is applied
2. Add `@JsonClass(generateAdapter = true)`
3. Rebuild project
4. Check `build/generated/ksp/`

### Null Safety Issues

```kotlin
// Bad - Can crash if null
@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String  // Will crash if null in JSON
)

// Good - Handle nullability
@JsonClass(generateAdapter = true)
data class User(
    val id: Long,
    val name: String?,  // Nullable
    val email: String = "unknown@example.com"  // Default value
)
```

## Advantages

- Kotlin-first design
- Respects null safety
- Faster than Gson with codegen
- Smaller library size
- Better type safety
- Actively maintained
- Clean API

## When to Use Moshi

- New Kotlin projects
- Need better Kotlin support than Gson
- Performance-critical apps
- Already using Square libraries

## Alternatives

- **Kotlinx Serialization** - Pure Kotlin, multiplatform
- **Gson** - Legacy projects, Java codebases
- **Jackson** - Complex JSON requirements

## Resources

- [Official Documentation](https://github.com/square/moshi)
- [Moshi GitHub](https://github.com/square/moshi)
- [Retrofit with Moshi](https://github.com/square/retrofit/tree/master/retrofit-converters/moshi)
- [KSP Documentation](https://kotlinlang.org/docs/ksp-overview.html)

