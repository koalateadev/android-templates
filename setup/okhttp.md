# OkHttp Setup

## Overview
OkHttp is a modern HTTP client for Android and Java applications that makes networking easy and efficient.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
okhttp = "4.12.0"

[libraries]
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
okhttp-mockwebserver = { group = "com.squareup.okhttp3", name = "mockwebserver", version.ref = "okhttp" }

[bundles]
okhttp = ["okhttp", "okhttp-logging-interceptor"]
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.bundles.okhttp)
    
    // For testing
    testImplementation(libs.okhttp.mockwebserver)
}
```

### 3. Add Internet Permission

In `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### 4. Basic OkHttp Client

```kotlin
import okhttp3.OkHttpClient
import okhttp3.Request
import java.util.concurrent.TimeUnit

object HttpClient {
    val client = OkHttpClient.Builder()
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()
}

// Usage
suspend fun fetchData(url: String): String = withContext(Dispatchers.IO) {
    val request = Request.Builder()
        .url(url)
        .build()
    
    client.newCall(request).execute().use { response ->
        if (!response.isSuccessful) throw IOException("Unexpected code $response")
        response.body?.string() ?: ""
    }
}
```

### 5. Advanced Configuration

```kotlin
import okhttp3.Cache
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import java.io.File
import java.util.concurrent.TimeUnit

class OkHttpProvider(private val context: Context) {
    
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(createLoggingInterceptor())
            .addInterceptor(createAuthInterceptor())
            .addNetworkInterceptor(createCacheInterceptor())
            .cache(createCache())
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .build()
    }
    
    private fun createLoggingInterceptor(): HttpLoggingInterceptor {
        return HttpLoggingInterceptor().apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }
    }
    
    private fun createAuthInterceptor(): Interceptor {
        return Interceptor { chain ->
            val original = chain.request()
            val request = original.newBuilder()
                .header("Authorization", "Bearer YOUR_TOKEN")
                .header("Accept", "application/json")
                .method(original.method, original.body)
                .build()
            chain.proceed(request)
        }
    }
    
    private fun createCacheInterceptor(): Interceptor {
        return Interceptor { chain ->
            val response = chain.proceed(chain.request())
            response.newBuilder()
                .header("Cache-Control", "public, max-age=3600")
                .removeHeader("Pragma")
                .build()
        }
    }
    
    private fun createCache(): Cache {
        val cacheSize = 10L * 1024 * 1024 // 10 MB
        val cacheDir = File(context.cacheDir, "http_cache")
        return Cache(cacheDir, cacheSize)
    }
}
```

### 6. With Hilt Dependency Injection

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideOkHttpClient(
        @ApplicationContext context: Context
    ): OkHttpClient {
        return OkHttpProvider(context).provideOkHttpClient()
    }
}
```

## Common Use Cases

### Synchronous GET Request

```kotlin
fun fetchData(url: String): String {
    val request = Request.Builder()
        .url(url)
        .build()
    
    client.newCall(request).execute().use { response ->
        return response.body?.string() ?: ""
    }
}
```

### Asynchronous GET Request

```kotlin
fun fetchDataAsync(url: String, callback: (String?) -> Unit) {
    val request = Request.Builder()
        .url(url)
        .build()
    
    client.newCall(request).enqueue(object : Callback {
        override fun onFailure(call: Call, e: IOException) {
            callback(null)
        }
        
        override fun onResponse(call: Call, response: Response) {
            response.use {
                callback(response.body?.string())
            }
        }
    })
}
```

### POST Request with JSON

```kotlin
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.RequestBody.Companion.toRequestBody

suspend fun postJson(url: String, json: String): String = withContext(Dispatchers.IO) {
    val mediaType = "application/json; charset=utf-8".toMediaType()
    val body = json.toRequestBody(mediaType)
    
    val request = Request.Builder()
        .url(url)
        .post(body)
        .build()
    
    client.newCall(request).execute().use { response ->
        response.body?.string() ?: ""
    }
}
```

### Multipart File Upload

```kotlin
import okhttp3.MediaType.Companion.toMediaTypeOrNull
import okhttp3.MultipartBody
import okhttp3.RequestBody.Companion.asRequestBody

suspend fun uploadFile(url: String, file: File): String = withContext(Dispatchers.IO) {
    val requestBody = MultipartBody.Builder()
        .setType(MultipartBody.FORM)
        .addFormDataPart(
            "file",
            file.name,
            file.asRequestBody("image/*".toMediaTypeOrNull())
        )
        .build()
    
    val request = Request.Builder()
        .url(url)
        .post(requestBody)
        .build()
    
    client.newCall(request).execute().use { response ->
        response.body?.string() ?: ""
    }
}
```

### WebSocket Connection

```kotlin
import okhttp3.WebSocket
import okhttp3.WebSocketListener

class MyWebSocketListener : WebSocketListener() {
    override fun onOpen(webSocket: WebSocket, response: Response) {
        webSocket.send("Hello, Server!")
    }
    
    override fun onMessage(webSocket: WebSocket, text: String) {
        println("Received: $text")
    }
    
    override fun onFailure(webSocket: WebSocket, t: Throwable, response: Response?) {
        println("Error: ${t.message}")
    }
}

fun connectWebSocket(url: String) {
    val request = Request.Builder()
        .url(url)
        .build()
    
    val webSocket = client.newWebSocket(request, MyWebSocketListener())
}
```

## Testing with MockWebServer

```kotlin
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import org.junit.After
import org.junit.Before
import org.junit.Test

class NetworkTest {
    private lateinit var mockWebServer: MockWebServer
    
    @Before
    fun setup() {
        mockWebServer = MockWebServer()
        mockWebServer.start()
    }
    
    @After
    fun teardown() {
        mockWebServer.shutdown()
    }
    
    @Test
    fun testApi() = runTest {
        // Arrange
        val response = MockResponse()
            .setResponseCode(200)
            .setBody("""{"message": "success"}""")
        mockWebServer.enqueue(response)
        
        val url = mockWebServer.url("/api/test")
        
        // Act
        val result = fetchData(url.toString())
        
        // Assert
        assertThat(result).contains("success")
    }
}
```

## Best Practices

- Reuse OkHttpClient instances
- Set appropriate timeouts
- Add logging only in debug builds
- Implement caching for performance
- Use interceptors for cross-cutting concerns
- Close response bodies to avoid leaks

## Common Interceptors

### Retry Interceptor

```kotlin
class RetryInterceptor(private val maxRetries: Int = 3) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        var attempt = 0
        var response: Response? = null
        var exception: IOException? = null
        
        while (attempt < maxRetries) {
            try {
                response = chain.proceed(chain.request())
                if (response.isSuccessful) {
                    return response
                }
                response.close()
            } catch (e: IOException) {
                exception = e
            }
            attempt++
        }
        
        throw exception ?: IOException("Max retries exceeded")
    }
}
```

### Network Connectivity Check

```kotlin
class ConnectivityInterceptor(private val context: Context) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        if (!isNetworkAvailable()) {
            throw NoConnectivityException()
        }
        return chain.proceed(chain.request())
    }
    
    private fun isNetworkAvailable(): Boolean {
        val cm = context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = cm.activeNetwork ?: return false
        val capabilities = cm.getNetworkCapabilities(network) ?: return false
        return capabilities.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
    }
}

class NoConnectivityException : IOException("No network connection")
```

## Resources

- [Official Documentation](https://square.github.io/okhttp/)
- [Recipes](https://square.github.io/okhttp/recipes/)
- [GitHub Repository](https://github.com/square/okhttp)

