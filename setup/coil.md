# Coil (Image Loading) Setup

## Overview
Coil is a modern image loading library for Android backed by Kotlin Coroutines, optimized for Jetpack Compose.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
coil = "2.7.0"

[libraries]
coil = { group = "io.coil-kt", name = "coil", version.ref = "coil" }
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }
coil-gif = { group = "io.coil-kt", name = "coil-gif", version.ref = "coil" }
coil-svg = { group = "io.coil-kt", name = "coil-svg", version.ref = "coil" }
coil-video = { group = "io.coil-kt", name = "coil-video", version.ref = "coil" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    // For Compose
    implementation(libs.coil.compose)
    
    // Optional: GIF support
    implementation(libs.coil.gif)
    
    // Optional: SVG support
    implementation(libs.coil.svg)
    
    // Optional: Video frames
    implementation(libs.coil.video)
}
```

### 3. Internet Permission

In `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
```

## Basic Usage (Compose)

### Simple Image Loading

```kotlin
import coil.compose.AsyncImage

@Composable
fun UserAvatar(imageUrl: String) {
    AsyncImage(
        model = imageUrl,
        contentDescription = "User avatar",
        modifier = Modifier
            .size(100.dp)
            .clip(CircleShape)
    )
}
```

### With Placeholder and Error

```kotlin
@Composable
fun ImageWithPlaceholder(imageUrl: String) {
    AsyncImage(
        model = imageUrl,
        contentDescription = "Image",
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error),
        modifier = Modifier.fillMaxWidth()
    )
}
```

### Advanced Configuration

```kotlin
import coil.compose.AsyncImage
import coil.request.ImageRequest
import androidx.compose.ui.platform.LocalContext

@Composable
fun AdvancedImage(imageUrl: String) {
    val context = LocalContext.current
    
    AsyncImage(
        model = ImageRequest.Builder(context)
            .data(imageUrl)
            .crossfade(true)
            .crossfade(300)
            .placeholder(R.drawable.placeholder)
            .error(R.drawable.error)
            .fallback(R.drawable.fallback)
            .memoryCacheKey(imageUrl)
            .diskCacheKey(imageUrl)
            .build(),
        contentDescription = "Image",
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    )
}
```

### SubcomposeAsyncImage (More Control)

```kotlin
import coil.compose.SubcomposeAsyncImage

@Composable
fun CustomLoadingImage(imageUrl: String) {
    SubcomposeAsyncImage(
        model = imageUrl,
        contentDescription = "Image",
        loading = {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        },
        error = {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Icon(
                    imageVector = Icons.Default.Error,
                    contentDescription = "Error loading image"
                )
            }
        },
        success = { success ->
            success.result.image?.let {
                Image(
                    painter = rememberAsyncImagePainter(it),
                    contentDescription = "Loaded image",
                    contentScale = ContentScale.Crop,
                    modifier = Modifier.fillMaxSize()
                )
            }
        }
    )
}
```

## ImageRequest Builder

### Transformations

```kotlin
import coil.transform.*

val imageRequest = ImageRequest.Builder(context)
    .data(imageUrl)
    .transformations(
        CircleCropTransformation(),
        RoundedCornersTransformation(16.dp.toPx()),
        GrayscaleTransformation(),
        BlurTransformation(context, 25f, 3f)
    )
    .build()
```

### Custom Transformations

```kotlin
import coil.transform.Transformation
import coil.size.Size

class CustomTransformation : Transformation {
    override val cacheKey: String = "custom_transformation"
    
    override suspend fun transform(input: Bitmap, size: Size): Bitmap {
        // Apply custom transformation
        return input
    }
}
```

### Size and Scale

```kotlin
ImageRequest.Builder(context)
    .data(imageUrl)
    .size(800, 600)              // Resize to specific size
    .size(Size.ORIGINAL)         // Use original size
    .scale(Scale.FILL)           // Scale.FIT or Scale.FILL
    .precision(Precision.EXACT)  // Size precision
    .build()
```

### Target and Listener

```kotlin
ImageRequest.Builder(context)
    .data(imageUrl)
    .target(
        onStart = { placeholder ->
            // Image loading started
        },
        onSuccess = { result ->
            // Image loaded successfully
        },
        onError = { error ->
            // Error loading image
        }
    )
    .listener(
        onStart = { request ->
            Log.d("Coil", "Loading started")
        },
        onSuccess = { request, result ->
            Log.d("Coil", "Loading success")
        },
        onError = { request, result ->
            Log.e("Coil", "Loading error")
        }
    )
    .build()
```

## Configuration

### Global Configuration

```kotlin
import android.app.Application
import coil.ImageLoader
import coil.ImageLoaderFactory
import coil.disk.DiskCache
import coil.memory.MemoryCache
import coil.request.CachePolicy
import okhttp3.OkHttpClient

class MyApplication : Application(), ImageLoaderFactory {
    
    override fun newImageLoader(): ImageLoader {
        return ImageLoader.Builder(this)
            .memoryCache {
                MemoryCache.Builder(this)
                    .maxSizePercent(0.25) // Use 25% of app memory
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50 * 1024 * 1024) // 50 MB
                    .build()
            }
            .okHttpClient {
                OkHttpClient.Builder()
                    .cache(
                        Cache(
                            directory = cacheDir.resolve("http_cache"),
                            maxSize = 20 * 1024 * 1024 // 20 MB
                        )
                    )
                    .build()
            }
            .respectCacheHeaders(false)
            .crossfade(true)
            .build()
    }
}
```

### With Hilt

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object CoilModule {
    
    @Provides
    @Singleton
    fun provideImageLoader(
        @ApplicationContext context: Context,
        okHttpClient: OkHttpClient
    ): ImageLoader {
        return ImageLoader.Builder(context)
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25)
                    .build()
            }
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(50 * 1024 * 1024)
                    .build()
            }
            .okHttpClient(okHttpClient)
            .crossfade(true)
            .build()
    }
}
```

## Caching

### Cache Policies

```kotlin
ImageRequest.Builder(context)
    .data(imageUrl)
    .memoryCachePolicy(CachePolicy.ENABLED)  // ENABLED, DISABLED, READ_ONLY, WRITE_ONLY
    .diskCachePolicy(CachePolicy.ENABLED)
    .networkCachePolicy(CachePolicy.ENABLED)
    .build()
```

### Clear Cache

```kotlin
// Clear memory cache
imageLoader.memoryCache?.clear()

// Clear disk cache
imageLoader.diskCache?.clear()

// Clear specific image
imageLoader.diskCache?.remove(imageUrl)
```

## Custom Data Types

### Load from Custom Sources

```kotlin
// Custom data class
data class CustomImage(val id: String, val url: String)

// Custom Fetcher
class CustomImageFetcher(
    private val data: CustomImage,
    private val options: Options
) : Fetcher {
    
    override suspend fun fetch(): FetchResult {
        // Fetch image data
        val url = data.url
        // ... fetch logic
        return SourceResult(source, mimeType, dataSource)
    }
    
    class Factory : Fetcher.Factory<CustomImage> {
        override fun create(
            data: CustomImage,
            options: Options,
            imageLoader: ImageLoader
        ): Fetcher {
            return CustomImageFetcher(data, options)
        }
    }
}

// Register in ImageLoader
ImageLoader.Builder(context)
    .components {
        add(CustomImageFetcher.Factory())
    }
    .build()
```

## Preloading

### Preload Images

```kotlin
val imageLoader = ImageLoader(context)

// Preload single image
val request = ImageRequest.Builder(context)
    .data(imageUrl)
    .build()
imageLoader.enqueue(request)

// Preload multiple images
imageUrls.forEach { url ->
    val request = ImageRequest.Builder(context)
        .data(url)
        .build()
    imageLoader.enqueue(request)
}
```

## Special Use Cases

### Load GIF

```kotlin
// Add dependency: implementation(libs.coil.gif)

AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/image.gif")
        .decoderFactory(ImageDecoderDecoder.Factory())
        .build(),
    contentDescription = "Animated GIF"
)
```

### Load SVG

```kotlin
// Add dependency: implementation(libs.coil.svg)

AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/image.svg")
        .build(),
    contentDescription = "SVG Image"
)
```

### Load Video Thumbnail

```kotlin
// Add dependency: implementation(libs.coil.video)

AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data(videoUri)
        .build(),
    contentDescription = "Video thumbnail"
)
```

### Load from Resources

```kotlin
AsyncImage(
    model = R.drawable.my_image,
    contentDescription = "Resource image"
)
```

### Load from File

```kotlin
val file = File("/path/to/image.jpg")
AsyncImage(
    model = file,
    contentDescription = "File image"
)
```

## Testing

### Fake ImageLoader

```kotlin
import coil.test.FakeImageLoaderEngine

val engine = FakeImageLoaderEngine.Builder()
    .intercept("https://example.com/image.jpg", ColorDrawable(Color.RED))
    .default(ColorDrawable(Color.BLUE))
    .build()

val imageLoader = ImageLoader.Builder(context)
    .components { add(engine) }
    .build()
```

## Best Practices

1. ✅ Use AsyncImage for Compose
2. ✅ Configure global ImageLoader in Application class
3. ✅ Use appropriate cache policies
4. ✅ Add placeholders for better UX
5. ✅ Handle error states
6. ✅ Use transformations for common image manipulations
7. ✅ Preload images when needed
8. ✅ Set appropriate content descriptions for accessibility
9. ✅ Use disk cache for network images
10. ✅ Monitor memory usage

## Common Patterns

### List Item Images

```kotlin
@Composable
fun UserListItem(user: User) {
    Row {
        AsyncImage(
            model = ImageRequest.Builder(LocalContext.current)
                .data(user.avatarUrl)
                .crossfade(true)
                .transformations(CircleCropTransformation())
                .build(),
            contentDescription = "User avatar",
            modifier = Modifier.size(48.dp)
        )
        
        Text(user.name)
    }
}
```

### Full Screen Image

```kotlin
@Composable
fun FullScreenImage(imageUrl: String) {
    SubcomposeAsyncImage(
        model = ImageRequest.Builder(LocalContext.current)
            .data(imageUrl)
            .size(Size.ORIGINAL)
            .build(),
        contentDescription = "Full screen image",
        contentScale = ContentScale.Fit,
        modifier = Modifier.fillMaxSize(),
        loading = {
            Box(modifier = Modifier.fillMaxSize()) {
                CircularProgressIndicator(
                    modifier = Modifier.align(Alignment.Center)
                )
            }
        }
    )
}
```

## Resources

- [Official Documentation](https://coil-kt.github.io/coil/)
- [Compose Guide](https://coil-kt.github.io/coil/compose/)
- [GitHub Repository](https://github.com/coil-kt/coil)

