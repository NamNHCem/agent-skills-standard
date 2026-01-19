---
name: Jetpack Compose Testing
description: Testing strategies for Jetpack Compose UI.
metadata:
  labels: [compose, testing, ui-test, semantics]
  triggers:
    files: ['**/test/**/*.kt', '**/androidTest/**/*.kt']
    keywords: [ComposeTestRule, onNodeWithText, performClick, assertIsDisplayed]
---

# Jetpack Compose Testing

## **Priority: P2 (MAINTENANCE)**

Comprehensive testing strategies for Compose UI components.

## Test Setup

### Dependencies
```kotlin
dependencies {
    androidTestImplementation("androidx.compose.ui:ui-test-junit4:1.5.4")
    debugImplementation("androidx.compose.ui:ui-test-manifest:1.5.4")
    
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}
```

### ComposeTestRule
```kotlin
class HomeScreenTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Test
    fun homeScreen_displaysTitle() {
        composeTestRule.setContent {
            HomeScreen()
        }
        
        composeTestRule
            .onNodeWithText("Welcome")
            .assertIsDisplayed()
    }
}
```

## Finding Nodes

### By Text
```kotlin
composeTestRule.onNodeWithText("Click me").performClick()
composeTestRule.onNodeWithText("Welcome", substring = true)
```

### By Content Description
```kotlin
composeTestRule.onNodeWithContentDescription("Profile icon").assertExists()
```

### By Test Tag
```kotlin
// In composable
Button(
    onClick = { },
    modifier = Modifier.testTag("login_button")
) {
    Text("Login")
}

// In test
composeTestRule.onNodeWithTag("login_button").performClick()
```

### By Semantics
```kotlin
composeTestRule
    .onNode(hasText("Submit") and isEnabled())
    .performClick()
```

## Assertions

### Existence & Visibility
```kotlin
composeTestRule.onNodeWithText("Title").assertExists()
composeTestRule.onNodeWithText("Title").assertIsDisplayed()
composeTestRule.onNodeWithText("Hidden").assertDoesNotExist()
```

### State Assertions
```kotlin
composeTestRule.onNodeWithTag("checkbox").assertIsOn()
composeTestRule.onNodeWithTag("button").assertIsEnabled()
composeTestRule.onNodeWithTag("text").assertTextEquals("Expected")
```

## Actions

### Click & Touch
```kotlin
composeTestRule.onNodeWithText("Button").performClick()
composeTestRule.onNodeWithTag("item").performTouchInput {
    swipeLeft()
    swipeRight()
    longClick()
}
```

### Text Input
```kotlin
composeTestRule
    .onNodeWithTag("email_field")
    .performTextInput("test@example.com")

composeTestRule
    .onNodeWithTag("search")
    .performTextClearance()
```

### Scrolling
```kotlin
composeTestRule
    .onNodeWithTag("lazy_column")
    .performScrollToIndex(10)

composeTestRule
    .onNodeWithText("Item 50")
    .performScrollTo()
```

## Testing ViewModels

### With Fake Repository
```kotlin
@Test
fun viewModel_loadsData_updatesState() = runTest {
    val fakeRepository = FakeUserRepository()
    val viewModel = HomeViewModel(fakeRepository)
    
    // Collect state
    val states = mutableListOf<UiState>()
    backgroundScope.launch {
        viewModel.uiState.collect { states.add(it) }
    }
    
    // Trigger action
    viewModel.loadData()
    advanceUntilIdle()
    
    // Assert
    assertEquals(UiState.Loading, states[0])
    assertTrue(states[1] is UiState.Success)
}
```

### Integration Test
```kotlin
@Test
fun homeScreen_loadsAndDisplaysUser() {
    val fakeRepository = FakeUserRepository()
    val viewModel = HomeViewModel(fakeRepository)
    
    composeTestRule.setContent {
        HomeScreen(viewModel = viewModel)
    }
    
    // Initially loading
    composeTestRule.onNodeWithTag("loading").assertExists()
    
    // Wait for data
    composeTestRule.waitUntil(timeoutMillis = 5000) {
        composeTestRule
            .onAllNodesWithTag("user_item")
            .fetchSemanticsNodes().isNotEmpty()
    }
    
    // Assert data displayed
    composeTestRule.onNodeWithText("John Doe").assertIsDisplayed()
}
```

## Semantics

### Custom Semantics
```kotlin
@Composable
fun CustomButton(onClick: () -> Unit) {
    Button(
        onClick = onClick,
        modifier = Modifier.semantics {
            contentDescription = "Custom action button"
            role = Role.Button
        }
    ) {
        Text("Action")
    }
}

// Test
composeTestRule
    .onNode(hasContentDescription("Custom action button"))
    .performClick()
```

## Waiting & Synchronization

### Wait Until
```kotlin
composeTestRule.waitUntil(timeoutMillis = 5000) {
    composeTestRule
        .onAllNodesWithText("Loaded")
        .fetchSemanticsNodes().isNotEmpty()
}
```

### Idle Waiting
```kotlin
composeTestRule.waitForIdle()
```

## Screenshot Testing

### Capture Screenshots
```kotlin
@Test
fun homeScreen_matchesScreenshot() {
    composeTestRule.setContent {
        HomeScreen()
    }
    
    composeTestRule.onRoot().captureToImage()
        .assertAgainstGolden("home_screen")
}
```

## Anti-Patterns

- **No Hardcoded Delays**: Use waitUntil or waitForIdle
- **No Testing Implementation**: Test behavior, not internals
- **No Missing Test Tags**: Tag important UI elements
- **No Flaky Tests**: Ensure deterministic state

## Reference & Examples

See [references/REFERENCE.md](references/REFERENCE.md).

## Related Topics

jetpack-compose/ui-components | jetpack-compose/state-management
