# Common Patterns

Code templates for common Android development scenarios.

## Available Patterns

### [Compose UI Patterns](./compose-ui-patterns.md)
UI components and patterns:
- Loading states
- Error handling
- Empty states
- Pull to refresh
- Infinite scroll
- Search with debounce
- Dialogs (confirmation, input, date/time, multi-select)
- Bottom sheets
- Swipe to delete
- Form validation
- Toasts and snackbars
- Multi-step forms
- Filters and sorting

### [State Management](./state-management.md)
State management in Compose:
- State hoisting
- Remember variants
- Side effects (LaunchedEffect, DisposableEffect)
- MVI architecture
- Unidirectional data flow
- CompositionLocal
- Derived state
- State restoration

### [Animation & Transitions](./animations.md)
Animation patterns:
- Basic animations
- AnimatedVisibility
- AnimatedContent
- List animations
- Gesture animations
- Screen transitions
- Shared elements
- Custom animations

### [Security Patterns](./security.md)
Security implementation:
- Encrypted storage
- API key management
- Certificate pinning
- Token management
- Biometric authentication
- Data encryption
- Network security

### [Performance Optimization](./performance.md)
Performance patterns:
- Recomposition optimization
- LazyColumn best practices
- Image caching
- Memory management
- Stability annotations
- Profiling techniques

### [Offline-First Patterns](./offline-first.md)
Offline-first implementation:
- Network + database strategy
- Connection monitoring
- Background sync
- Conflict resolution
- Optimistic updates

### [Accessibility](./accessibility.md)
Accessibility implementation:
- Content descriptions
- Semantic properties
- Touch target sizing
- Screen reader support
- Keyboard navigation
- Color contrast

### [Architecture Patterns](./architecture.md)
Code organization:
- Repository pattern
- Use Case pattern
- Mapper pattern
- MVVM
- Clean Architecture
- Multi-module setup

### [Testing Patterns](./testing.md)
Testing strategies:
- Test data builders
- Robot pattern
- Fake repositories
- ViewModel testing
- UI testing
- Flow testing

## Usage

1. Find the pattern you need
2. Copy the code snippet
3. Adapt to your use case

## Examples

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

## Related

- [Setup Guides](../setup/)
- [Concepts](../concepts/)
- [Quick Start](../QUICKSTART.md)

