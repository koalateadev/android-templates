# Android Common Patterns

Practical, copy-paste ready code templates for common Android development scenarios.

## 📱 Available Patterns

### [Compose UI Patterns](./compose-ui-patterns.md)
Ready-to-use UI components and patterns:
- ✅ Loading states (spinner, skeleton, overlay)
- ✅ Error handling (full screen, inline)
- ✅ Empty states
- ✅ Pull to refresh
- ✅ Infinite scroll / pagination
- ✅ Search with debounce
- ✅ Dialogs (confirmation, input, date/time, multi-select)
- ✅ Bottom sheets
- ✅ Swipe to delete
- ✅ Form validation
- ✅ Tab layouts
- ✅ **Toasts** (simple, custom)
- ✅ **Snackbars** (basic, with action, custom)
- ✅ **Advanced forms** (multi-step, dynamic fields)
- ✅ **Filters & Sort** (chips, bottom sheet, menu)

### [State Management](./state-management.md)
Master state management in Compose:
- ✅ State hoisting strategies
- ✅ Remember variants (remember, rememberSaveable, rememberUpdatedState)
- ✅ Side effects (LaunchedEffect, DisposableEffect, SideEffect)
- ✅ MVI architecture implementation
- ✅ Unidirectional data flow
- ✅ CompositionLocal usage
- ✅ Derived state patterns
- ✅ State restoration
- ✅ Undo/Redo patterns
- ✅ Best practices and common mistakes

### [Animation & Transitions](./animations.md)
Create beautiful animations:
- ✅ Basic animations (color, size, float)
- ✅ AnimatedVisibility (enter/exit)
- ✅ AnimatedContent (screen transitions)
- ✅ List animations (enter, exit, reordering)
- ✅ Gesture animations (swipe, drag)
- ✅ Screen transitions (fade, slide, scale)
- ✅ Shared element transitions
- ✅ Custom animations (pulse, shimmer, bounce)
- ✅ Progress animations
- ✅ Animation specs (spring, tween, keyframes)

### [Security Patterns](./security.md)
Secure your Android app:
- ✅ Secure data storage (EncryptedSharedPreferences, EncryptedFile)
- ✅ API key management (BuildConfig, NDK, backend)
- ✅ Certificate pinning
- ✅ JWT token management (refresh, expiration)
- ✅ Biometric authentication
- ✅ Data encryption (Android Keystore)
- ✅ ProGuard security rules
- ✅ Network security config
- ✅ Root detection
- ✅ Input validation
- ✅ Secure WebView
- ✅ Security checklist

### [Performance Optimization](./performance.md)
Optimize app performance:
- ✅ Recomposition optimization
- ✅ LazyColumn optimization (keys, contentType)
- ✅ Image loading and caching
- ✅ Memory management
- ✅ State optimization
- ✅ Stability annotations
- ✅ derivedStateOf usage
- ✅ Profiling with Composition Tracing
- ✅ Performance checklist

### [Offline-First Patterns](./offline-first.md)
Build apps that work offline:
- ✅ Network + Database strategy
- ✅ Connection monitoring
- ✅ Background sync with WorkManager
- ✅ Incremental sync
- ✅ Conflict resolution
- ✅ Queue-based sync
- ✅ Optimistic updates
- ✅ Cache expiration

### [Accessibility](./accessibility.md)
Make apps accessible to everyone:
- ✅ Content descriptions
- ✅ Semantic properties
- ✅ Touch target sizes
- ✅ Screen reader support
- ✅ Keyboard navigation
- ✅ Color contrast checking
- ✅ Text scaling
- ✅ Accessibility testing
- ✅ TalkBack support
- ✅ Accessibility checklist

### [Architecture Patterns](./architecture.md)
Structure your codebase:
- ✅ Repository pattern
- ✅ Use Case pattern
- ✅ Mapper pattern (Entity → Domain → UI)
- ✅ MVVM implementation
- ✅ Clean Architecture
- ✅ Multi-module architecture
- ✅ Dependency injection strategies

### [Testing Patterns](./testing.md)
Write better tests:
- ✅ Test data builders
- ✅ Robot pattern for UI tests
- ✅ Fake repositories
- ✅ ViewModel testing
- ✅ Compose UI testing
- ✅ Flow testing with Turbine
- ✅ Repository testing with MockWebServer
- ✅ Integration testing
- ✅ Test helpers and extensions

## 🎯 How to Use

1. **Browse** the pattern you need
2. **Copy** the code snippet
3. **Adapt** to your specific use case
4. **Test** thoroughly in your app

## 💡 Examples

### Quick Loading State

```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> LoadingScreen()
        is UiState.Success -> SuccessContent((uiState as UiState.Success).data)
        is UiState.Error -> ErrorScreen((uiState as UiState.Error).message)
    }
}
```

### Share Text

```kotlin
val context = LocalContext.current
Button(onClick = {
    val sendIntent = Intent().apply {
        action = Intent.ACTION_SEND
        putExtra(Intent.EXTRA_TEXT, "Check this out!")
        type = "text/plain"
    }
    context.startActivity(Intent.createChooser(sendIntent, null))
}) {
    Text("Share")
}
```

### Request Camera Permission

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraFeature() {
    val permissionState = rememberPermissionState(android.Manifest.permission.CAMERA)
    
    if (permissionState.status.isGranted) {
        CameraScreen()
    } else {
        Button(onClick = { permissionState.launchPermissionRequest() }) {
            Text("Grant Camera Permission")
        }
    }
}
```

## 🔗 Related

- [Setup Guides](../setup/) - Library configuration and setup
- [Quick Start](../QUICKSTART.md) - Complete project setup

## 📝 Best Practices

- Always handle edge cases (no network, no permission, etc.)
- Provide user feedback (loading, errors, success)
- Test on different Android versions
- Follow Material Design guidelines
- Keep code clean and maintainable
- Add proper content descriptions for accessibility

## 🤝 Contributing

Have a useful pattern to share? Feel free to add it!

Patterns should be:
- ✅ Practical and commonly needed
- ✅ Well-documented with examples
- ✅ Copy-paste ready
- ✅ Following Android best practices
- ✅ Compatible with modern Android (API 24+)

