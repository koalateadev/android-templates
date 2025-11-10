# Compose UI Testing Setup

## Overview
Test your Jetpack Compose UI with Compose Testing framework for reliable UI tests.

## Setup Steps

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
compose-bom = "2024.09.03"

[libraries]
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
androidx-compose-ui-test-junit4 = { group = "androidx.compose.ui", name = "ui-test-junit4" }
androidx-compose-ui-test-manifest = { group = "androidx.compose.ui", name = "ui-test-manifest" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    // BOM manages versions
    androidTestImplementation(platform(libs.androidx.compose.bom))
    androidTestImplementation(libs.androidx.compose.ui.test.junit4)
    
    // Debug implementations
    debugImplementation(libs.androidx.compose.ui.test.manifest)
    debugImplementation(libs.androidx.compose.ui.tooling)
}
```

## Basic Test Structure

### Simple Compose Test

```kotlin
import androidx.compose.ui.test.*
import androidx.compose.ui.test.junit4.createComposeRule
import org.junit.Rule
import org.junit.Test

class ButtonTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Test
    fun button_displays_text() {
        // Arrange
        composeTestRule.setContent {
            Button(onClick = {}) {
                Text("Click Me")
            }
        }
        
        // Assert
        composeTestRule
            .onNodeWithText("Click Me")
            .assertExists()
            .assertIsDisplayed()
    }
    
    @Test
    fun button_click_triggers_callback() {
        // Arrange
        var clicked = false
        composeTestRule.setContent {
            Button(onClick = { clicked = true }) {
                Text("Click Me")
            }
        }
        
        // Act
        composeTestRule
            .onNodeWithText("Click Me")
            .performClick()
        
        // Assert
        assert(clicked)
    }
}
```

## Finders (Matchers)

### Finding Nodes

```kotlin
// By text
composeTestRule.onNodeWithText("Hello")
composeTestRule.onNodeWithText("Hello", substring = true, ignoreCase = true)

// By content description
composeTestRule.onNodeWithContentDescription("Profile picture")

// By test tag
composeTestRule.onNodeWithTag("login_button")

// By semantics
composeTestRule.onNode(hasText("Hello"))
composeTestRule.onNode(hasContentDescription("Icon"))
composeTestRule.onNode(isEnabled())

// Multiple nodes
composeTestRule.onAllNodesWithText("Item")
composeTestRule.onAllNodesWithTag("list_item")

// Get specific node from collection
composeTestRule.onAllNodesWithTag("item")[0]
composeTestRule.onAllNodesWithTag("item").onFirst()
composeTestRule.onAllNodesWithTag("item").onLast()
```

### Custom Matchers

```kotlin
fun hasBackgroundColor(color: Color): SemanticsMatcher {
    return SemanticsMatcher("hasBackgroundColor $color") { node ->
        // Custom matching logic
        true
    }
}

// Usage
composeTestRule
    .onNode(hasBackgroundColor(Color.Red))
    .assertExists()
```

## Actions

### User Interactions

```kotlin
// Click
composeTestRule.onNodeWithText("Button").performClick()

// Long click
composeTestRule.onNodeWithTag("item").performLongClick()

// Text input
composeTestRule
    .onNodeWithTag("email_input")
    .performTextInput("user@example.com")

// Clear text
composeTestRule.onNodeWithTag("input").performTextClearance()

// Replace text
composeTestRule.onNodeWithTag("input").performTextReplacement("New text")

// Scroll
composeTestRule.onNodeWithTag("list").performScrollToIndex(10)
composeTestRule.onNodeWithText("Item 5").performScrollTo()

// Swipe
composeTestRule.onNodeWithTag("item").performTouchInput {
    swipeLeft()
    swipeRight()
    swipeUp()
    swipeDown()
}

// Custom gestures
composeTestRule.onNodeWithTag("canvas").performTouchInput {
    down(Offset(10f, 10f))
    moveTo(Offset(100f, 100f))
    up()
}
```

## Assertions

### State Assertions

```kotlin
// Existence
composeTestRule.onNodeWithText("Hello").assertExists()
composeTestRule.onNodeWithText("Goodbye").assertDoesNotExist()

// Display
composeTestRule.onNodeWithTag("button").assertIsDisplayed()
composeTestRule.onNodeWithTag("hidden").assertIsNotDisplayed()

// Enable/Disable
composeTestRule.onNodeWithText("Submit").assertIsEnabled()
composeTestRule.onNodeWithText("Submit").assertIsNotEnabled()

// Selection
composeTestRule.onNodeWithTag("checkbox").assertIsSelected()
composeTestRule.onNodeWithTag("checkbox").assertIsNotSelected()

// Toggle
composeTestRule.onNodeWithTag("switch").assertIsOn()
composeTestRule.onNodeWithTag("switch").assertIsOff()

// Text
composeTestRule.onNodeWithTag("label").assertTextEquals("Expected Text")
composeTestRule.onNodeWithTag("label").assertTextContains("Expected")

// Content description
composeTestRule.onNodeWithTag("image").assertContentDescriptionEquals("Profile")

// Count
composeTestRule.onAllNodesWithTag("item").assertCountEquals(5)
```

### Custom Assertions

```kotlin
fun SemanticsNodeInteraction.assertHasBackgroundColor(expectedColor: Color): SemanticsNodeInteraction {
    return assert(
        SemanticsMatcher("has background color $expectedColor") { node ->
            // Custom assertion logic
            true
        }
    )
}
```

## Testing Lists

### LazyColumn/LazyRow

```kotlin
@Test
fun lazyList_displays_all_items() {
    val items = listOf("Item 1", "Item 2", "Item 3")
    
    composeTestRule.setContent {
        LazyColumn {
            items(items) { item ->
                Text(
                    text = item,
                    modifier = Modifier.testTag("list_item_$item")
                )
            }
        }
    }
    
    // Assert all items exist
    items.forEach { item ->
        composeTestRule
            .onNodeWithTag("list_item_$item")
            .assertExists()
    }
}

@Test
fun lazyList_scrolls_to_item() {
    composeTestRule.setContent {
        LazyColumn {
            items(100) { index ->
                Text(
                    text = "Item $index",
                    modifier = Modifier.testTag("item_$index")
                )
            }
        }
    }
    
    // Scroll to item
    composeTestRule
        .onNodeWithTag("item_50")
        .performScrollTo()
        .assertIsDisplayed()
}
```

## Testing with Test Tags

### Add Test Tags

```kotlin
@Composable
fun LoginScreen() {
    Column(modifier = Modifier.testTag("login_screen")) {
        TextField(
            value = email,
            onValueChange = { email = it },
            modifier = Modifier.testTag("email_input")
        )
        TextField(
            value = password,
            onValueChange = { password = it },
            modifier = Modifier.testTag("password_input")
        )
        Button(
            onClick = onLogin,
            modifier = Modifier.testTag("login_button")
        ) {
            Text("Login")
        }
    }
}
```

### Test with Tags

```kotlin
@Test
fun login_screen_validates_input() {
    composeTestRule.setContent {
        LoginScreen()
    }
    
    // Fill form
    composeTestRule
        .onNodeWithTag("email_input")
        .performTextInput("user@example.com")
    
    composeTestRule
        .onNodeWithTag("password_input")
        .performTextInput("password123")
    
    // Submit
    composeTestRule
        .onNodeWithTag("login_button")
        .performClick()
}
```

## Testing Navigation

```kotlin
@Test
fun navigation_from_home_to_details() {
    val navController = TestNavHostController(
        ApplicationProvider.getApplicationContext()
    )
    
    composeTestRule.setContent {
        navController.navigatorProvider.addNavigator(ComposeNavigator())
        
        NavHost(navController = navController, startDestination = "home") {
            composable("home") {
                HomeScreen(
                    onNavigateToDetails = { navController.navigate("details") }
                )
            }
            composable("details") {
                DetailsScreen()
            }
        }
    }
    
    // Verify home screen
    composeTestRule.onNodeWithTag("home_screen").assertExists()
    
    // Navigate
    composeTestRule.onNodeWithText("Go to Details").performClick()
    
    // Verify navigation
    composeTestRule.waitUntil {
        navController.currentDestination?.route == "details"
    }
    composeTestRule.onNodeWithTag("details_screen").assertExists()
}
```

## Testing with ViewModels

### With Fake ViewModel

```kotlin
@Test
fun displays_users_from_viewmodel() {
    val fakeViewModel = UserViewModel(FakeRepository())
    
    composeTestRule.setContent {
        UserScreen(viewModel = fakeViewModel)
    }
    
    // Wait for data to load
    composeTestRule.waitUntil(timeoutMillis = 2000) {
        composeTestRule
            .onAllNodesWithTag("user_item")
            .fetchSemanticsNodes()
            .isNotEmpty()
    }
    
    // Verify items displayed
    composeTestRule
        .onNodeWithText("John Doe")
        .assertIsDisplayed()
}
```

### With Hilt

```kotlin
@HiltAndroidTest
class UserScreenTest {
    
    @get:Rule(order = 0)
    val hiltRule = HiltAndroidRule(this)
    
    @get:Rule(order = 1)
    val composeTestRule = createAndroidComposeRule<MainActivity>()
    
    @Inject
    lateinit var repository: UserRepository
    
    @Before
    fun init() {
        hiltRule.inject()
    }
    
    @Test
    fun test_screen_with_injected_dependencies() {
        composeTestRule.setContent {
            UserScreen()
        }
        
        // Test assertions
    }
}
```

## Waiting and Synchronization

### Wait for Condition

```kotlin
// Wait until condition is true
composeTestRule.waitUntil(timeoutMillis = 5000) {
    composeTestRule
        .onAllNodesWithTag("item")
        .fetchSemanticsNodes()
        .size == 10
}

// Wait for idle
composeTestRule.waitForIdle()

// Wait until exists
composeTestRule.waitUntil {
    composeTestRule
        .onNodeWithText("Loaded")
        .fetchSemanticsNodes()
        .isNotEmpty()
}
```

### Main Clock Control

```kotlin
@Test
fun test_with_delayed_content() {
    composeTestRule.mainClock.autoAdvance = false
    
    composeTestRule.setContent {
        var show by remember { mutableStateOf(false) }
        
        LaunchedEffect(Unit) {
            delay(1000)
            show = true
        }
        
        if (show) {
            Text("Delayed Text")
        }
    }
    
    // Content not shown yet
    composeTestRule.onNodeWithText("Delayed Text").assertDoesNotExist()
    
    // Advance time
    composeTestRule.mainClock.advanceTimeBy(1000)
    
    // Content now shown
    composeTestRule.onNodeWithText("Delayed Text").assertExists()
}
```

## Testing State Changes

```kotlin
@Test
fun counter_increments_on_button_click() {
    composeTestRule.setContent {
        CounterScreen()
    }
    
    // Initial state
    composeTestRule.onNodeWithText("Count: 0").assertExists()
    
    // Click increment
    composeTestRule.onNodeWithTag("increment_button").performClick()
    
    // Verify state changed
    composeTestRule.onNodeWithText("Count: 1").assertExists()
    composeTestRule.onNodeWithText("Count: 0").assertDoesNotExist()
}
```

## Semantics Properties

### Add Custom Semantics

```kotlin
@Composable
fun CustomComponent() {
    Box(
        modifier = Modifier.semantics {
            contentDescription = "Custom component"
            testTag = "custom"
            // Custom properties
            this["customProperty"] = "value"
        }
    ) {
        // Content
    }
}
```

## Screenshot Testing

### Capture Screenshots

```kotlin
@Test
fun capture_screenshot() {
    composeTestRule.setContent {
        MyScreen()
    }
    
    composeTestRule
        .onNodeWithTag("screen")
        .captureToImage()
        .also { image ->
            // Save or compare image
        }
}
```

## Best Practices

- Use test tags for reliable element selection
- Wait for asynchronous operations
- Test user interactions, not implementation
- Keep tests focused and independent
- Test accessibility with semantics
- Test error states
- Verify navigation flows

## Common Patterns

### Test Loading State

```kotlin
@Test
fun shows_loading_indicator() {
    composeTestRule.setContent {
        UserScreen(uiState = UiState.Loading)
    }
    
    composeTestRule
        .onNodeWithTag("loading_indicator")
        .assertIsDisplayed()
}
```

### Test Error State

```kotlin
@Test
fun shows_error_message() {
    composeTestRule.setContent {
        UserScreen(uiState = UiState.Error("Network error"))
    }
    
    composeTestRule
        .onNodeWithText("Network error")
        .assertIsDisplayed()
    
    composeTestRule
        .onNodeWithTag("retry_button")
        .assertIsDisplayed()
}
```

### Test Empty State

```kotlin
@Test
fun shows_empty_state() {
    composeTestRule.setContent {
        UserList(users = emptyList())
    }
    
    composeTestRule
        .onNodeWithText("No users found")
        .assertIsDisplayed()
}
```

## Resources

- [Compose Testing Guide](https://developer.android.com/jetpack/compose/testing)
- [Testing Cheatsheet](https://developer.android.com/jetpack/compose/testing-cheatsheet)
- [Semantics in Compose](https://developer.android.com/jetpack/compose/semantics)

