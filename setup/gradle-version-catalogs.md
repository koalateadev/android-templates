# Gradle Setup & Version Catalogs

## Overview
Version catalogs provide a centralized way to manage dependency versions across your Android project, making updates easier and more consistent.

## Setup Steps

### 1. Create Version Catalog File

Create `gradle/libs.versions.toml` in your project root:

```toml
[versions]
agp = "8.5.2"
kotlin = "2.0.20"
compileSdk = "35"
minSdk = "24"
targetSdk = "35"

[libraries]
# Add your libraries here (see other setup guides)

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }

[bundles]
# Group related dependencies together
```

### 2. Update Root `build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
}
```

### 3. Update App `build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
}

android {
    namespace = "com.example.app"
    compileSdk = libs.versions.compileSdk.get().toInt()

    defaultConfig {
        applicationId = "com.example.app"
        minSdk = libs.versions.minSdk.get().toInt()
        targetSdk = libs.versions.targetSdk.get().toInt()
        versionCode = 1
        versionName = "1.0"
    }
}

dependencies {
    // Use: implementation(libs.library.name)
}
```

## Benefits

- Single source of truth for versions
- IDE autocomplete support
- Type-safe dependency references
- Shared across modules

## Notes

- Group related dependencies in bundles
- Keep the catalog organized by category
- Commit to version control

