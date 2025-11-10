# Android Lint Setup

## Overview
Lint is a code scanning tool that helps you identify and correct problems with the structural quality of your Android code.

## Built-in Configuration

Lint is built into Android Gradle Plugin, no additional dependencies needed!

## Basic Configuration

### Configure in `build.gradle.kts`

```kotlin
android {
    lint {
        // Turns off checks for the issue IDs you specify
        disable += listOf("TypographyFractions", "TypographyQuotes")
        
        // Turns on checks for the issue IDs you specify
        enable += listOf("RtlHardcoded", "RtlCompat", "RtlEnabled")
        
        // To enable checks that are off by default
        checkOnly += listOf("NewApi", "InlinedApi")
        
        // Treat warnings as errors
        warningsAsErrors = true
        
        // Turn off checking for other issue IDs
        abortOnError = false
        
        // Check all warnings, including those off by default
        checkAllWarnings = true
        
        // Ignore all warnings in test sources
        ignoreTestSources = true
        
        // Ignore warnings in generated code
        ignoreWarnings = false
        
        // Generate XML report
        xmlReport = true
        xmlOutput = file("build/reports/lint-results.xml")
        
        // Generate HTML report
        htmlReport = true
        htmlOutput = file("build/reports/lint-results.html")
        
        // Generate SARIF report
        sarifReport = true
        sarifOutput = file("build/reports/lint-results.sarif")
        
        // Set baseline file
        baseline = file("lint-baseline.xml")
        
        // Check dependencies
        checkDependencies = true
        
        // Enforce fatal severity
        fatal += listOf("StopShip")
        
        // Set error severity
        error += listOf("NewApi", "InlinedApi")
        
        // Set warning severity
        warning += listOf("UnusedResources")
        
        // Ignore severity
        informational += listOf("GoogleAppIndexingWarning")
    }
}
```

## Custom Lint Rules

### Create Lint Module

Create a new Java/Kotlin library module:

`lint-rules/build.gradle.kts`:
```kotlin
plugins {
    id("java-library")
    id("org.jetbrains.kotlin.jvm")
}

dependencies {
    compileOnly("com.android.tools.lint:lint-api:31.5.2")
    compileOnly("com.android.tools.lint:lint-checks:31.5.2")
    
    testImplementation("com.android.tools.lint:lint:31.5.2")
    testImplementation("com.android.tools.lint:lint-tests:31.5.2")
    testImplementation("junit:junit:4.13.2")
}

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
```

### Create Custom Rule

```kotlin
import com.android.tools.lint.detector.api.*
import org.jetbrains.uast.UCallExpression

class LogUsageDetector : Detector(), SourceCodeScanner {
    
    override fun getApplicableMethodNames(): List<String> {
        return listOf("d", "e", "i", "v", "w")
    }
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        if (context.evaluator.isMemberInClass(method, "android.util.Log")) {
            context.report(
                issue = ISSUE,
                scope = node,
                location = context.getLocation(node),
                message = "Use Timber instead of android.util.Log"
            )
        }
    }
    
    companion object {
        val ISSUE = Issue.create(
            id = "LogNotTimber",
            briefDescription = "Using android.util.Log instead of Timber",
            explanation = """
                Timber is preferred over android.util.Log for better logging control.
                Replace Log.d() with Timber.d(), Log.e() with Timber.e(), etc.
            """,
            category = Category.CORRECTNESS,
            priority = 6,
            severity = Severity.WARNING,
            implementation = Implementation(
                LogUsageDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

### Register Issues

```kotlin
import com.android.tools.lint.client.api.IssueRegistry
import com.android.tools.lint.detector.api.CURRENT_API

class CustomIssueRegistry : IssueRegistry() {
    override val issues = listOf(
        LogUsageDetector.ISSUE,
        // Add more custom issues here
    )
    
    override val api: Int = CURRENT_API
}
```

### Register in META-INF

Create `lint-rules/src/main/resources/META-INF/services/com.android.tools.lint.client.api.IssueRegistry`:
```
com.example.lint.CustomIssueRegistry
```

### Use Custom Lint Rules

In app `build.gradle.kts`:
```kotlin
dependencies {
    lintChecks(project(":lint-rules"))
}
```

## Common Lint Checks

### Hardcoded Strings

```xml
<!-- Bad -->
<TextView
    android:text="Hello World" />

<!-- Good -->
<TextView
    android:text="@string/hello_world" />
```

### Missing ContentDescription

```xml
<!-- Bad -->
<ImageView
    android:src="@drawable/icon" />

<!-- Good -->
<ImageView
    android:src="@drawable/icon"
    android:contentDescription="@string/icon_description" />
```

### Unused Resources

Lint will identify:
- Unused layouts
- Unused drawables
- Unused strings
- Unused dimensions

### Performance Issues

- Nested layouts
- Overdraw
- Memory leaks
- Inefficient data structures

## Lint Baselines

### Create Baseline

```bash
./gradlew lintDebug
# Creates lint-baseline.xml
```

### Use Baseline

```kotlin
android {
    lint {
        baseline = file("lint-baseline.xml")
    }
}
```

This allows you to acknowledge existing issues and focus on new ones.

## Suppress Warnings

### In XML

```xml
<LinearLayout
    tools:ignore="UselessParent">
    <!-- Content -->
</LinearLayout>
```

### In Kotlin/Java

```kotlin
@SuppressLint("SetTextI18n")
fun setText(textView: TextView) {
    textView.text = "Hello " + userName
}
```

### Multiple Issues

```kotlin
@SuppressLint("SetTextI18n", "HardcodedText")
fun setText(textView: TextView) {
    textView.text = "Hello World"
}
```

## Lint Configuration File

Create `lint.xml` in your project root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lint>
    <!-- Disable specific checks -->
    <issue id="IconMissingDensityFolder" severity="ignore" />
    
    <!-- Treat as error -->
    <issue id="NewApi" severity="error" />
    
    <!-- Ignore in specific paths -->
    <issue id="UnusedResources">
        <ignore path="src/main/res/drawable/legacy_*.xml" />
    </issue>
    
    <!-- Custom configuration -->
    <issue id="HardcodedText">
        <ignore path="src/androidTest/**" />
        <ignore path="src/test/**" />
    </issue>
</lint>
```

## Run Lint

### Command Line

```bash
# Run lint on all variants
./gradlew lint

# Run lint on specific variant
./gradlew lintDebug
./gradlew lintRelease

# Generate report
./gradlew lint

# Fix auto-fixable issues
./gradlew lintFix
```

### Android Studio

- **Analyze → Inspect Code**
- **Analyze → Run Inspection by Name**
- **Code → Inspect Code**

## CI/CD Integration

### GitHub Actions

```yaml
name: Lint Check

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run Lint
        run: ./gradlew lintDebug
      
      - name: Upload Lint Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: lint-report
          path: app/build/reports/lint-results-*.html
```

### GitLab CI

```yaml
lint:
  stage: test
  script:
    - ./gradlew lintDebug
  artifacts:
    when: always
    paths:
      - app/build/reports/lint-results-*.html
    reports:
      junit: app/build/test-results/lint-results.xml
```

## Common Issues and Fixes

### NewApi

```kotlin
// Bad
fun someMethod() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        // Use API 23+ feature
    }
}

// Good
@RequiresApi(Build.VERSION_CODES.M)
fun someMethod() {
    // Use API 23+ feature
}

// Or
@TargetApi(Build.VERSION_CODES.M)
fun someMethod() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
        // Use API 23+ feature
    }
}
```

### Recycle

```kotlin
// Bad
val cursor = contentResolver.query(uri, null, null, null, null)
// Use cursor without closing

// Good
contentResolver.query(uri, null, null, null, null)?.use { cursor ->
    // Use cursor
}
```

### WrongThread

```kotlin
// Bad
class MyViewModel : ViewModel() {
    fun loadData() {
        // Network call on main thread
        apiService.getData()
    }
}

// Good
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch(Dispatchers.IO) {
            apiService.getData()
        }
    }
}
```

## Custom Lint Checks Examples

### No Direct Toast Usage

```kotlin
class NoToastDetector : Detector(), SourceCodeScanner {
    override fun getApplicableMethodNames() = listOf("makeText")
    
    override fun visitMethodCall(context: JavaContext, node: UCallExpression, method: PsiMethod) {
        if (context.evaluator.isMemberInClass(method, "android.widget.Toast")) {
            context.report(
                issue = ISSUE,
                scope = node,
                location = context.getLocation(node),
                message = "Use SnackbarManager instead of Toast"
            )
        }
    }
    
    companion object {
        val ISSUE = Issue.create(
            id = "DirectToastUsage",
            briefDescription = "Direct Toast usage",
            explanation = "Use centralized SnackbarManager for better control",
            category = Category.CORRECTNESS,
            priority = 5,
            severity = Severity.WARNING,
            implementation = Implementation(
                NoToastDetector::class.java,
                Scope.JAVA_FILE_SCOPE
            )
        )
    }
}
```

## Best Practices

- Run lint regularly during development
- Fix errors before committing
- Use baseline for legacy code
- Create custom rules for project-specific issues
- Integrate into CI/CD pipeline
- Review reports in PRs
- Document suppressed warnings

## Performance

### Speed Up Lint

```kotlin
android {
    lint {
        checkDependencies = false  // Skip dependency checks
        checkOnly += listOf("NewApi", "InlinedApi")  // Check only specific issues
        ignoreTestSources = true
    }
}
```

## Resources

- [Official Documentation](https://developer.android.com/studio/write/lint)
- [Lint Checks Reference](https://googlesamples.github.io/android-custom-lint-rules/)
- [Writing Custom Lint Rules](https://github.com/googlesamples/android-custom-lint-rules)
- [Lint API Guide](https://googlesamples.github.io/android-custom-lint-rules/api-guide.html)

