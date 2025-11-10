# Quick Start Guide

Complete setup for a modern Android application with recommended libraries and architecture.

## Prerequisites

- Android Studio (latest stable version)
- JDK 17 or higher
- Basic knowledge of Kotlin and Android development

## Project Setup Steps

### 1. Create New Project

1. Open Android Studio
2. File → New → New Project
3. Select "Empty Activity" (Compose)
4. Configure project:
   - Name: MyApp
   - Package: com.example.myapp
   - Minimum SDK: 24
   - Build configuration language: Kotlin DSL

### 2. Configure Version Catalog

Create `gradle/libs.versions.toml`:

```toml
[versions]
agp = "8.5.2"
kotlin = "2.0.20"
compileSdk = "35"
minSdk = "24"
targetSdk = "35"

# AndroidX
androidx-core-ktx = "1.15.0"
androidx-lifecycle = "2.8.6"
androidx-activity-compose = "1.9.2"
androidx-navigation = "2.8.3"

# Compose
compose-bom = "2024.09.03"

# Kotlin
coroutines = "1.9.0"
kotlinx-serialization = "1.7.3"

# Dependency Injection
hilt = "2.52"
hilt-navigation-compose = "1.2.0"

# Network
okhttp = "4.12.0"
retrofit = "2.11.0"

# Database
room = "2.6.1"
ksp = "2.0.20-1.0.25"

# Storage
datastore = "1.1.1"

# Image Loading
coil = "2.7.0"

# Logging
timber = "5.0.1"

# Testing
junit = "4.13.2"
junit5 = "5.11.3"
mockk = "1.13.13"
turbine = "1.1.0"
truth = "1.4.4"

# Debug
leakcanary = "2.14"

[libraries]
# AndroidX Core
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "androidx-core-ktx" }

# Lifecycle
androidx-lifecycle-viewmodel-ktx = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-ktx", version.ref = "androidx-lifecycle" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "androidx-lifecycle" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "androidx-lifecycle" }

# Compose
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-compose-material-icons-extended = { group = "androidx.compose.material", name = "material-icons-extended" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "androidx-activity-compose" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "androidx-navigation" }

# Coroutines
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }

# Serialization
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "kotlinx-serialization" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hilt-navigation-compose" }

# Network
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-kotlinx-serialization = { group = "com.squareup.retrofit2", name = "converter-kotlinx-serialization", version.ref = "retrofit" }

# Room
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# DataStore
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }

# Image Loading
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

# Logging
timber = { group = "com.jakewharton.timber", name = "timber", version.ref = "timber" }

# Testing
junit = { group = "junit", name = "junit", version.ref = "junit" }
junit-jupiter-api = { group = "org.junit.jupiter", name = "junit-jupiter-api", version.ref = "junit5" }
junit-jupiter-engine = { group = "org.junit.jupiter", name = "junit-jupiter-engine", version.ref = "junit5" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
turbine = { group = "app.cash.turbine", name = "turbine", version.ref = "turbine" }
truth = { group = "com.google.truth", name = "truth", version.ref = "truth" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }
androidx-compose-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-compose-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }

# Debug
leakcanary-android = { group = "com.squareup.leakcanary", name = "leakcanary-android", version.ref = "leakcanary" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }

[bundles]
compose = [
    "androidx-compose-ui",
    "androidx-compose-ui-tooling-preview",
    "androidx-compose-material3",
    "androidx-activity-compose"
]
lifecycle = [
    "androidx-lifecycle-viewmodel-ktx",
    "androidx-lifecycle-runtime-ktx",
    "androidx-lifecycle-runtime-compose"
]
coroutines = [
    "kotlinx-coroutines-core",
    "kotlinx-coroutines-android"
]
network = [
    "okhttp",
    "okhttp-logging-interceptor",
    "retrofit",
    "retrofit-converter-kotlinx-serialization"
]
room = [
    "androidx-room-runtime",
    "androidx-room-ktx"
]
testing = [
    "junit-jupiter-api",
    "mockk",
    "truth",
    "turbine",
    "kotlinx-coroutines-test"
]
```

### 3. Configure Root build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
}
```

### 4. Configure App build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.example.myapp"
    compileSdk = libs.versions.compileSdk.get().toInt()

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = libs.versions.minSdk.get().toInt()
        targetSdk = libs.versions.targetSdk.get().toInt()
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        
        buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")
    }

    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-DEBUG"
        }
        
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    
    kotlinOptions {
        jvmTarget = "17"
    }
    
    buildFeatures {
        compose = true
        buildConfig = true
    }
    
    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    // Core
    implementation(libs.androidx.core.ktx)
    
    // Compose
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.bundles.compose)
    implementation(libs.androidx.navigation.compose)
    implementation(libs.androidx.compose.material.icons.extended)
    
    // Lifecycle
    implementation(libs.bundles.lifecycle)
    
    // Coroutines
    implementation(libs.bundles.coroutines)
    
    // Serialization
    implementation(libs.kotlinx.serialization.json)
    
    // Hilt
    implementation(libs.hilt.android)
    implementation(libs.hilt.navigation.compose)
    ksp(libs.hilt.compiler)
    
    // Network
    implementation(libs.bundles.network)
    
    // Room
    implementation(libs.bundles.room)
    ksp(libs.androidx.room.compiler)
    
    // DataStore
    implementation(libs.androidx.datastore.preferences)
    
    // Image Loading
    implementation(libs.coil.compose)
    
    // Logging
    implementation(libs.timber)
    
    // Debug
    debugImplementation(libs.androidx.compose.ui.tooling)
    debugImplementation(libs.androidx.compose.ui.test.manifest)
    debugImplementation(libs.leakcanary.android)
    
    // Testing
    testImplementation(libs.bundles.testing)
    testRuntimeOnly(libs.junit.jupiter.engine)
    
    // Android Testing
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
}
```

### 5. Create Application Class

```kotlin
package com.example.myapp

import android.app.Application
import dagger.hilt.android.HiltAndroidApp
import timber.log.Timber

@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }
}
```

Update `AndroidManifest.xml`:
```xml
<application
    android:name=".MyApplication"
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/Theme.MyApp">
    
    <activity
        android:name=".MainActivity"
        android:exported="true"
        android:theme="@style/Theme.MyApp">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

## Project Structure

```
app/src/main/kotlin/com/example/myapp/
├── MyApplication.kt
├── MainActivity.kt
├── data/
│   ├── local/
│   │   ├── database/
│   │   │   ├── AppDatabase.kt
│   │   │   └── dao/
│   │   └── preferences/
│   │       └── PreferencesManager.kt
│   ├── remote/
│   │   ├── ApiService.kt
│   │   └── dto/
│   └── repository/
│       └── UserRepository.kt
├── di/
│   ├── AppModule.kt
│   ├── DatabaseModule.kt
│   └── NetworkModule.kt
├── domain/
│   └── model/
│       └── User.kt
└── ui/
    ├── theme/
    │   ├── Color.kt
    │   ├── Theme.kt
    │   └── Type.kt
    ├── navigation/
    │   └── Navigation.kt
    └── screens/
        ├── home/
        │   ├── HomeScreen.kt
        │   └── HomeViewModel.kt
        └── details/
            ├── DetailsScreen.kt
            └── DetailsViewModel.kt
```

## Sample Implementation

### 1. Network Module

```kotlin
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
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG) {
                    HttpLoggingInterceptor.Level.BODY
                } else {
                    HttpLoggingInterceptor.Level.NONE
                }
            })
            .build()
    }
    
    @Provides
    @Singleton
    fun provideRetrofit(
        okHttpClient: OkHttpClient,
        json: Json
    ): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
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

### 2. ViewModel Example

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    init {
        loadData()
    }
    
    fun loadData() {
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
}

sealed interface UiState {
    data object Loading : UiState
    data class Success(val users: List<User>) : UiState
    data class Error(val message: String) : UiState
}
```

### 3. Composable Screen

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    Scaffold(
        topBar = {
            TopAppBar(title = { Text("Home") })
        }
    ) { paddingValues ->
        when (uiState) {
            is UiState.Loading -> {
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
            
            is UiState.Success -> {
                val users = (uiState as UiState.Success).users
                LazyColumn(
                    modifier = Modifier.padding(paddingValues)
                ) {
                    items(users) { user ->
                        UserItem(user)
                    }
                }
            }
            
            is UiState.Error -> {
                val message = (uiState as UiState.Error).message
                Box(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(paddingValues),
                    contentAlignment = Alignment.Center
                ) {
                    Text(message)
                }
            }
        }
    }
}
```

## Additional Configuration

1. Review individual setup guides for detailed configuration
2. Set up Git and `.gitignore`
3. Configure CI/CD pipeline
4. Add signing configuration
5. Configure ProGuard rules
6. Set up code quality tools (lint, detekt)

## Additional Resources

- [Android Architecture Guidelines](https://developer.android.com/topic/architecture)
- [Compose Guidelines](https://developer.android.com/jetpack/compose/guidelines)
- [Kotlin Style Guide](https://developer.android.com/kotlin/style-guide)

## Troubleshooting

### Common Issues

**Build fails with "Duplicate class" error:**
- Check for conflicting dependencies
- Use `./gradlew app:dependencies` to see dependency tree

**Hilt compilation errors:**
- Ensure `@HiltAndroidApp` is on Application class
- Check all `@AndroidEntryPoint` annotations are correct
- Clean and rebuild project

**Compose preview not working:**
- Ensure `@Preview` annotation is present
- Check if `debugImplementation(libs.androidx.compose.ui.tooling)` is added

**Room compilation errors:**
- Verify KSP plugin is applied
- Check entity and DAO annotations
- Ensure all queries are valid SQL

## Next Steps

Refer to individual setup guides in `setup/` for detailed library configuration.

