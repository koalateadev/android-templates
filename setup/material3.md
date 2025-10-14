# Material3 Setup

## Overview
Material3 (Material Design 3) is Google's latest design system with dynamic color, updated components, and modern aesthetics.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
material3 = "1.3.0"
androidx-compose-material3-adaptive = "1.0.0"

[libraries]
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3", version.ref = "material3" }
androidx-compose-material3-window-size = { group = "androidx.compose.material3", name = "material3-window-size-class", version.ref = "material3" }
androidx-compose-material3-adaptive = { group = "androidx.compose.material3.adaptive", name = "adaptive", version.ref = "androidx-compose-material3-adaptive" }
androidx-compose-material-icons-extended = { group = "androidx.compose.material", name = "material-icons-extended" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.material3)
    implementation(libs.androidx.compose.material3.window.size)
    
    // Optional: Extended icons
    implementation(libs.androidx.compose.material.icons.extended)
    
    // Optional: Adaptive layouts
    implementation(libs.androidx.compose.material3.adaptive)
}
```

### 3. Create Material3 Theme

Create `ui/theme/Color.kt`:

```kotlin
package com.example.app.ui.theme

import androidx.compose.ui.graphics.Color

// Light theme colors
val primaryLight = Color(0xFF6750A4)
val onPrimaryLight = Color(0xFFFFFFFF)
val primaryContainerLight = Color(0xFFEADDFF)
val onPrimaryContainerLight = Color(0xFF21005D)

// Dark theme colors
val primaryDark = Color(0xFFD0BCFF)
val onPrimaryDark = Color(0xFF381E72)
val primaryContainerDark = Color(0xFF4F378B)
val onPrimaryContainerDark = Color(0xFFEADDFF)

// Add more colors as needed
```

Create `ui/theme/Theme.kt`:

```kotlin
package com.example.app.ui.theme

import android.app.Activity
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.material3.dynamicDarkColorScheme
import androidx.compose.material3.dynamicLightColorScheme
import androidx.compose.material3.lightColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.SideEffect
import androidx.compose.ui.graphics.toArgb
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalView
import androidx.core.view.WindowCompat

private val LightColorScheme = lightColorScheme(
    primary = primaryLight,
    onPrimary = onPrimaryLight,
    primaryContainer = primaryContainerLight,
    onPrimaryContainer = onPrimaryContainerLight,
    // Add more colors
)

private val DarkColorScheme = darkColorScheme(
    primary = primaryDark,
    onPrimary = onPrimaryDark,
    primaryContainer = primaryContainerDark,
    onPrimaryContainer = onPrimaryContainerDark,
    // Add more colors
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true, // Android 12+
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = colorScheme.primary.toArgb()
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = !darkTheme
        }
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

Create `ui/theme/Type.kt`:

```kotlin
package com.example.app.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val Typography = Typography(
    displayLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = 0.sp
    ),
    headlineLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 32.sp,
        lineHeight = 40.sp,
        letterSpacing = 0.sp
    ),
    bodyLarge = TextStyle(
        fontFamily = FontFamily.Default,
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.5.sp
    )
    // Add more text styles
)
```

### 4. Use Material3 Components

```kotlin
@Composable
fun Material3Example() {
    Column(modifier = Modifier.padding(16.dp)) {
        Button(onClick = { /* Click */ }) {
            Text("Material3 Button")
        }
        
        FilledTonalButton(onClick = { /* Click */ }) {
            Text("Tonal Button")
        }
        
        Card(
            modifier = Modifier.fillMaxWidth()
        ) {
            Text(
                text = "Material3 Card",
                modifier = Modifier.padding(16.dp)
            )
        }
        
        FloatingActionButton(
            onClick = { /* Click */ }
        ) {
            Icon(Icons.Default.Add, contentDescription = "Add")
        }
    }
}
```

## Key Components

- 🔘 **Buttons**: Button, FilledTonalButton, OutlinedButton, TextButton
- 📇 **Cards**: Card, ElevatedCard, OutlinedCard
- 📊 **Navigation**: NavigationBar, NavigationRail, NavigationDrawer
- 🎛️ **Input**: TextField, OutlinedTextField
- 📱 **Layout**: Scaffold, TopAppBar, BottomAppBar
- 💬 **Feedback**: Snackbar, Dialog, AlertDialog

## Dynamic Color (Android 12+)

Material3 can extract colors from the user's wallpaper:

```kotlin
AppTheme(dynamicColor = true) {
    // Your content
}
```

## Resources

- [Material3 Guidelines](https://m3.material.io/)
- [Compose Material3 Catalog](https://github.com/androidx/androidx/tree/androidx-main/compose/material3)
- [Color Tool](https://material-foundation.github.io/material-theme-builder/)

