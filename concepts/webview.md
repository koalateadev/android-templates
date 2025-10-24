# WebView Deep Dive

## Overview

WebView allows you to display web content in your Android app. Understanding its capabilities, limitations, and security implications is crucial.

## WebView vs Alternatives

### When to Use What

| Feature | WebView | Custom Tabs | Browser Intent |
|---------|---------|-------------|----------------|
| **Control** | Full control | Limited | None |
| **Security** | Your responsibility | Chrome handles | Chrome handles |
| **Performance** | Good | Better | Best |
| **UI Customization** | ✅ Full | ⚠️ Limited | ❌ None |
| **Cookie Sharing** | ❌ Isolated | ✅ Shares with Chrome | ✅ Shares |
| **Login State** | ❌ Separate | ✅ Shared | ✅ Shared |
| **File Downloads** | Manual | Automatic | Automatic |
| **Back Button** | Manual | Automatic | N/A |
| **Use Case** | Embedded content | External links | Open browser |

### Custom Tabs (Recommended for URLs)

```kotlin
implementation("androidx.browser:browser:1.8.0")

fun openCustomTab(context: Context, url: String) {
    val builder = CustomTabsIntent.Builder()
    builder.setShowTitle(true)
    builder.setColorScheme(CustomTabsIntent.COLOR_SCHEME_SYSTEM)
    
    val customTabsIntent = builder.build()
    customTabsIntent.launchUrl(context, Uri.parse(url))
}
```

### Browser Intent (Simple Links)

```kotlin
fun openBrowser(context: Context, url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    context.startActivity(intent)
}
```

## Basic WebView Setup

### Dependencies

```toml
[versions]
webkit = "1.12.1"

[libraries]
androidx-webkit = { group = "androidx.webkit", name = "webkit", version.ref = "webkit" }
```

```kotlin
dependencies {
    implementation(libs.androidx.webkit)  // For WebViewCompat features
}
```

### XML Layout

```xml
<WebView
    android:id="@+id/webView"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

### Basic Usage

```kotlin
class WebViewActivity : AppCompatActivity() {
    
    private lateinit var webView: WebView
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_webview)
        
        webView = findViewById(R.id.webView)
        
        // Enable JavaScript (disabled by default)
        webView.settings.javaScriptEnabled = true
        
        // Load URL
        webView.loadUrl("https://www.example.com")
        
        // Or load HTML
        webView.loadData(
            "<html><body><h1>Hello World</h1></body></html>",
            "text/html",
            "UTF-8"
        )
    }
    
    // Handle back button
    override fun onBackPressed() {
        if (webView.canGoBack()) {
            webView.goBack()
        } else {
            super.onBackPressed()
        }
    }
    
    // Clean up
    override fun onDestroy() {
        webView.destroy()
        super.onDestroy()
    }
}
```

## WebView Settings

### Security Settings

```kotlin
webView.settings.apply {
    // JavaScript
    javaScriptEnabled = false  // Only enable if necessary
    
    // File access
    allowFileAccess = false  // Prevent file:// access
    allowContentAccess = false  // Prevent content:// access
    allowFileAccessFromFileURLs = false  // Prevent file:// → file://
    allowUniversalAccessFromFileURLs = false  // Prevent file:// → http://
    
    // Mixed content (HTTPS page with HTTP resources)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
    }
    
    // Geolocation
    setGeolocationEnabled(false)  // Only enable if needed
    
    // Storage
    databaseEnabled = false  // Disable web SQL
    domStorageEnabled = false  // Disable DOM storage (enable if needed)
    
    // Save form data (deprecated, disable)
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) {
        @Suppress("DEPRECATION")
        saveFormData = false
    }
}
```

### Display Settings

```kotlin
webView.settings.apply {
    // Zoom
    setSupportZoom(true)
    builtInZoomControls = true
    displayZoomControls = false  // Hide zoom buttons
    
    // Viewport
    useWideViewPort = true
    loadWithOverviewMode = true
    
    // Text
    textZoom = 100  // Default size
    
    // Cache
    cacheMode = WebSettings.LOAD_DEFAULT
    // LOAD_CACHE_ELSE_NETWORK - prefer cache
    // LOAD_NO_CACHE - don't use cache
    // LOAD_CACHE_ONLY - only use cache
}
```

## WebViewClient (Handle Navigation)

### Basic WebViewClient

```kotlin
webView.webViewClient = object : WebViewClient() {
    
    override fun shouldOverrideUrlLoading(
        view: WebView?,
        request: WebResourceRequest?
    ): Boolean {
        val url = request?.url?.toString() ?: return false
        
        return when {
            // Handle custom schemes
            url.startsWith("myapp://") -> {
                handleDeepLink(url)
                true  // Don't load in WebView
            }
            
            // Open external links in browser
            url.startsWith("https://external.com") -> {
                openInBrowser(url)
                true  // Don't load in WebView
            }
            
            // Load in WebView
            else -> false
        }
    }
    
    override fun onPageStarted(view: WebView?, url: String?, favicon: Bitmap?) {
        super.onPageStarted(view, url, favicon)
        // Show loading indicator
        showLoading(true)
    }
    
    override fun onPageFinished(view: WebView?, url: String?) {
        super.onPageFinished(view, url)
        // Hide loading indicator
        showLoading(false)
    }
    
    override fun onReceivedError(
        view: WebView?,
        request: WebResourceRequest?,
        error: WebResourceError?
    ) {
        super.onReceivedError(view, request, error)
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            showError("Error: ${error?.description}")
        }
    }
    
    override fun onReceivedSslError(
        view: WebView?,
        handler: SslErrorHandler?,
        error: SslError?
    ) {
        // ❌ NEVER call handler.proceed() in production!
        handler?.cancel()
        showError("SSL Error")
    }
}
```

## WebChromeClient (JavaScript Dialogs)

### Handle Dialogs and Progress

```kotlin
webView.webChromeClient = object : WebChromeClient() {
    
    override fun onProgressChanged(view: WebView?, newProgress: Int) {
        super.onProgressChanged(view, newProgress)
        // Update progress bar (0-100)
        progressBar.progress = newProgress
        
        if (newProgress == 100) {
            progressBar.visibility = View.GONE
        }
    }
    
    override fun onJsAlert(
        view: WebView?,
        url: String?,
        message: String?,
        result: JsResult?
    ): Boolean {
        // Handle JavaScript alert()
        AlertDialog.Builder(context)
            .setTitle("Alert")
            .setMessage(message)
            .setPositiveButton("OK") { _, _ ->
                result?.confirm()
            }
            .setOnCancelListener {
                result?.cancel()
            }
            .create()
            .show()
        
        return true  // We handled it
    }
    
    override fun onJsConfirm(
        view: WebView?,
        url: String?,
        message: String?,
        result: JsResult?
    ): Boolean {
        // Handle JavaScript confirm()
        AlertDialog.Builder(context)
            .setTitle("Confirm")
            .setMessage(message)
            .setPositiveButton("OK") { _, _ ->
                result?.confirm()
            }
            .setNegativeButton("Cancel") { _, _ ->
                result?.cancel()
            }
            .setOnCancelListener {
                result?.cancel()
            }
            .create()
            .show()
        
        return true
    }
    
    override fun onConsoleMessage(consoleMessage: ConsoleMessage?): Boolean {
        // Log JavaScript console messages
        consoleMessage?.let {
            Timber.d("JS Console [${it.sourceId()}:${it.lineNumber()}]: ${it.message()}")
        }
        return true
    }
    
    // File upload (camera/gallery)
    override fun onShowFileChooser(
        webView: WebView?,
        filePathCallback: ValueCallback<Array<Uri>>?,
        fileChooserParams: FileChooserParams?
    ): Boolean {
        // Handle file selection
        this@WebViewActivity.filePathCallback = filePathCallback
        
        val intent = fileChooserParams?.createIntent()
        startActivityForResult(intent, FILE_CHOOSER_REQUEST_CODE)
        
        return true
    }
}
```

## JavaScript Bridge

### Android → JavaScript

```kotlin
// Call JavaScript from Android
webView.evaluateJavascript(
    "updateUserName('John Doe')"
) { result ->
    // result contains return value
    Timber.d("JavaScript returned: $result")
}

// Load JavaScript
webView.loadUrl("javascript:alert('Hello from Android')")
```

### JavaScript → Android

```kotlin
class WebAppInterface(private val context: Context) {
    
    @JavascriptInterface
    fun showToast(message: String) {
        Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
    }
    
    @JavascriptInterface
    fun getUserData(): String {
        return """{"name": "John", "email": "john@example.com"}"""
    }
    
    @JavascriptInterface
    fun navigateToScreen(screenName: String) {
        // Handle navigation
        Handler(Looper.getMainLooper()).post {
            when (screenName) {
                "profile" -> navigateToProfile()
                "settings" -> navigateToSettings()
            }
        }
    }
}

// Add interface
webView.addJavascriptInterface(WebAppInterface(this), "Android")

// Call from JavaScript:
// Android.showToast("Hello!");
// var userData = Android.getUserData();
// Android.navigateToScreen("profile");
```

### Security Considerations

```kotlin
// ⚠️ Only add JavaScript interface if:
// 1. You control the web content
// 2. You use HTTPS only
// 3. You validate all inputs

class SecureWebAppInterface(private val context: Context) {
    
    @JavascriptInterface
    fun processData(data: String) {
        // ✅ Validate input
        if (!isValidInput(data)) {
            Timber.w("Invalid input from JavaScript")
            return
        }
        
        // ✅ Sanitize data
        val sanitized = sanitizeInput(data)
        
        // Process safely
        processSecurely(sanitized)
    }
    
    private fun isValidInput(data: String): Boolean {
        // Implement validation
        return data.length < 1000 && !data.contains("<script")
    }
    
    private fun sanitizeInput(data: String): String {
        return data.replace("<", "&lt;").replace(">", "&gt;")
    }
}
```

## WebView in Compose

### Using AndroidView

```kotlin
@Composable
fun WebViewScreen(url: String) {
    var loading by remember { mutableStateOf(true) }
    var error by remember { mutableStateOf<String?>(null) }
    
    Box(modifier = Modifier.fillMaxSize()) {
        AndroidView(
            factory = { context ->
                WebView(context).apply {
                    settings.javaScriptEnabled = true
                    
                    webViewClient = object : WebViewClient() {
                        override fun onPageStarted(
                            view: WebView?,
                            url: String?,
                            favicon: Bitmap?
                        ) {
                            loading = true
                        }
                        
                        override fun onPageFinished(view: WebView?, url: String?) {
                            loading = false
                        }
                        
                        override fun onReceivedError(
                            view: WebView?,
                            request: WebResourceRequest?,
                            webError: WebResourceError?
                        ) {
                            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                                error = webError?.description?.toString()
                            }
                        }
                    }
                    
                    loadUrl(url)
                }
            },
            update = { webView ->
                webView.loadUrl(url)
            }
        )
        
        if (loading) {
            CircularProgressIndicator(
                modifier = Modifier.align(Alignment.Center)
            )
        }
        
        error?.let { errorMessage ->
            ErrorScreen(
                message = errorMessage,
                onRetry = { error = null },
                modifier = Modifier.align(Alignment.Center)
            )
        }
    }
}
```

### Advanced Compose WebView

```kotlin
@Composable
fun rememberWebViewState(url: String): WebViewState {
    return remember(url) {
        WebViewState(url)
    }
}

class WebViewState(val url: String) {
    var loading by mutableStateOf(true)
    var error by mutableStateOf<String?>(null)
    var canGoBack by mutableStateOf(false)
    var canGoForward by mutableStateOf(false)
    var progress by mutableStateOf(0)
    
    lateinit var webView: WebView
}

@Composable
fun AdvancedWebView(state: WebViewState) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Web View") },
                navigationIcon = {
                    if (state.canGoBack) {
                        IconButton(onClick = { state.webView.goBack() }) {
                            Icon(Icons.Default.ArrowBack, "Back")
                        }
                    }
                },
                actions = {
                    IconButton(onClick = { state.webView.reload() }) {
                        Icon(Icons.Default.Refresh, "Reload")
                    }
                }
            )
        }
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            if (state.progress < 100) {
                LinearProgressIndicator(
                    progress = { state.progress / 100f },
                    modifier = Modifier.fillMaxWidth()
                )
            }
            
            AndroidView(
                factory = { context ->
                    WebView(context).apply {
                        state.webView = this
                        setupWebView(state)
                        loadUrl(state.url)
                    }
                },
                modifier = Modifier.fillMaxSize()
            )
        }
    }
}

private fun WebView.setupWebView(state: WebViewState) {
    settings.javaScriptEnabled = true
    
    webViewClient = object : WebViewClient() {
        override fun onPageStarted(view: WebView?, url: String?, favicon: Bitmap?) {
            state.loading = true
        }
        
        override fun onPageFinished(view: WebView?, url: String?) {
            state.loading = false
            state.canGoBack = canGoBack()
            state.canGoForward = canGoForward()
        }
        
        override fun onReceivedError(
            view: WebView?,
            request: WebResourceRequest?,
            error: WebResourceError?
        ) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                state.error = error?.description?.toString()
            }
        }
    }
    
    webChromeClient = object : WebChromeClient() {
        override fun onProgressChanged(view: WebView?, newProgress: Int) {
            state.progress = newProgress
        }
    }
}
```

## File Upload Support

### Handle File Selection

```kotlin
class WebViewActivity : AppCompatActivity() {
    
    private var filePathCallback: ValueCallback<Array<Uri>>? = null
    
    private val filePickerLauncher = registerForActivityResult(
        ActivityResultContracts.GetContent()
    ) { uri: Uri? ->
        val result = uri?.let { arrayOf(it) }
        filePathCallback?.onReceiveValue(result)
        filePathCallback = null
    }
    
    private fun setupWebView() {
        webView.webChromeClient = object : WebChromeClient() {
            override fun onShowFileChooser(
                webView: WebView?,
                callback: ValueCallback<Array<Uri>>?,
                params: FileChooserParams?
            ): Boolean {
                filePathCallback?.onReceiveValue(null)
                filePathCallback = callback
                
                // Launch file picker
                val mimeTypes = params?.acceptTypes ?: arrayOf("*/*")
                filePickerLauncher.launch(mimeTypes.firstOrNull() ?: "*/*")
                
                return true
            }
        }
    }
}
```

### Camera Support

```kotlin
private val cameraLauncher = registerForActivityResult(
    ActivityResultContracts.TakePicture()
) { success ->
    val result = if (success && cameraUri != null) {
        arrayOf(cameraUri!!)
    } else {
        null
    }
    filePathCallback?.onReceiveValue(result)
    filePathCallback = null
}

override fun onShowFileChooser(
    webView: WebView?,
    callback: ValueCallback<Array<Uri>>?,
    params: FileChooserParams?
): Boolean {
    filePathCallback = callback
    
    // Show options: Camera or Gallery
    val options = arrayOf("Take Photo", "Choose from Gallery")
    AlertDialog.Builder(context)
        .setItems(options) { _, which ->
            when (which) {
                0 -> launchCamera()
                1 -> launchGallery()
            }
        }
        .setOnCancelListener {
            callback?.onReceiveValue(null)
        }
        .show()
    
    return true
}
```

## Download Support

### Handle Downloads

```kotlin
webView.setDownloadListener { url, userAgent, contentDisposition, mimeType, contentLength ->
    // Option 1: Use DownloadManager
    val request = DownloadManager.Request(Uri.parse(url))
    request.setMimeType(mimeType)
    request.setTitle("Downloading file")
    request.setDescription("Downloading from WebView")
    request.setNotificationVisibility(
        DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED
    )
    request.setDestinationInExternalPublicDir(
        Environment.DIRECTORY_DOWNLOADS,
        getFileNameFromUrl(url)
    )
    
    val downloadManager = context.getSystemService<DownloadManager>()
    downloadManager?.enqueue(request)
    
    Toast.makeText(context, "Downloading...", Toast.LENGTH_SHORT).show()
}

private fun getFileNameFromUrl(url: String): String {
    return url.substringAfterLast("/").substringBefore("?")
}
```

## Cookie Management

### Access Cookies

```kotlin
val cookieManager = CookieManager.getInstance()

// Set cookie
cookieManager.setCookie(url, "session_id=abc123")

// Get cookies
val cookies = cookieManager.getCookie(url)

// Accept cookies
cookieManager.setAcceptCookie(true)

// Accept third-party cookies
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    cookieManager.setAcceptThirdPartyCookies(webView, false)
}

// Clear cookies
cookieManager.removeAllCookies { success ->
    Timber.d("Cookies cleared: $success")
}

// Flush cookies to storage
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    cookieManager.flush()
}
```

## Loading Local Assets

### Load from Assets

```kotlin
// Load HTML file from assets
webView.loadUrl("file:///android_asset/index.html")

// Load HTML string with base URL
val htmlContent = """
    <html>
    <head><link rel="stylesheet" href="style.css"></head>
    <body><h1>Hello</h1></body>
    </html>
"""

webView.loadDataWithBaseURL(
    "file:///android_asset/",  // Base URL for relative resources
    htmlContent,
    "text/html",
    "UTF-8",
    null
)
```

### Inject CSS/JavaScript

```kotlin
// Inject CSS
webView.evaluateJavascript(
    """
    (function() {
        var style = document.createElement('style');
        style.innerHTML = 'body { background-color: #f0f0f0; }';
        document.head.appendChild(style);
    })();
    """.trimIndent()
) { }

// Inject JavaScript
webView.evaluateJavascript(
    """
    (function() {
        document.getElementById('myButton').click();
    })();
    """.trimIndent()
) { result ->
    Timber.d("Result: $result")
}
```

## Security Best Practices

### Secure WebView Configuration

```kotlin
fun createSecureWebView(context: Context): WebView {
    return WebView(context).apply {
        settings.apply {
            // Minimal permissions
            javaScriptEnabled = false  // Only if absolutely needed
            allowFileAccess = false
            allowContentAccess = false
            allowFileAccessFromFileURLs = false
            allowUniversalAccessFromFileURLs = false
            
            // HTTPS only
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
            }
            
            // Disable features not needed
            databaseEnabled = false
            setGeolocationEnabled(false)
            
            // Prevent caching sensitive data
            cacheMode = WebSettings.LOAD_NO_CACHE
        }
        
        // Validate SSL certificates
        webViewClient = object : WebViewClient() {
            override fun onReceivedSslError(
                view: WebView?,
                handler: SslErrorHandler?,
                error: SslError?
            ) {
                // NEVER call handler.proceed()
                handler?.cancel()
            }
        }
        
        // Don't add JavaScript interface unless necessary
        // If you must, validate ALL inputs
    }
}
```

### Input Validation

```kotlin
class SecureJavaScriptInterface(private val context: Context) {
    
    @JavascriptInterface
    fun processUserInput(input: String) {
        // ✅ Validate length
        if (input.length > 1000) {
            Timber.w("Input too long")
            return
        }
        
        // ✅ Validate format
        if (!isValidFormat(input)) {
            Timber.w("Invalid format")
            return
        }
        
        // ✅ Sanitize
        val sanitized = sanitizeHtml(input)
        
        // ✅ Process on main thread if updating UI
        Handler(Looper.getMainLooper()).post {
            processData(sanitized)
        }
    }
    
    private fun sanitizeHtml(input: String): String {
        return input
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;")
    }
}
```

## Debugging WebView

### Enable Remote Debugging

```kotlin
if (BuildConfig.DEBUG) {
    WebView.setWebContentsDebuggingEnabled(true)
}

// Then in Chrome: chrome://inspect
```

### Log Web Console

```kotlin
webView.webChromeClient = object : WebChromeClient() {
    override fun onConsoleMessage(message: ConsoleMessage): Boolean {
        Timber.tag("WebView").d(
            "${message.message()} -- From line ${message.lineNumber()} of ${message.sourceId()}"
        )
        return true
    }
}
```

## Memory Management

### Prevent Memory Leaks

```kotlin
class WebViewActivity : AppCompatActivity() {
    
    private lateinit var webView: WebView
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Create WebView programmatically with Application context
        webView = WebView(applicationContext)
        
        val container = findViewById<ViewGroup>(R.id.webview_container)
        container.addView(webView)
    }
    
    override fun onDestroy() {
        // Proper cleanup
        (webView.parent as? ViewGroup)?.removeView(webView)
        webView.clearHistory()
        webView.clearCache(true)
        webView.loadUrl("about:blank")
        webView.onPause()
        webView.removeAllViews()
        webView.destroy()
        
        super.onDestroy()
    }
}
```

### Pause/Resume

```kotlin
override fun onResume() {
    super.onResume()
    webView.onResume()  // Resume JavaScript timers
}

override fun onPause() {
    webView.onPause()  // Pause JavaScript timers
    super.onPause()
}
```

## Advanced Features

### Custom User Agent

```kotlin
webView.settings.userAgentString = "MyApp/1.0 ${webView.settings.userAgentString}"
```

### Handle Authentication

```kotlin
webView.webViewClient = object : WebViewClient() {
    override fun onReceivedHttpAuthRequest(
        view: WebView?,
        handler: HttpAuthHandler?,
        host: String?,
        realm: String?
    ) {
        // Show authentication dialog
        showAuthDialog { username, password ->
            handler?.proceed(username, password)
        }
    }
}
```

### Request Headers

```kotlin
val headers = mapOf(
    "Authorization" to "Bearer $token",
    "X-Custom-Header" to "value"
)

webView.loadUrl(url, headers)
```

## Testing WebView

### Test WebView Content

```kotlin
@Test
fun testWebViewLoads() {
    val scenario = launchActivity<WebViewActivity>()
    
    scenario.onActivity { activity ->
        val webView = activity.findViewById<WebView>(R.id.webView)
        
        // Wait for page load
        val latch = CountDownLatch(1)
        webView.webViewClient = object : WebViewClient() {
            override fun onPageFinished(view: WebView?, url: String?) {
                latch.countDown()
            }
        }
        
        webView.loadUrl("https://example.com")
        
        latch.await(10, TimeUnit.SECONDS)
        
        // Verify
        assertThat(webView.url).isEqualTo("https://example.com")
    }
}
```

## Best Practices

### Security Checklist

- [ ] JavaScript disabled unless required
- [ ] File access disabled
- [ ] Mixed content blocked
- [ ] SSL errors never ignored
- [ ] JavaScript interface validated
- [ ] Content from HTTPS only
- [ ] User input sanitized
- [ ] Cookies managed appropriately
- [ ] WebView debugging disabled in release
- [ ] Proper cleanup in onDestroy

### Performance Checklist

- [ ] WebView created with Application context
- [ ] Reuse WebView when possible
- [ ] Cache mode set appropriately
- [ ] onPause/onResume called
- [ ] Proper cleanup to prevent leaks
- [ ] Hardware acceleration enabled
- [ ] Large pages paginated

### User Experience Checklist

- [ ] Loading indicator shown
- [ ] Error states handled
- [ ] Back button navigation
- [ ] Pull-to-refresh (optional)
- [ ] Progress bar for loading
- [ ] File download support
- [ ] Zoom controls configured
- [ ] Viewport configured

## Common Issues

### WebView Not Loading

```kotlin
// Check internet permission
<uses-permission android:name="android.permission.INTERNET" />

// Check cleartext traffic (HTTP)
// In AndroidManifest.xml:
<application
    android:usesCleartextTraffic="true">  // Only for development!
```

### JavaScript Not Working

```kotlin
// Ensure JavaScript is enabled
webView.settings.javaScriptEnabled = true

// Check console for errors
webView.webChromeClient = object : WebChromeClient() {
    override fun onConsoleMessage(message: ConsoleMessage): Boolean {
        Timber.e("JS Error: ${message.message()}")
        return true
    }
}
```

### Memory Leaks

```kotlin
// ✅ Always call destroy()
override fun onDestroy() {
    webView.destroy()
    super.onDestroy()
}

// ✅ Use Application context
WebView(applicationContext)

// ✅ Remove from parent before destroy
(webView.parent as? ViewGroup)?.removeView(webView)
```

## When NOT to Use WebView

❌ **Don't use WebView for:**
- Simple external links → Use Custom Tabs
- OAuth login → Use Custom Tabs (better security)
- Full browser experience → Use Browser Intent
- Native performance needed → Build native UI
- Complex native features → Use native code

✅ **Use WebView for:**
- Embedded content you control
- Hybrid apps (React Native, Flutter use WebView)
- Displaying dynamic HTML content
- Web-based forms in your app
- In-app help/documentation

## Summary

### Key Takeaways

1. **Security First**
   - Disable JavaScript unless needed
   - Block file access
   - Never ignore SSL errors
   - Validate JavaScript bridge inputs

2. **Choose Right Tool**
   - WebView: Embedded content
   - Custom Tabs: External links
   - Browser: Simple URLs

3. **Proper Lifecycle**
   - Call onPause/onResume
   - Clean up in onDestroy
   - Handle back navigation

4. **Performance**
   - Use Application context
   - Reuse WebView instances
   - Proper memory management

5. **User Experience**
   - Show loading states
   - Handle errors gracefully
   - Support file uploads
   - Enable zoom if appropriate

## Resources

- [WebView Documentation](https://developer.android.com/reference/android/webkit/WebView)
- [WebView Best Practices](https://developer.android.com/guide/webapps/best-practices)
- [WebView Security](https://developer.android.com/guide/webapps/managing-webview)
- [Custom Tabs](https://developer.chrome.com/docs/android/custom-tabs/)
- [WebViewCompat](https://developer.android.com/reference/androidx/webkit/WebViewCompat)

