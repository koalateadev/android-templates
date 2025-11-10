# Jetpack Compose Setup

## Overview
Jetpack Compose is Android's modern declarative UI toolkit that simplifies and accelerates UI development.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
compose-bom = "2024.09.03"
compose-compiler = "1.5.14"
androidx-activity-compose = "1.9.2"

[libraries]
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-compose-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "androidx-activity-compose" }

[bundles]
compose = ["androidx-compose-ui", "androidx-compose-ui-tooling-preview", "androidx-activity-compose"]
```

### 2. Enable Compose in `build.gradle.kts` (Module Level)

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
}

android {
    buildFeatures {
        compose = true
    }
}

dependencies {
    // BOM manages versions
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.bundles.compose)
    
    // Debug tools
    debugImplementation(libs.androidx.compose.ui.tooling)
    debugImplementation(libs.androidx.compose.ui.test.manifest)
    
    // Testing
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
}
```

### 3. Create a Compose Activity

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            AppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    MainScreen()
                }
            }
        }
    }
}
```

### 4. Basic Composable Example

```kotlin
@Composable
fun MainScreen() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Hello Compose!",
            style = MaterialTheme.typography.headlineMedium
        )
    }
}

@Preview(showBackground = true)
@Composable
fun MainScreenPreview() {
    AppTheme {
        MainScreen()
    }
}
```

## Features

- Declarative UI
- Automatic recomposition
- Type-safe builders
- Live preview support
- Built-in testing APIs

## Best Practices

1. **State Management**: Use `remember`, `rememberSaveable`, and `ViewModel`
2. **Side Effects**: Use `LaunchedEffect`, `DisposableEffect` appropriately
3. **Performance**: Use `derivedStateOf` and `key()` when needed
4. **Reusability**: Create small, focused composables
5. **Preview**: Add `@Preview` annotations for quick iteration

## Resources

- [Official Documentation](https://developer.android.com/jetpack/compose)
- [Compose Samples](https://github.com/android/compose-samples)

