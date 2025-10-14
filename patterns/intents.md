# Common Intents in Android

## Table of Contents
- [Opening URLs](#opening-urls)
- [Sharing Content](#sharing-content)
- [Email](#email)
- [Phone & SMS](#phone--sms)
- [Maps & Navigation](#maps--navigation)
- [Camera & Gallery](#camera--gallery)
- [File Picking](#file-picking)
- [App Settings](#app-settings)
- [Other Apps](#other-apps)

## Opening URLs

### Open Web Browser

```kotlin
@Composable
fun OpenUrlButton(url: String) {
    val context = LocalContext.current
    
    Button(onClick = {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
        context.startActivity(intent)
    }) {
        Text("Open Website")
    }
}

// Usage
OpenUrlButton("https://www.example.com")
```

### Open URL with Custom Tab

```kotlin
// Add dependency
implementation("androidx.browser:browser:1.8.0")

@Composable
fun OpenCustomTabButton(url: String) {
    val context = LocalContext.current
    
    Button(onClick = {
        val builder = CustomTabsIntent.Builder()
        builder.setShowTitle(true)
        builder.setColorScheme(CustomTabsIntent.COLOR_SCHEME_SYSTEM)
        
        val customTabsIntent = builder.build()
        customTabsIntent.launchUrl(context, Uri.parse(url))
    }) {
        Text("Open Link")
    }
}
```

### Open URL with Fallback

```kotlin
fun Context.openUrl(url: String) {
    try {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
        startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        Toast.makeText(this, "No browser found", Toast.LENGTH_SHORT).show()
    }
}

@Composable
fun SafeOpenUrl(url: String) {
    val context = LocalContext.current
    
    Button(onClick = { context.openUrl(url) }) {
        Text("Open URL")
    }
}
```

## Sharing Content

### Share Text

```kotlin
@Composable
fun ShareTextButton(text: String) {
    val context = LocalContext.current
    
    Button(onClick = {
        val sendIntent = Intent().apply {
            action = Intent.ACTION_SEND
            putExtra(Intent.EXTRA_TEXT, text)
            type = "text/plain"
        }
        val shareIntent = Intent.createChooser(sendIntent, null)
        context.startActivity(shareIntent)
    }) {
        Icon(Icons.Default.Share, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Share")
    }
}
```

### Share with Title

```kotlin
fun Context.shareText(text: String, title: String = "Share via") {
    val sendIntent = Intent().apply {
        action = Intent.ACTION_SEND
        putExtra(Intent.EXTRA_TEXT, text)
        putExtra(Intent.EXTRA_SUBJECT, title)
        type = "text/plain"
    }
    startActivity(Intent.createChooser(sendIntent, title))
}
```

### Share Image

```kotlin
@Composable
fun ShareImageButton(imageUri: Uri) {
    val context = LocalContext.current
    
    Button(onClick = {
        val shareIntent = Intent().apply {
            action = Intent.ACTION_SEND
            putExtra(Intent.EXTRA_STREAM, imageUri)
            type = "image/*"
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(shareIntent, "Share image"))
    }) {
        Text("Share Image")
    }
}
```

### Share Multiple Items

```kotlin
fun Context.shareMultipleImages(imageUris: ArrayList<Uri>) {
    val shareIntent = Intent().apply {
        action = Intent.ACTION_SEND_MULTIPLE
        putParcelableArrayListExtra(Intent.EXTRA_STREAM, imageUris)
        type = "image/*"
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    startActivity(Intent.createChooser(shareIntent, "Share images"))
}
```

## Email

### Send Email

```kotlin
@Composable
fun SendEmailButton(
    email: String,
    subject: String = "",
    body: String = ""
) {
    val context = LocalContext.current
    
    Button(onClick = {
        val intent = Intent(Intent.ACTION_SENDTO).apply {
            data = Uri.parse("mailto:")
            putExtra(Intent.EXTRA_EMAIL, arrayOf(email))
            putExtra(Intent.EXTRA_SUBJECT, subject)
            putExtra(Intent.EXTRA_TEXT, body)
        }
        
        try {
            context.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            Toast.makeText(context, "No email app found", Toast.LENGTH_SHORT).show()
        }
    }) {
        Icon(Icons.Default.Email, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Send Email")
    }
}
```

### Send Email with Attachment

```kotlin
fun Context.sendEmailWithAttachment(
    email: String,
    subject: String,
    body: String,
    attachmentUri: Uri
) {
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "message/rfc822"
        putExtra(Intent.EXTRA_EMAIL, arrayOf(email))
        putExtra(Intent.EXTRA_SUBJECT, subject)
        putExtra(Intent.EXTRA_TEXT, body)
        putExtra(Intent.EXTRA_STREAM, attachmentUri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    
    try {
        startActivity(Intent.createChooser(intent, "Send email"))
    } catch (e: ActivityNotFoundException) {
        Toast.makeText(this, "No email app found", Toast.LENGTH_SHORT).show()
    }
}
```

## Phone & SMS

### Make Phone Call

```kotlin
// Add permission to AndroidManifest.xml
// <uses-permission android:name="android.permission.CALL_PHONE" />

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CallPhoneButton(phoneNumber: String) {
    val context = LocalContext.current
    val permissionState = rememberPermissionState(android.Manifest.permission.CALL_PHONE)
    
    Button(onClick = {
        if (permissionState.status.isGranted) {
            val intent = Intent(Intent.ACTION_CALL, Uri.parse("tel:$phoneNumber"))
            context.startActivity(intent)
        } else {
            permissionState.launchPermissionRequest()
        }
    }) {
        Icon(Icons.Default.Phone, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Call")
    }
}
```

### Open Dialer

```kotlin
@Composable
fun OpenDialerButton(phoneNumber: String) {
    val context = LocalContext.current
    
    Button(onClick = {
        val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:$phoneNumber"))
        context.startActivity(intent)
    }) {
        Text("Open Dialer")
    }
}
```

### Send SMS

```kotlin
@Composable
fun SendSmsButton(phoneNumber: String, message: String = "") {
    val context = LocalContext.current
    
    Button(onClick = {
        val intent = Intent(Intent.ACTION_SENDTO).apply {
            data = Uri.parse("smsto:$phoneNumber")
            putExtra("sms_body", message)
        }
        
        try {
            context.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            Toast.makeText(context, "No SMS app found", Toast.LENGTH_SHORT).show()
        }
    }) {
        Icon(Icons.Default.Message, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Send SMS")
    }
}
```

## Maps & Navigation

### Open Location in Maps

```kotlin
@Composable
fun OpenMapButton(latitude: Double, longitude: Double, label: String = "") {
    val context = LocalContext.current
    
    Button(onClick = {
        val uri = if (label.isNotEmpty()) {
            Uri.parse("geo:$latitude,$longitude?q=$latitude,$longitude($label)")
        } else {
            Uri.parse("geo:$latitude,$longitude")
        }
        
        val intent = Intent(Intent.ACTION_VIEW, uri)
        
        try {
            context.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            Toast.makeText(context, "No map app found", Toast.LENGTH_SHORT).show()
        }
    }) {
        Icon(Icons.Default.Map, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Open in Maps")
    }
}
```

### Navigate to Location

```kotlin
@Composable
fun NavigateButton(latitude: Double, longitude: Double) {
    val context = LocalContext.current
    
    Button(onClick = {
        val uri = Uri.parse("google.navigation:q=$latitude,$longitude")
        val intent = Intent(Intent.ACTION_VIEW, uri)
        intent.setPackage("com.google.android.apps.maps")
        
        try {
            context.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            // Fallback to browser
            val browserUri = Uri.parse(
                "https://www.google.com/maps/dir/?api=1&destination=$latitude,$longitude"
            )
            context.startActivity(Intent(Intent.ACTION_VIEW, browserUri))
        }
    }) {
        Icon(Icons.Default.Navigation, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Navigate")
    }
}
```

### Search Place in Maps

```kotlin
fun Context.searchInMaps(query: String) {
    val uri = Uri.parse("geo:0,0?q=${Uri.encode(query)}")
    val intent = Intent(Intent.ACTION_VIEW, uri)
    startActivity(intent)
}
```

## Camera & Gallery

### Take Photo

```kotlin
@Composable
fun TakePhotoButton(onPhotoTaken: (Uri) -> Unit) {
    val context = LocalContext.current
    var photoUri by remember { mutableStateOf<Uri?>(null) }
    
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture()
    ) { success ->
        if (success && photoUri != null) {
            onPhotoTaken(photoUri!!)
        }
    }
    
    Button(onClick = {
        val uri = ComposeFileProvider.getImageUri(context)
        photoUri = uri
        launcher.launch(uri)
    }) {
        Icon(Icons.Default.Camera, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Take Photo")
    }
}
```

### Pick from Gallery

```kotlin
@Composable
fun PickImageButton(onImagePicked: (Uri) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let { onImagePicked(it) }
    }
    
    Button(onClick = {
        launcher.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
        )
    }) {
        Icon(Icons.Default.PhotoLibrary, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Pick Image")
    }
}
```

### Pick Multiple Images

```kotlin
@Composable
fun PickMultipleImagesButton(onImagesPicked: (List<Uri>) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
    ) { uris ->
        if (uris.isNotEmpty()) {
            onImagesPicked(uris)
        }
    }
    
    Button(onClick = {
        launcher.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
        )
    }) {
        Text("Pick Images (Max 5)")
    }
}
```

### Record Video

```kotlin
@Composable
fun RecordVideoButton(onVideoRecorded: (Uri) -> Unit) {
    val context = LocalContext.current
    var videoUri by remember { mutableStateOf<Uri?>(null) }
    
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CaptureVideo()
    ) { success ->
        if (success && videoUri != null) {
            onVideoRecorded(videoUri!!)
        }
    }
    
    Button(onClick = {
        val uri = ComposeFileProvider.getVideoUri(context)
        videoUri = uri
        launcher.launch(uri)
    }) {
        Icon(Icons.Default.Videocam, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Record Video")
    }
}
```

## File Picking

### Pick Any File

```kotlin
@Composable
fun PickFileButton(onFilePicked: (Uri) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent()
    ) { uri ->
        uri?.let { onFilePicked(it) }
    }
    
    Button(onClick = {
        launcher.launch("*/*")
    }) {
        Icon(Icons.Default.AttachFile, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Pick File")
    }
}
```

### Pick PDF

```kotlin
@Composable
fun PickPdfButton(onPdfPicked: (Uri) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent()
    ) { uri ->
        uri?.let { onPdfPicked(it) }
    }
    
    Button(onClick = {
        launcher.launch("application/pdf")
    }) {
        Text("Pick PDF")
    }
}
```

### Create Document

```kotlin
@Composable
fun CreateDocumentButton(onDocumentCreated: (Uri) -> Unit) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.CreateDocument("text/plain")
    ) { uri ->
        uri?.let { onDocumentCreated(it) }
    }
    
    Button(onClick = {
        launcher.launch("document.txt")
    }) {
        Text("Create Document")
    }
}
```

## App Settings

### Open App Settings

```kotlin
@Composable
fun OpenAppSettingsButton() {
    val context = LocalContext.current
    
    Button(onClick = {
        val intent = Intent(
            Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
            Uri.fromParts("package", context.packageName, null)
        )
        context.startActivity(intent)
    }) {
        Icon(Icons.Default.Settings, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("App Settings")
    }
}
```

### Open Notification Settings

```kotlin
fun Context.openNotificationSettings() {
    val intent = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).apply {
            putExtra(Settings.EXTRA_APP_PACKAGE, packageName)
        }
    } else {
        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
            data = Uri.fromParts("package", packageName, null)
        }
    }
    startActivity(intent)
}
```

### Open Location Settings

```kotlin
fun Context.openLocationSettings() {
    val intent = Intent(Settings.ACTION_LOCATION_SOURCE_SETTINGS)
    startActivity(intent)
}
```

## Other Apps

### Open Play Store

```kotlin
@Composable
fun RateAppButton() {
    val context = LocalContext.current
    
    Button(onClick = {
        val uri = Uri.parse("market://details?id=${context.packageName}")
        val intent = Intent(Intent.ACTION_VIEW, uri)
        
        try {
            context.startActivity(intent)
        } catch (e: ActivityNotFoundException) {
            // Fallback to browser
            val browserUri = Uri.parse(
                "https://play.google.com/store/apps/details?id=${context.packageName}"
            )
            context.startActivity(Intent(Intent.ACTION_VIEW, browserUri))
        }
    }) {
        Icon(Icons.Default.Star, contentDescription = null)
        Spacer(modifier = Modifier.width(8.dp))
        Text("Rate App")
    }
}
```

### Open YouTube

```kotlin
fun Context.openYouTubeVideo(videoId: String) {
    val appIntent = Intent(Intent.ACTION_VIEW, Uri.parse("vnd.youtube:$videoId"))
    val webIntent = Intent(
        Intent.ACTION_VIEW,
        Uri.parse("https://www.youtube.com/watch?v=$videoId")
    )
    
    try {
        startActivity(appIntent)
    } catch (e: ActivityNotFoundException) {
        startActivity(webIntent)
    }
}
```

### Open WhatsApp

```kotlin
fun Context.openWhatsApp(phoneNumber: String, message: String = "") {
    try {
        val intent = Intent(Intent.ACTION_VIEW).apply {
            data = Uri.parse("https://wa.me/$phoneNumber?text=${Uri.encode(message)}")
        }
        startActivity(intent)
    } catch (e: Exception) {
        Toast.makeText(this, "WhatsApp not installed", Toast.LENGTH_SHORT).show()
    }
}
```

### Open Twitter

```kotlin
fun Context.openTwitterProfile(username: String) {
    try {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse("twitter://user?screen_name=$username"))
        startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        val browserIntent = Intent(
            Intent.ACTION_VIEW,
            Uri.parse("https://twitter.com/$username")
        )
        startActivity(browserIntent)
    }
}
```

## Reusable Intent Extensions

### Context Extensions

```kotlin
fun Context.openUrl(url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    startActivity(intent)
}

fun Context.shareText(text: String, title: String = "Share") {
    val sendIntent = Intent().apply {
        action = Intent.ACTION_SEND
        putExtra(Intent.EXTRA_TEXT, text)
        type = "text/plain"
    }
    startActivity(Intent.createChooser(sendIntent, title))
}

fun Context.sendEmail(
    email: String,
    subject: String = "",
    body: String = ""
) {
    val intent = Intent(Intent.ACTION_SENDTO).apply {
        data = Uri.parse("mailto:")
        putExtra(Intent.EXTRA_EMAIL, arrayOf(email))
        putExtra(Intent.EXTRA_SUBJECT, subject)
        putExtra(Intent.EXTRA_TEXT, body)
    }
    
    try {
        startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        Toast.makeText(this, "No email app found", Toast.LENGTH_SHORT).show()
    }
}

fun Context.dialPhone(phoneNumber: String) {
    val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:$phoneNumber"))
    startActivity(intent)
}

fun Context.openMap(latitude: Double, longitude: Double, label: String = "") {
    val uri = if (label.isNotEmpty()) {
        Uri.parse("geo:$latitude,$longitude?q=$latitude,$longitude($label)")
    } else {
        Uri.parse("geo:$latitude,$longitude")
    }
    startActivity(Intent(Intent.ACTION_VIEW, uri))
}
```

## Best Practices

1. ✅ Always handle ActivityNotFoundException
2. ✅ Use Intent.createChooser() for better UX
3. ✅ Check for permissions before using intents that require them
4. ✅ Provide fallbacks when specific apps aren't installed
5. ✅ Use appropriate MIME types
6. ✅ Add FLAG_GRANT_READ_URI_PERMISSION for sharing files
7. ✅ Use FileProvider for sharing files
8. ✅ Test on different devices and Android versions
9. ✅ Handle edge cases gracefully
10. ✅ Show user feedback (Toast/Snackbar)

## Resources

- [Common Intents](https://developer.android.com/guide/components/intents-common)
- [Intent Documentation](https://developer.android.com/reference/android/content/Intent)
- [Activity Result APIs](https://developer.android.com/training/basics/intents/result)

