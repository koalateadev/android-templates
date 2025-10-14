# Lottie Animations Setup

## Overview
Lottie is a library that parses Adobe After Effects animations exported as JSON and renders them natively on mobile platforms.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
lottie = "6.5.2"

[libraries]
lottie-compose = { group = "com.airbnb.android", name = "lottie-compose", version.ref = "lottie" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.lottie.compose)
}
```

## Basic Usage

### Simple Animation

```kotlin
import com.airbnb.lottie.compose.*

@Composable
fun SimpleAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    
    LottieAnimation(
        composition = composition,
        modifier = Modifier.size(200.dp)
    )
}
```

### With Progress Control

```kotlin
@Composable
fun AnimationWithProgress() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    val progress by animateLottieCompositionAsState(composition)
    
    LottieAnimation(
        composition = composition,
        progress = { progress },
        modifier = Modifier.size(200.dp)
    )
}
```

## Animation Control

### Loop Animation

```kotlin
@Composable
fun LoopingAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.loading)
    )
    val progress by animateLottieCompositionAsState(
        composition = composition,
        iterations = LottieConstants.IterateForever  // Loop forever
    )
    
    LottieAnimation(
        composition = composition,
        progress = { progress }
    )
}
```

### Play Once

```kotlin
@Composable
fun OneTimeAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.success)
    )
    val progress by animateLottieCompositionAsState(
        composition = composition,
        iterations = 1  // Play once
    )
    
    LottieAnimation(
        composition = composition,
        progress = { progress }
    )
}
```

### Control Playback

```kotlin
@Composable
fun ControlledAnimation() {
    var isPlaying by remember { mutableStateOf(true) }
    
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    val progress by animateLottieCompositionAsState(
        composition = composition,
        isPlaying = isPlaying,
        restartOnPlay = true,
        speed = 1.5f  // 1.5x speed
    )
    
    Column {
        LottieAnimation(
            composition = composition,
            progress = { progress },
            modifier = Modifier.size(200.dp)
        )
        
        Button(onClick = { isPlaying = !isPlaying }) {
            Text(if (isPlaying) "Pause" else "Play")
        }
    }
}
```

## Loading Animations

### From Network

```kotlin
@Composable
fun NetworkAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.Url("https://assets.lottiefiles.com/packages/lf20_uwR49r.json")
    )
    val progress by animateLottieCompositionAsState(composition)
    
    LottieAnimation(
        composition = composition,
        progress = { progress }
    )
}
```

### From Assets

```kotlin
@Composable
fun AssetAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.Asset("animations/loading.json")
    )
    val progress by animateLottieCompositionAsState(composition)
    
    LottieAnimation(
        composition = composition,
        progress = { progress }
    )
}
```

### From File

```kotlin
@Composable
fun FileAnimation(file: File) {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.File(file.absolutePath)
    )
    val progress by animateLottieCompositionAsState(composition)
    
    LottieAnimation(
        composition = composition,
        progress = { progress }
    )
}
```

## Advanced Features

### Handle Loading States

```kotlin
@Composable
fun AnimationWithLoadingState() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    
    when {
        composition == null -> {
            CircularProgressIndicator()
        }
        composition?.warnings?.isNotEmpty() == true -> {
            Text("Animation has warnings")
        }
        else -> {
            val progress by animateLottieCompositionAsState(composition)
            LottieAnimation(
                composition = composition,
                progress = { progress }
            )
        }
    }
}
```

### Dynamic Properties

```kotlin
import com.airbnb.lottie.compose.rememberLottieDynamicProperties
import com.airbnb.lottie.compose.rememberLottieDynamicProperty
import com.airbnb.lottie.LottieProperty

@Composable
fun DynamicColorAnimation() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    
    val dynamicProperties = rememberLottieDynamicProperties(
        rememberLottieDynamicProperty(
            property = LottieProperty.COLOR,
            value = Color.Red.toArgb(),
            keyPath = arrayOf("**")
        )
    )
    
    LottieAnimation(
        composition = composition,
        dynamicProperties = dynamicProperties
    )
}
```

### Animation Callbacks

```kotlin
@Composable
fun AnimationWithCallbacks() {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.animation)
    )
    val animationState = rememberLottieAnimatable()
    
    LaunchedEffect(composition) {
        composition?.let {
            animationState.animate(
                composition = it,
                iterations = 1
            )
        }
    }
    
    LaunchedEffect(animationState.isAtEnd) {
        if (animationState.isAtEnd) {
            // Animation completed
        }
    }
    
    LottieAnimation(
        composition = composition,
        progress = { animationState.progress }
    )
}
```

## Common Use Cases

### Loading Indicator

```kotlin
@Composable
fun LoadingIndicator() {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        val composition by rememberLottieComposition(
            LottieCompositionSpec.RawRes(R.raw.loading)
        )
        val progress by animateLottieCompositionAsState(
            composition = composition,
            iterations = LottieConstants.IterateForever
        )
        
        LottieAnimation(
            composition = composition,
            progress = { progress },
            modifier = Modifier.size(100.dp)
        )
    }
}
```

### Success Animation

```kotlin
@Composable
fun SuccessAnimation(onComplete: () -> Unit = {}) {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.success)
    )
    val progress by animateLottieCompositionAsState(
        composition = composition,
        iterations = 1
    )
    
    LaunchedEffect(progress) {
        if (progress == 1f) {
            onComplete()
        }
    }
    
    LottieAnimation(
        composition = composition,
        progress = { progress },
        modifier = Modifier.size(150.dp)
    )
}
```

### Error Animation

```kotlin
@Composable
fun ErrorAnimation(onRetry: () -> Unit) {
    Column(
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        val composition by rememberLottieComposition(
            LottieCompositionSpec.RawRes(R.raw.error)
        )
        val progress by animateLottieCompositionAsState(
            composition = composition,
            iterations = 1
        )
        
        LottieAnimation(
            composition = composition,
            progress = { progress },
            modifier = Modifier.size(150.dp)
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Text("Something went wrong")
        
        Button(onClick = onRetry) {
            Text("Retry")
        }
    }
}
```

### Splash Screen

```kotlin
@Composable
fun SplashScreen(onComplete: () -> Unit) {
    Box(
        modifier = Modifier.fillMaxSize(),
        contentAlignment = Alignment.Center
    ) {
        val composition by rememberLottieComposition(
            LottieCompositionSpec.RawRes(R.raw.splash)
        )
        val progress by animateLottieCompositionAsState(
            composition = composition,
            iterations = 1
        )
        
        LaunchedEffect(progress) {
            if (progress == 1f) {
                delay(500)
                onComplete()
            }
        }
        
        LottieAnimation(
            composition = composition,
            progress = { progress },
            modifier = Modifier.size(250.dp)
        )
    }
}
```

### Pull to Refresh

```kotlin
@Composable
fun PullToRefreshAnimation(progress: Float) {
    val composition by rememberLottieComposition(
        LottieCompositionSpec.RawRes(R.raw.pull_to_refresh)
    )
    
    LottieAnimation(
        composition = composition,
        progress = { progress },
        modifier = Modifier.size(50.dp)
    )
}
```

## Performance Tips

### Preload Animations

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        // Preload commonly used animations
        LottieCompositionFactory.fromRawRes(this, R.raw.loading)
        LottieCompositionFactory.fromRawRes(this, R.raw.success)
    }
}
```

### Cache Animations

```kotlin
@Composable
fun CachedAnimation() {
    val composition by rememberLottieComposition(
        spec = LottieCompositionSpec.RawRes(R.raw.animation),
        cacheKey = "my_animation_cache_key"
    )
    
    LottieAnimation(composition = composition)
}
```

## Finding Animations

Free Lottie animations:
- [LottieFiles](https://lottiefiles.com/)
- [IconScout](https://iconscout.com/lottie-animations)
- [Lottie Lab](https://lottielab.com/)

## Best Practices

1. ✅ Use appropriate animation size (don't make them too large)
2. ✅ Cache frequently used animations
3. ✅ Handle loading states
4. ✅ Use loops for continuous animations
5. ✅ Clean up animations when not visible
6. ✅ Test animations on different devices
7. ✅ Keep animation files small (< 100KB)
8. ✅ Use hardware acceleration
9. ✅ Provide fallback for old devices
10. ✅ Test animations with different speeds

## Common Issues

### Animation Not Playing

```kotlin
// Make sure to animate
val progress by animateLottieCompositionAsState(
    composition = composition,
    isPlaying = true  // Ensure this is true
)
```

### Animation Too Fast/Slow

```kotlin
val progress by animateLottieCompositionAsState(
    composition = composition,
    speed = 0.5f  // Adjust speed (0.5 = half speed, 2.0 = double speed)
)
```

### Memory Issues

```kotlin
// Dispose when not needed
DisposableEffect(Unit) {
    onDispose {
        // Clean up if needed
    }
}
```

## Resources

- [Official Documentation](https://airbnb.io/lottie/)
- [Lottie Compose](https://github.com/airbnb/lottie/blob/master/android-compose.md)
- [LottieFiles Marketplace](https://lottiefiles.com/)
- [After Effects Plugin](https://airbnb.io/lottie/#/after-effects)

