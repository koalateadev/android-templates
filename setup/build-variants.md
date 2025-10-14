# Build Variants & Flavors Setup

## Overview
Build variants allow you to create different versions of your app from a single project, useful for development, staging, and production environments.

## Basic Configuration

### 1. Configure Build Types

In `build.gradle.kts`:

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-DEBUG"
            isDebuggable = true
            isMinifyEnabled = false
        }
        
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
        
        create("staging") {
            initWith(getByName("debug"))
            applicationIdSuffix = ".staging"
            versionNameSuffix = "-STAGING"
            isDebuggable = true
        }
    }
}
```

### 2. Configure Product Flavors

```kotlin
android {
    flavorDimensions += "environment"
    
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            
            buildConfigField("String", "API_BASE_URL", "\"https://dev-api.example.com\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "true")
            resValue("string", "app_name", "MyApp Dev")
        }
        
        create("staging") {
            dimension = "environment"
            applicationIdSuffix = ".staging"
            versionNameSuffix = "-staging"
            
            buildConfigField("String", "API_BASE_URL", "\"https://staging-api.example.com\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "true")
            resValue("string", "app_name", "MyApp Staging")
        }
        
        create("prod") {
            dimension = "environment"
            
            buildConfigField("String", "API_BASE_URL", "\"https://api.example.com\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "false")
            resValue("string", "app_name", "MyApp")
        }
    }
}
```

## BuildConfig Fields

### Define Fields

```kotlin
android {
    defaultConfig {
        buildConfigField("String", "API_KEY", "\"default_key\"")
        buildConfigField("int", "API_VERSION", "1")
        buildConfigField("boolean", "FEATURE_ENABLED", "false")
    }
    
    buildTypes {
        release {
            buildConfigField("String", "API_KEY", "\"production_key\"")
            buildConfigField("boolean", "FEATURE_ENABLED", "true")
        }
    }
}
```

### Use in Code

```kotlin
object ApiConfig {
    val baseUrl: String = BuildConfig.API_BASE_URL
    val apiKey: String = BuildConfig.API_KEY
    val loggingEnabled: Boolean = BuildConfig.ENABLE_LOGGING
}

// In module setup
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideLoggingInterceptor(): HttpLoggingInterceptor {
        return HttpLoggingInterceptor().apply {
            level = if (BuildConfig.ENABLE_LOGGING) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }
    }
}
```

## Resource Values

### Define Variant-Specific Resources

```kotlin
android {
    productFlavors {
        create("free") {
            resValue("string", "app_name", "MyApp Free")
            resValue("color", "primary_color", "#FF0000")
            resValue("bool", "show_ads", "true")
        }
        
        create("premium") {
            resValue("string", "app_name", "MyApp Premium")
            resValue("color", "primary_color", "#0000FF")
            resValue("bool", "show_ads", "false")
        }
    }
}
```

### Use in XML

```xml
<string name="welcome_message">@string/app_name</string>
<color name="brand_color">@color/primary_color</color>
<bool name="ads_enabled">@bool/show_ads</bool>
```

### Use in Code

```kotlin
val appName = getString(R.string.app_name)
val showAds = resources.getBoolean(R.bool.show_ads)
```

## Multiple Dimensions

### Configure Multiple Dimensions

```kotlin
android {
    flavorDimensions += listOf("environment", "api")
    
    productFlavors {
        // Environment dimension
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
        }
        
        create("prod") {
            dimension = "environment"
        }
        
        // API dimension
        create("free") {
            dimension = "api"
            buildConfigField("boolean", "IS_PREMIUM", "false")
        }
        
        create("premium") {
            dimension = "api"
            buildConfigField("boolean", "IS_PREMIUM", "true")
        }
    }
}
```

This creates variants:
- devFreeDebug
- devFreeRelease
- devPremiumDebug
- devPremiumRelease
- prodFreeDebug
- prodFreeRelease
- prodPremiumDebug
- prodPremiumRelease

## Variant-Specific Source Sets

### Directory Structure

```
app/src/
├── main/                 # Common code
├── debug/                # Debug build type
├── release/              # Release build type
├── dev/                  # Dev flavor
├── staging/              # Staging flavor
├── prod/                 # Prod flavor
├── devDebug/             # Specific variant
└── prodRelease/          # Specific variant
```

### Variant-Specific Code

`app/src/dev/kotlin/com/example/app/config/AppConfig.kt`:
```kotlin
object AppConfig {
    const val BASE_URL = "https://dev-api.example.com"
    const val DEBUG_MODE = true
}
```

`app/src/prod/kotlin/com/example/app/config/AppConfig.kt`:
```kotlin
object AppConfig {
    const val BASE_URL = "https://api.example.com"
    const val DEBUG_MODE = false
}
```

### Variant-Specific Resources

`app/src/dev/res/values/strings.xml`:
```xml
<resources>
    <string name="app_name">MyApp Dev</string>
</resources>
```

`app/src/prod/res/values/strings.xml`:
```xml
<resources>
    <string name="app_name">MyApp</string>
</resources>
```

## Different App Icons

### Configure Icons Per Flavor

```
app/src/
├── dev/res/
│   ├── mipmap-hdpi/ic_launcher.png
│   ├── mipmap-mdpi/ic_launcher.png
│   └── ...
├── staging/res/
│   ├── mipmap-hdpi/ic_launcher.png
│   └── ...
└── prod/res/
    ├── mipmap-hdpi/ic_launcher.png
    └── ...
```

Each flavor can have its own app icon, launcher, and resources.

## Signing Configs

### Configure Signing

```kotlin
android {
    signingConfigs {
        create("release") {
            storeFile = file("../keystore/release.jks")
            storePassword = System.getenv("RELEASE_STORE_PASSWORD")
            keyAlias = System.getenv("RELEASE_KEY_ALIAS")
            keyPassword = System.getenv("RELEASE_KEY_PASSWORD")
        }
        
        create("staging") {
            storeFile = file("../keystore/staging.jks")
            storePassword = "staging_password"
            keyAlias = "staging"
            keyPassword = "staging_password"
        }
    }
    
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
        
        getByName("staging") {
            signingConfig = signingConfigs.getByName("staging")
        }
    }
}
```

### Use gradle.properties

`gradle.properties`:
```properties
RELEASE_STORE_FILE=../keystore/release.jks
RELEASE_STORE_PASSWORD=your_password
RELEASE_KEY_ALIAS=release
RELEASE_KEY_PASSWORD=your_password
```

`build.gradle.kts`:
```kotlin
val releaseStoreFile: String by project
val releaseStorePassword: String by project
val releaseKeyAlias: String by project
val releaseKeyPassword: String by project

android {
    signingConfigs {
        create("release") {
            storeFile = file(releaseStoreFile)
            storePassword = releaseStorePassword
            keyAlias = releaseKeyAlias
            keyPassword = releaseKeyPassword
        }
    }
}
```

## Variant Filters

### Filter Out Unwanted Variants

```kotlin
android {
    variantFilter {
        if (buildType.name == "release" && flavors[0].name == "dev") {
            ignore = true // Skip devRelease variant
        }
    }
}
```

## Dependencies Per Variant

### Variant-Specific Dependencies

```kotlin
dependencies {
    // All variants
    implementation(libs.androidx.core.ktx)
    
    // Debug only
    debugImplementation(libs.leakcanary)
    debugImplementation(libs.chucker)
    
    // Flavor-specific
    "devImplementation"(libs.mock.api)
    "prodImplementation"(libs.real.api)
    
    // Build type + flavor
    "devDebugImplementation"(libs.debug.tools)
}
```

## Manifest Placeholders

### Configure Placeholders

```kotlin
android {
    defaultConfig {
        manifestPlaceholders["appLabel"] = "MyApp"
        manifestPlaceholders["hostName"] = "www.example.com"
    }
    
    productFlavors {
        create("dev") {
            manifestPlaceholders["appLabel"] = "MyApp Dev"
            manifestPlaceholders["hostName"] = "dev.example.com"
        }
    }
}
```

### Use in AndroidManifest.xml

```xml
<application
    android:label="${appLabel}"
    ...>
    
    <activity android:name=".MainActivity">
        <intent-filter>
            <data
                android:scheme="https"
                android:host="${hostName}" />
        </intent-filter>
    </activity>
</application>
```

## Testing Variants

### Run Tests for Specific Variant

```bash
# Run unit tests
./gradlew testDevDebugUnitTest
./gradlew testProdReleaseUnitTest

# Run instrumented tests
./gradlew connectedDevDebugAndroidTest
```

### Variant-Specific Tests

```
app/src/
├── test/              # Shared unit tests
├── testDev/           # Dev unit tests
├── testProd/          # Prod unit tests
├── androidTest/       # Shared instrumented tests
├── androidTestDev/    # Dev instrumented tests
└── androidTestProd/   # Prod instrumented tests
```

## Building Variants

### Build Specific Variant

```bash
# Build debug APK
./gradlew assembleDevDebug
./gradlew assembleProdDebug

# Build release APK
./gradlew assembleDevRelease
./gradlew assembleProdRelease

# Build all variants
./gradlew assemble

# Build AAB (Android App Bundle)
./gradlew bundleDevRelease
./gradlew bundleProdRelease
```

## Version Management

### Dynamic Version Codes

```kotlin
android {
    flavorDimensions += "environment"
    
    productFlavors {
        create("dev") {
            dimension = "environment"
            versionCode = 1000
            versionNameSuffix = "-dev"
        }
        
        create("staging") {
            dimension = "environment"
            versionCode = 2000
            versionNameSuffix = "-staging"
        }
        
        create("prod") {
            dimension = "environment"
            versionCode = 3000
        }
    }
    
    // Or calculate dynamically
    applicationVariants.all {
        val variant = this
        variant.outputs.all {
            val output = this as com.android.build.gradle.internal.api.BaseVariantOutputImpl
            val versionCode = variant.versionCode
            val flavorCode = when (variant.flavorName) {
                "dev" -> 1000000
                "staging" -> 2000000
                "prod" -> 3000000
                else -> 0
            }
            output.versionCodeOverride = flavorCode + versionCode
        }
    }
}
```

## Best Practices

1. ✅ Use flavors for environment differences (dev/staging/prod)
2. ✅ Use build types for debug/release configurations
3. ✅ Keep variant-specific code minimal
4. ✅ Use BuildConfig for configuration values
5. ✅ Store secrets in environment variables or secure storage
6. ✅ Test all important variants
7. ✅ Use different app icons for easy identification
8. ✅ Use different package names to install multiple variants
9. ✅ Filter out unnecessary variants
10. ✅ Automate builds with CI/CD

## Common Use Cases

### Free vs Premium

```kotlin
productFlavors {
    create("free") {
        dimension = "version"
        buildConfigField("boolean", "SHOW_ADS", "true")
        buildConfigField("boolean", "PREMIUM_FEATURES", "false")
    }
    
    create("premium") {
        dimension = "version"
        buildConfigField("boolean", "SHOW_ADS", "false")
        buildConfigField("boolean", "PREMIUM_FEATURES", "true")
    }
}
```

### Multiple Backends

```kotlin
productFlavors {
    create("mock") {
        buildConfigField("String", "API_URL", "\"http://localhost:8080\"")
    }
    
    create("development") {
        buildConfigField("String", "API_URL", "\"https://dev-api.example.com\"")
    }
    
    create("production") {
        buildConfigField("String", "API_URL", "\"https://api.example.com\"")
    }
}
```

## Resources

- [Configure Build Variants](https://developer.android.com/studio/build/build-variants)
- [Product Flavors](https://developer.android.com/studio/build/build-variants#product-flavors)
- [Build Types](https://developer.android.com/studio/build/build-variants#build-types)

