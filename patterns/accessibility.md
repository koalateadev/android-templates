# Accessibility Patterns

## Table of Contents
- [Content Descriptions](#content-descriptions)
- [Semantic Properties](#semantic-properties)
- [Touch Target Sizes](#touch-target-sizes)
- [Screen Reader Support](#screen-reader-support)
- [Keyboard Navigation](#keyboard-navigation)
- [Color Contrast](#color-contrast)
- [Text Scaling](#text-scaling)
- [Accessibility Testing](#accessibility-testing)

## Content Descriptions

### Images and Icons

```kotlin
@Composable
fun AccessibleImage() {
    // Bad - No content description
    Icon(
        imageVector = Icons.Default.Settings,
        contentDescription = null
    )
    
    // Good - Descriptive content description
    Icon(
        imageVector = Icons.Default.Settings,
        contentDescription = "Open settings"
    )
    
    // Decorative images
    Icon(
        imageVector = Icons.Default.Star,
        contentDescription = null  // OK for decorative images
    )
}
```

### Buttons

```kotlin
@Composable
fun AccessibleButtons() {
    // Icon button
    IconButton(
        onClick = { /* Delete */ }
    ) {
        Icon(
            imageVector = Icons.Default.Delete,
            contentDescription = "Delete item"
        )
    }
    
    // Floating action button
    FloatingActionButton(
        onClick = { /* Add */ }
    ) {
        Icon(
            imageVector = Icons.Default.Add,
            contentDescription = "Add new item"
        )
    }
}
```

### Complex Components

```kotlin
@Composable
fun AccessibleCard(user: User) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .semantics(mergeDescendants = true) {
                contentDescription = "User ${user.name}, email ${user.email}, " +
                        "joined ${user.joinDate}"
            }
            .clickable { /* Navigate */ }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(user.name)
            Text(user.email)
            Text(user.joinDate)
        }
    }
}
```

## Semantic Properties

### Custom Semantics

```kotlin
@Composable
fun CustomSemantics() {
    Box(
        modifier = Modifier
            .size(100.dp)
            .semantics {
                // Role
                role = Role.Button
                
                // Content description
                contentDescription = "Custom button"
                
                // State
                stateDescription = "Enabled"
                
                // Custom actions
                onClick {
                    performAction()
                    true
                }
            }
            .clickable { performAction() }
    )
}
```

### Heading Semantics

```kotlin
@Composable
fun ScreenTitle(title: String) {
    Text(
        text = title,
        style = MaterialTheme.typography.headlineMedium,
        modifier = Modifier.semantics {
            heading()  // Mark as heading for screen readers
        }
    )
}
```

### Live Region

```kotlin
@Composable
fun LiveRegionExample(status: String) {
    Text(
        text = status,
        modifier = Modifier.semantics {
            liveRegion = LiveRegionMode.Polite
            // or LiveRegionMode.Assertive for urgent updates
        }
    )
}
```

### Toggle Semantics

```kotlin
@Composable
fun AccessibleToggle() {
    var checked by remember { mutableStateOf(false) }
    
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .toggleable(
                value = checked,
                role = Role.Switch,
                onValueChange = { checked = it }
            )
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Text("Enable notifications")
        Spacer(modifier = Modifier.weight(1f))
        Switch(
            checked = checked,
            onCheckedChange = null  // Handled by parent
        )
    }
}
```

## Touch Target Sizes

### Minimum Touch Targets

```kotlin
@Composable
fun AccessibleButton() {
    // Minimum 48dp x 48dp touch target
    IconButton(
        onClick = { /* Action */ },
        modifier = Modifier.size(48.dp)  // Ensure minimum size
    ) {
        Icon(
            imageVector = Icons.Default.Favorite,
            contentDescription = "Add to favorites",
            modifier = Modifier.size(24.dp)  // Icon can be smaller
        )
    }
}
```

### Increase Touch Target

```kotlin
@Composable
fun SmallIconWithLargeTouchTarget() {
    Box(
        modifier = Modifier
            .size(48.dp)  // Touch target
            .clickable { /* Action */ },
        contentAlignment = Alignment.Center
    ) {
        Icon(
            imageVector = Icons.Default.Close,
            contentDescription = "Close",
            modifier = Modifier.size(16.dp)  // Actual icon
        )
    }
}
```

## Screen Reader Support

### Merge Descendants

```kotlin
@Composable
fun ListItemWithMergedSemantics(item: Item) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clickable { /* Click */ }
            .semantics(mergeDescendants = true) {
                // Screen reader reads all text as one
            }
            .padding(16.dp)
    ) {
        AsyncImage(
            model = item.imageUrl,
            contentDescription = null,  // Merged with text
            modifier = Modifier.size(48.dp)
        )
        
        Column {
            Text(item.name)
            Text(item.description)
        }
    }
}
```

### Clear All Semantics

```kotlin
@Composable
fun DecorativeElement() {
    Box(
        modifier = Modifier
            .size(100.dp)
            .background(Color.Gray)
            .clearAndSetSemantics {}  // Hide from screen readers
    )
}
```

### Custom Accessibility Actions

```kotlin
@Composable
fun ItemWithActions(item: Item) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .semantics {
                customActions = listOf(
                    CustomAccessibilityAction("Edit") {
                        editItem(item)
                        true
                    },
                    CustomAccessibilityAction("Delete") {
                        deleteItem(item)
                        true
                    },
                    CustomAccessibilityAction("Share") {
                        shareItem(item)
                        true
                    }
                )
            }
    ) {
        Text(item.name, modifier = Modifier.padding(16.dp))
    }
}
```

## Keyboard Navigation

### Focus Management

```kotlin
@Composable
fun FocusableForm() {
    val focusManager = LocalFocusManager.current
    val (nameRef, emailRef, passwordRef) = remember { FocusRequester.createRefs() }
    
    Column {
        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("Name") },
            modifier = Modifier
                .focusRequester(nameRef)
                .focusProperties {
                    next = emailRef
                },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            )
        )
        
        OutlinedTextField(
            value = email,
            onValueChange = { email = it },
            label = { Text("Email") },
            modifier = Modifier
                .focusRequester(emailRef)
                .focusProperties {
                    previous = nameRef
                    next = passwordRef
                },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
            keyboardActions = KeyboardActions(
                onNext = { focusManager.moveFocus(FocusDirection.Down) }
            )
        )
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            modifier = Modifier
                .focusRequester(passwordRef)
                .focusProperties {
                    previous = emailRef
                },
            keyboardOptions = KeyboardOptions(imeAction = ImeAction.Done),
            keyboardActions = KeyboardActions(
                onDone = {
                    focusManager.clearFocus()
                    submitForm()
                }
            )
        )
    }
    
    LaunchedEffect(Unit) {
        nameRef.requestFocus()
    }
}
```

### Tab Navigation

```kotlin
@Composable
fun TabNavigableGrid(items: List<Item>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2)
    ) {
        items(items) { item ->
            Card(
                modifier = Modifier
                    .focusable()
                    .clickable { /* Click */ }
            ) {
                Text(item.name)
            }
        }
    }
}
```

## Color Contrast

### Check Contrast Ratio

```kotlin
object AccessibilityColors {
    
    fun getContrastRatio(foreground: Color, background: Color): Float {
        val fgLuminance = getLuminance(foreground)
        val bgLuminance = getLuminance(background)
        
        val lighter = maxOf(fgLuminance, bgLuminance)
        val darker = minOf(fgLuminance, bgLuminance)
        
        return (lighter + 0.05f) / (darker + 0.05f)
    }
    
    private fun getLuminance(color: Color): Float {
        val r = if (color.red <= 0.03928f) {
            color.red / 12.92f
        } else {
            ((color.red + 0.055f) / 1.055f).pow(2.4f)
        }
        
        val g = if (color.green <= 0.03928f) {
            color.green / 12.92f
        } else {
            ((color.green + 0.055f) / 1.055f).pow(2.4f)
        }
        
        val b = if (color.blue <= 0.03928f) {
            color.blue / 12.92f
        } else {
            ((color.blue + 0.055f) / 1.055f).pow(2.4f)
        }
        
        return 0.2126f * r + 0.7152f * g + 0.0722f * b
    }
    
    fun meetsWCAG_AA(contrastRatio: Float, isLargeText: Boolean = false): Boolean {
        return contrastRatio >= if (isLargeText) 3.0f else 4.5f
    }
    
    fun meetsWCAG_AAA(contrastRatio: Float, isLargeText: Boolean = false): Boolean {
        return contrastRatio >= if (isLargeText) 4.5f else 7.0f
    }
}
```

### High Contrast Theme

```kotlin
@Composable
fun HighContrastTheme(content: @Composable () -> Unit) {
    val highContrastColors = darkColorScheme(
        primary = Color(0xFFFFFFFF),
        onPrimary = Color(0xFF000000),
        background = Color(0xFF000000),
        onBackground = Color(0xFFFFFFFF),
        surface = Color(0xFF000000),
        onSurface = Color(0xFFFFFFFF)
    )
    
    MaterialTheme(
        colorScheme = highContrastColors,
        content = content
    )
}
```

## Text Scaling

### Responsive Text Sizes

```kotlin
@Composable
fun ScalableText(text: String) {
    // Text automatically scales with system font size
    Text(
        text = text,
        style = MaterialTheme.typography.bodyLarge,
        maxLines = 3,
        overflow = TextOverflow.Ellipsis
    )
}
```

### Limit Text Scale

```kotlin
@Composable
fun LimitedScaleText(text: String) {
    // Limit scaling if necessary (use sparingly)
    Text(
        text = text,
        modifier = Modifier.graphicsLayer {
            val scale = minOf(
                LocalDensity.current.fontScale,
                1.3f  // Max 1.3x scale
            )
            scaleX = scale
            scaleY = scale
        }
    )
}
```

## Accessibility Testing

### Enable Accessibility Scanner

```kotlin
// In debug builds, enable accessibility warnings
@Composable
fun DebugAccessibility(content: @Composable () -> Unit) {
    if (BuildConfig.DEBUG) {
        Box(
            modifier = Modifier.semantics {
                testTag = "accessibility_test"
            }
        ) {
            content()
        }
    } else {
        content()
    }
}
```

### Test with TalkBack

Manual testing steps:
1. Enable TalkBack: Settings → Accessibility → TalkBack
2. Navigate with swipe gestures
3. Verify all interactive elements are announced
4. Check content descriptions are meaningful
5. Test custom actions
6. Verify heading navigation

### Compose Accessibility Tests

```kotlin
@Test
fun testAccessibility() {
    composeTestRule.setContent {
        IconButton(onClick = {}) {
            Icon(
                imageVector = Icons.Default.Delete,
                contentDescription = "Delete item"
            )
        }
    }
    
    composeTestRule
        .onNodeWithContentDescription("Delete item")
        .assertIsDisplayed()
        .assertHasClickAction()
}
```

## Accessible Patterns

### Accessible List Items

```kotlin
@Composable
fun AccessibleListItem(
    title: String,
    description: String,
    timestamp: String,
    isUnread: Boolean,
    onClick: () -> Unit
) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .clickable(
                onClick = onClick,
                role = Role.Button
            )
            .semantics(mergeDescendants = true) {
                contentDescription = buildString {
                    append(title)
                    append(", ")
                    append(description)
                    append(", ")
                    append(timestamp)
                    if (isUnread) {
                        append(", unread")
                    }
                }
            }
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        if (isUnread) {
            Box(
                modifier = Modifier
                    .size(8.dp)
                    .background(Color.Blue, CircleShape)
                    .clearAndSetSemantics {}  // Hide from screen reader
            )
            Spacer(modifier = Modifier.width(8.dp))
        }
        
        Column(modifier = Modifier.weight(1f)) {
            Text(title, style = MaterialTheme.typography.titleMedium)
            Text(description, style = MaterialTheme.typography.bodyMedium)
            Text(timestamp, style = MaterialTheme.typography.bodySmall)
        }
    }
}
```

### Accessible Form

```kotlin
@Composable
fun AccessibleLoginForm() {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(
            text = "Login",
            style = MaterialTheme.typography.headlineMedium,
            modifier = Modifier.semantics { heading() }
        )
        
        var email by remember { mutableStateOf("") }
        var emailError by remember { mutableStateOf<String?>(null) }
        
        OutlinedTextField(
            value = email,
            onValueChange = {
                email = it
                emailError = if (it.isEmpty()) "Email is required" else null
            },
            label = { Text("Email") },
            isError = emailError != null,
            supportingText = emailError?.let { { Text(it) } },
            modifier = Modifier
                .fillMaxWidth()
                .semantics {
                    if (emailError != null) {
                        error(emailError)
                    }
                }
        )
        
        var password by remember { mutableStateOf("") }
        var passwordVisible by remember { mutableStateOf(false) }
        
        OutlinedTextField(
            value = password,
            onValueChange = { password = it },
            label = { Text("Password") },
            visualTransformation = if (passwordVisible) {
                VisualTransformation.None
            } else {
                PasswordVisualTransformation()
            },
            trailingIcon = {
                IconButton(onClick = { passwordVisible = !passwordVisible }) {
                    Icon(
                        imageVector = if (passwordVisible) {
                            Icons.Default.VisibilityOff
                        } else {
                            Icons.Default.Visibility
                        },
                        contentDescription = if (passwordVisible) {
                            "Hide password"
                        } else {
                            "Show password"
                        }
                    )
                }
            },
            modifier = Modifier.fillMaxWidth()
        )
        
        Button(
            onClick = { /* Login */ },
            modifier = Modifier
                .fillMaxWidth()
                .padding(top = 16.dp)
        ) {
            Text("Login")
        }
    }
}
```

### Accessible Dialog

```kotlin
@Composable
fun AccessibleDialog(
    title: String,
    message: String,
    onConfirm: () -> Unit,
    onDismiss: () -> Unit
) {
    AlertDialog(
        onDismissRequest = onDismiss,
        title = {
            Text(
                text = title,
                modifier = Modifier.semantics { heading() }
            )
        },
        text = {
            Text(message)
        },
        confirmButton = {
            TextButton(onClick = onConfirm) {
                Text("Confirm")
            }
        },
        dismissButton = {
            TextButton(onClick = onDismiss) {
                Text("Cancel")
            }
        },
        properties = DialogProperties(
            dismissOnClickOutside = true,
            dismissOnBackPress = true
        )
    )
}
```

## Announcing Changes

### Announce for Accessibility

```kotlin
@Composable
fun AnnouncementExample() {
    val context = LocalContext.current
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")
        
        Button(onClick = {
            count++
            announceForAccessibility(context, "Count is now $count")
        }) {
            Text("Increment")
        }
    }
}

fun announceForAccessibility(context: Context, message: String) {
    val view = (context as? Activity)?.findViewById<View>(android.R.id.content)
    view?.announceForAccessibility(message)
}
```

## Best Practices

- Add content descriptions to interactive elements
- Use semantic properties appropriately
- Ensure minimum touch targets (48dp)
- Support text scaling
- Provide sufficient color contrast (4.5:1)
- Merge semantics for complex components
- Mark headings with heading()
- Test with TalkBack
- Support keyboard navigation
- Provide error messages in forms
- Hide decorative elements from screen readers

## Accessibility Checklist

- [ ] All images have content descriptions (or marked decorative)
- [ ] All buttons have meaningful labels
- [ ] Touch targets are at least 48dp
- [ ] Color contrast meets WCAG AA (4.5:1)
- [ ] Text scales properly with system settings
- [ ] Forms have error announcements
- [ ] Interactive elements have proper roles
- [ ] Screen titles are marked as headings
- [ ] Tested with TalkBack
- [ ] Tested with large font sizes (200%)
- [ ] Keyboard navigation works
- [ ] Focus indicators are visible
- [ ] Custom actions work with TalkBack
- [ ] Live regions announce updates
- [ ] No information conveyed by color alone

## Resources

- [Accessibility in Compose](https://developer.android.com/jetpack/compose/accessibility)
- [Material Accessibility](https://m3.material.io/foundations/accessible-design)
- [WCAG Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Android Accessibility Guide](https://developer.android.com/guide/topics/ui/accessibility)
- [Accessibility Scanner](https://play.google.com/store/apps/details?id=com.google.android.apps.accessibility.auditor)

