# Material3 Design System Concepts

## Overview

Material Design 3 (Material You) is Google's design system that emphasizes personalization, accessibility, and adaptivity.

## Design Tokens

### What are Design Tokens?

Design tokens are the foundational elements of the design system:

```
Design Tokens
    ├── Color
    ├── Typography
    ├── Shape
    ├── Motion
    └── Spacing
```

### Color Tokens

```kotlin
// Material3 has 26 color tokens organized in pairs
colorScheme.primary              // Main brand color
colorScheme.onPrimary            // Content on primary
colorScheme.primaryContainer     // Less prominent primary
colorScheme.onPrimaryContainer   // Content on container

colorScheme.secondary            // Accent color
colorScheme.onSecondary
colorScheme.secondaryContainer
colorScheme.onSecondaryContainer

colorScheme.tertiary             // Third accent
colorScheme.onTertiary
colorScheme.tertiaryContainer
colorScheme.onTertiaryContainer

colorScheme.error                // Error states
colorScheme.onError
colorScheme.errorContainer
colorScheme.onErrorContainer

colorScheme.background           // App background
colorScheme.onBackground

colorScheme.surface              // Component backgrounds
colorScheme.onSurface
colorScheme.surfaceVariant       // Less prominent surfaces
colorScheme.onSurfaceVariant

colorScheme.outline              // Borders
colorScheme.outlineVariant       // Subtle borders
```

### Color Roles

| Role | Usage |
|------|-------|
| primary | FABs, prominent buttons, active states |
| secondary | Less prominent buttons, selection controls |
| tertiary | Contrasting accents, special content |
| error | Errors, destructive actions |
| surface | Card backgrounds, bottom sheets |
| background | Screen backgrounds |
| outline | Borders, dividers |

## Dynamic Color (Material You)

### How Dynamic Color Works

```
User's Wallpaper
    ↓
System extracts colors
    ↓
Generates color scheme
    ↓
App applies dynamic colors
```

### Implementation

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        // Dynamic color (Android 12+)
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        // Static dark theme
        darkTheme -> DarkColorScheme
        // Static light theme
        else -> LightColorScheme
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        content = content
    )
}
```

### Static vs Dynamic

| Feature | Static Colors | Dynamic Colors |
|---------|--------------|----------------|
| Personalization | ❌ Same for all | ✅ From wallpaper |
| Consistency | ✅ Predictable | ⚠️ Varies per device |
| Branding | ✅ Brand colors | ⚠️ May differ |
| User Experience | Good | ✅ Personalized |
| Android Version | All | 12+ only |

## Typography Scale

### Typography Roles

```kotlin
MaterialTheme.typography.displayLarge    // 57sp - Hero text
MaterialTheme.typography.displayMedium   // 45sp - Large headlines
MaterialTheme.typography.displaySmall    // 36sp - Headlines

MaterialTheme.typography.headlineLarge   // 32sp - Section headers
MaterialTheme.typography.headlineMedium  // 28sp - Card titles
MaterialTheme.typography.headlineSmall   // 24sp - List headers

MaterialTheme.typography.titleLarge      // 22sp - Toolbar titles
MaterialTheme.typography.titleMedium     // 16sp - Card subtitles
MaterialTheme.typography.titleSmall      // 14sp - Overlines

MaterialTheme.typography.bodyLarge       // 16sp - Main content
MaterialTheme.typography.bodyMedium      // 14sp - Secondary text
MaterialTheme.typography.bodySmall       // 12sp - Captions

MaterialTheme.typography.labelLarge      // 14sp - Buttons
MaterialTheme.typography.labelMedium     // 12sp - Chips
MaterialTheme.typography.labelSmall      // 11sp - Small labels
```

### Typography Usage

```kotlin
@Composable
fun TypographyExample() {
    Column {
        Text(
            "Page Title",
            style = MaterialTheme.typography.displayMedium
        )
        
        Text(
            "Section Header",
            style = MaterialTheme.typography.headlineMedium
        )
        
        Text(
            "This is body text that explains the content.",
            style = MaterialTheme.typography.bodyLarge
        )
        
        Button(onClick = {}) {
            Text("Button")  // Automatically uses labelLarge
        }
    }
}
```

## Shape System

### Shape Scale

```kotlin
// Small components (chips, buttons)
MaterialTheme.shapes.small           // 8dp corner radius

// Medium components (cards, dialogs)
MaterialTheme.shapes.medium          // 12dp corner radius

// Large components (bottom sheets)
MaterialTheme.shapes.large           // 16dp corner radius

// Extra large (large cards)
MaterialTheme.shapes.extraLarge      // 28dp corner radius
```

### Custom Shapes

```kotlin
val customShapes = Shapes(
    small = RoundedCornerShape(4.dp),
    medium = RoundedCornerShape(8.dp),
    large = RoundedCornerShape(16.dp),
    extraLarge = RoundedCornerShape(32.dp)
)

MaterialTheme(
    shapes = customShapes,
    content = content
)
```

## Elevation

### Elevation Levels

Material3 uses elevation for hierarchy:

```
Level 0 (0dp)   - Background
Level 1 (1dp)   - Cards
Level 2 (3dp)   - FAB at rest
Level 3 (6dp)   - FAB raised
Level 4 (8dp)   - Navigation drawer
Level 5 (12dp)  - Modal bottom sheet
```

### Surface Tonal Elevation

```kotlin
Surface(
    tonalElevation = 1.dp,  // Tints surface color
    shadowElevation = 0.dp  // Actual shadow
) {
    Text("Elevated Surface")
}

// tonalElevation changes color slightly
// shadowElevation adds shadow
```

## Motion & Transitions

### Motion Principles

1. **Informative**: Guides attention
2. **Focused**: Avoids distractions
3. **Expressive**: Adds personality

### Duration Guidelines

```kotlin
// Simple transitions: 100-200ms
val fastDuration = 150

// Complex transitions: 200-300ms
val mediumDuration = 250

// Elaborate transitions: 300-500ms
val slowDuration = 400

// Use appropriate easing
val easing = FastOutSlowInEasing
```

## Adaptive Layouts

### Window Size Classes

```kotlin
enum class WindowSizeClass {
    Compact,   // Phone portrait (< 600dp)
    Medium,    // Tablet portrait, phone landscape (600-840dp)
    Expanded   // Tablet landscape (> 840dp)
}

@Composable
fun AdaptiveLayout() {
    val windowSizeClass = calculateWindowSizeClass()
    
    when (windowSizeClass) {
        WindowSizeClass.Compact -> {
            // Single pane, bottom navigation
            CompactLayout()
        }
        WindowSizeClass.Medium -> {
            // Navigation rail
            MediumLayout()
        }
        WindowSizeClass.Expanded -> {
            // Navigation drawer, two panes
            ExpandedLayout()
        }
    }
}
```

### Responsive Navigation

```
Compact (Phone)          → Bottom Navigation
Medium (Small Tablet)    → Navigation Rail
Expanded (Large Tablet)  → Navigation Drawer
```

## Component Anatomy

### Button Structure

```
Button (Material3)
    ├── Container (shape, color, elevation)
    ├── State Layer (hover, pressed, focused)
    ├── Icon (optional leading icon)
    ├── Label (text)
    └── Ripple (interaction feedback)
```

### Card Structure

```
Card
    ├── Container
    │   ├── Surface color
    │   ├── Shape
    │   └── Elevation
    ├── Content padding
    └── Inner content
```

## Accessibility in Material3

### Built-in Accessibility

Material3 components include:
- ✅ Minimum touch targets (48dp)
- ✅ Sufficient color contrast
- ✅ Screen reader support
- ✅ Keyboard navigation
- ✅ Focus indicators
- ✅ State announcements

### Color Contrast

```kotlin
// Material3 color pairs ensure 4.5:1 contrast
primary + onPrimary             // ✅ Accessible
surface + onSurface             // ✅ Accessible
primaryContainer + onPrimaryContainer  // ✅ Accessible

// Don't mix incompatible pairs
primary + onSurface             // ❌ May not be accessible
```

## State Layers

### Component States

```
Component States:
    ├── Enabled
    ├── Disabled
    ├── Hovered (desktop/large screen)
    ├── Focused (keyboard navigation)
    ├── Pressed (touch)
    ├── Dragged
    └── Selected
```

### State Layer Opacity

```kotlin
// Material3 automatically applies state layers
Button(onClick = {}) {  // Container + State layers
    Text("Button")
}

// States add opacity overlay:
// Hover:   8% opacity
// Focus:   12% opacity
// Press:   12% opacity
// Drag:    16% opacity
```

## Material Theme Builder

### Generate Custom Theme

1. Go to [Material Theme Builder](https://material-foundation.github.io/material-theme-builder/)
2. Choose source color or upload image
3. Customize colors
4. Export for Android (Compose)
5. Copy generated code to your app

### Theme Structure

```kotlin
// Generated theme includes:
// - Color.kt (color palette)
// - Theme.kt (MaterialTheme setup)
// - Type.kt (typography scale)

// Light and dark variants automatically generated
```

## Best Practices

- Use dynamic color when appropriate
- Provide static fallback (pre-Android 12)
- Use semantic color tokens
- Follow typography scale
- Use proper elevation levels
- Test in light and dark modes
- Respect user preferences
- Use Material components
- Design for different window sizes

## Common Mistakes

### Hardcoding Colors

```kotlin
// Bad
Text("Hello", color = Color.Blue)

// Good
Text(
    "Hello",
    color = MaterialTheme.colorScheme.primary
)
```

### Mixing Color Pairs

```kotlin
// Bad - May have poor contrast
Surface(color = colorScheme.primary) {
    Text("Text", color = colorScheme.onSurface)  // Wrong pair!
}

// Good - Correct pairs
Surface(color = colorScheme.primary) {
    Text("Text", color = colorScheme.onPrimary)  // ✅
}
```

### Custom Typography without Scale

```kotlin
// Bad - Fixed size
Text("Title", fontSize = 24.sp)

// Good - Use type scale
Text(
    "Title",
    style = MaterialTheme.typography.headlineMedium
)
```

## Summary

- Design tokens: foundational values
- Color roles: semantic meaning
- Dynamic color: from wallpaper (Android 12+)
- Typography: 15 text styles
- Shape: 4 shape scales
- Elevation: tonal and shadow
- Adaptive to screen size
- Built-in accessibility

## Resources

- [Material3 Documentation](https://m3.material.io/)
- [Material Design Guidelines](https://m3.material.io/foundations)
- [Material Theme Builder](https://material-foundation.github.io/material-theme-builder/)
- [Compose Material3](https://developer.android.com/jetpack/compose/designsystems/material3)
- [Adaptive Layouts](https://m3.material.io/foundations/layout/applying-layout)

