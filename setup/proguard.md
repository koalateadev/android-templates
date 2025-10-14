# ProGuard/R8 Configuration

## Overview
R8 is Android's code shrinker and obfuscator that reduces app size and makes reverse engineering more difficult. It replaced ProGuard as the default.

## Enable R8

### Basic Configuration

In `build.gradle.kts`:

```kotlin
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
        
        debug {
            isMinifyEnabled = false
        }
    }
}
```

## ProGuard Rules

### Basic Rules (`proguard-rules.pro`)

```proguard
# Keep application class
-keep class com.example.app.MyApplication

# Keep all classes with native methods
-keepclasseswithmembernames class * {
    native <methods>;
}

# Keep custom views
-keepclasseswithmembers class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

# Keep Parcelable classes
-keepclassmembers class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# Keep Serializable classes
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}

# Keep enums
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

## Common Library Rules

### Retrofit

```proguard
# Retrofit
-keepattributes Signature
-keepattributes Exceptions
-keepattributes *Annotation*

-keep class retrofit2.** { *; }
-keepclasseswithmembers class * {
    @retrofit2.http.* <methods>;
}

-keep interface retrofit2.** { *; }
```

### OkHttp

```proguard
# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }
```

### Kotlinx Serialization

```proguard
# Kotlinx Serialization
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt

-keepclassmembers class kotlinx.serialization.json.** {
    *** Companion;
}
-keepclasseswithmembers class kotlinx.serialization.json.** {
    kotlinx.serialization.KSerializer serializer(...);
}

-keep,includedescriptorclasses class com.example.app.**$$serializer { *; }
-keepclassmembers class com.example.app.** {
    *** Companion;
}
-keepclasseswithmembers class com.example.app.** {
    kotlinx.serialization.KSerializer serializer(...);
}
```

### Gson

```proguard
# Gson
-keepattributes Signature
-keepattributes *Annotation*
-dontwarn sun.misc.**

-keep class com.google.gson.** { *; }
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer

# Keep data classes
-keep class com.example.app.data.model.** { *; }
```

### Room

```proguard
# Room
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-dontwarn androidx.room.paging.**
```

### Coroutines

```proguard
# Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
-keepclassmembernames class kotlinx.** {
    volatile <fields>;
}
```

### Hilt/Dagger

```proguard
# Hilt
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }
-keep class * extends dagger.hilt.android.lifecycle.HiltViewModel
-keep @dagger.hilt.android.lifecycle.HiltViewModel class * { *; }
```

### Jetpack Compose

```proguard
# Compose
-keep class androidx.compose.** { *; }
-dontwarn androidx.compose.**

# Keep composable functions
-keep @androidx.compose.runtime.Composable class * { *; }
-keep @androidx.compose.runtime.Composable interface * { *; }
```

### DataStore

```proguard
# DataStore
-keep class * extends com.google.protobuf.GeneratedMessageLite { *; }
```

## Data Classes and Models

### Keep Data Classes

```proguard
# Keep all data classes
-keep class com.example.app.data.** { *; }
-keep class com.example.app.domain.model.** { *; }

# Or specific classes
-keep class com.example.app.data.User { *; }
-keep class com.example.app.data.Post { *; }
```

### Keep with Annotations

```proguard
# Keep classes with @Keep annotation
-keep @androidx.annotation.Keep class * { *; }
-keepclassmembers class * {
    @androidx.annotation.Keep *;
}

# Keep classes with @Serializable
-keep @kotlinx.serialization.Serializable class * { *; }
```

## Debugging ProGuard Issues

### Keep Line Numbers

```proguard
# Keep line numbers for debugging
-keepattributes SourceFile,LineNumberTable

# Rename source file to "SourceFile"
-renamesourcefileattribute SourceFile
```

### Print Configuration

```proguard
# Print configuration to build/outputs/mapping/
-printconfiguration build/outputs/mapping/configuration.txt

# Print seeds (classes/members kept)
-printseeds build/outputs/mapping/seeds.txt

# Print usage (unused code removed)
-printusage build/outputs/mapping/usage.txt
```

### Warnings

```proguard
# Don't warn about missing classes
-dontwarn org.conscrypt.**
-dontwarn org.bouncycastle.**
-dontwarn org.openjsse.**

# Fail build on warnings (strict mode)
-dontwarn

# Or ignore warnings
-ignorewarnings
```

## Mapping Files

### Deobfuscate Stack Traces

ProGuard/R8 generates a `mapping.txt` file in `app/build/outputs/mapping/release/`.

Use this file to deobfuscate crash reports:

```bash
# Using retrace
retrace.bat mapping.txt stacktrace.txt
```

### Keep Mapping Files

Store mapping files for each release to deobfuscate future crash reports.

In your CI/CD:
```bash
# Upload mapping file
cp app/build/outputs/mapping/release/mapping.txt artifacts/mapping-${VERSION}.txt
```

## Optimization

### Aggressive Optimization

```proguard
# Optimize as much as possible
-optimizationpasses 5
-allowaccessmodification
-dontpreverify

# Merge classes and interfaces
-mergeinterfacesaggressively

# Optimize method calls
-optimizations !code/simplification/arithmetic,!code/simplification/cast,!field/*,!class/merging/*
```

### Conservative Optimization

```proguard
# Use default optimizations
-dontoptimize

# Or use standard optimizations
-optimizations !code/simplification/arithmetic
```

## Special Cases

### Keep JavaScript Interface

```proguard
# Keep WebView JavaScript interfaces
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}
```

### Keep Reflection-based Code

```proguard
# Keep classes used via reflection
-keep class com.example.app.reflection.** { *; }

# Keep methods accessed via reflection
-keepclassmembers class com.example.app.** {
    public void set*(***);
    public *** get*();
}
```

### Keep BuildConfig

```proguard
# Keep BuildConfig
-keep class com.example.app.BuildConfig { *; }
```

## Testing ProGuard

### Enable for Debug

```kotlin
android {
    buildTypes {
        debug {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

Then test your app thoroughly to catch any ProGuard issues early.

## Common Issues

### Missing Classes

**Problem:** ClassNotFoundException at runtime

**Solution:** Add keep rules for the missing class
```proguard
-keep class com.example.MissingClass { *; }
```

### Reflection Errors

**Problem:** Methods/fields not found via reflection

**Solution:** Keep classes/members used with reflection
```proguard
-keepclassmembers class com.example.ReflectedClass {
    public <fields>;
    public <methods>;
}
```

### Serialization Issues

**Problem:** Serialization fails for obfuscated classes

**Solution:** Keep serializable classes and their members
```proguard
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    !static !transient <fields>;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
}
```

### Native Method Issues

**Problem:** Native methods renamed/removed

**Solution:** Keep native methods
```proguard
-keepclasseswithmembernames,includedescriptorclasses class * {
    native <methods>;
}
```

## R8 Full Mode

### Enable Full Mode

In `gradle.properties`:
```properties
android.enableR8.fullMode=true
```

R8 full mode is more aggressive but may require additional rules.

### Compatibility Mode (Default)

R8 compatibility mode tries to emulate ProGuard behavior:
```properties
android.enableR8.fullMode=false
```

## Best Practices

1. ✅ Test with ProGuard enabled early and often
2. ✅ Keep mapping files for all releases
3. ✅ Use consumer ProGuard files in libraries
4. ✅ Don't keep more than necessary
5. ✅ Use @Keep annotations for public APIs
6. ✅ Test thoroughly before release
7. ✅ Monitor crash reports for ProGuard issues
8. ✅ Document custom rules
9. ✅ Use printconfiguration for debugging
10. ✅ Update rules when adding new libraries

## Consumer ProGuard Rules

### For Library Modules

In library `build.gradle.kts`:
```kotlin
android {
    defaultConfig {
        consumerProguardFiles("consumer-rules.pro")
    }
}
```

`consumer-rules.pro`:
```proguard
# Rules automatically applied to apps that use this library
-keep class com.example.library.PublicApi { *; }
```

## Template proguard-rules.pro

```proguard
# ===================================
# Project-specific ProGuard rules
# ===================================

# Uncomment to debug ProGuard
#-printconfiguration build/outputs/mapping/configuration.txt
#-printseeds build/outputs/mapping/seeds.txt
#-printusage build/outputs/mapping/usage.txt

# Keep line numbers for crash reports
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# ===================================
# Data Models
# ===================================
-keep class com.example.app.data.model.** { *; }
-keep class com.example.app.domain.model.** { *; }

# ===================================
# Kotlin
# ===================================
-keep class kotlin.** { *; }
-keep class kotlin.Metadata { *; }
-dontwarn kotlin.**
-keepclassmembers class **$WhenMappings {
    <fields>;
}
-keepclassmembers class kotlin.Metadata {
    public <methods>;
}

# ===================================
# Kotlinx Coroutines
# ===================================
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
-keepclassmembernames class kotlinx.** {
    volatile <fields>;
}

# ===================================
# Add your own rules below
# ===================================
```

## Resources

- [R8 Documentation](https://developer.android.com/studio/build/shrink-code)
- [ProGuard Manual](https://www.guardsquare.com/manual/home)
- [Common ProGuard Issues](https://developer.android.com/studio/build/shrink-code#troubleshoot)

