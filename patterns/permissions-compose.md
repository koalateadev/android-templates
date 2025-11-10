# Permissions in Jetpack Compose

## Setup

### Add Accompanist Permissions

```toml
[versions]
accompanist = "0.36.0"

[libraries]
accompanist-permissions = { group = "com.google.accompanist", name = "accompanist-permissions", version.ref = "accompanist" }
```

```kotlin
dependencies {
    implementation(libs.accompanist.permissions)
}
```

## Single Permission

### Basic Permission Request

```kotlin
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.rememberPermissionState
import com.google.accompanist.permissions.isGranted

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraPermissionScreen() {
    val cameraPermissionState = rememberPermissionState(
        android.Manifest.permission.CAMERA
    )
    
    when {
        cameraPermissionState.status.isGranted -> {
            CameraScreen()
        }
        else -> {
            PermissionRationale(
                onRequestPermission = {
                    cameraPermissionState.launchPermissionRequest()
                }
            )
        }
    }
}

@Composable
fun PermissionRationale(onRequestPermission: () -> Unit) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.Camera,
            contentDescription = null,
            modifier = Modifier.size(64.dp)
        )
        
        Spacer(modifier = Modifier.height(16.dp))
        
        Text(
            text = "Camera Permission Required",
            style = MaterialTheme.typography.titleLarge
        )
        
        Spacer(modifier = Modifier.height(8.dp))
        
        Text(
            text = "This app needs camera permission to take photos",
            style = MaterialTheme.typography.bodyMedium,
            textAlign = TextAlign.Center
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Button(onClick = onRequestPermission) {
            Text("Grant Permission")
        }
    }
}
```

### Handle Permission Denied

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraPermissionWithDenied() {
    val permissionState = rememberPermissionState(android.Manifest.permission.CAMERA)
    val context = LocalContext.current
    
    when {
        permissionState.status.isGranted -> {
            CameraScreen()
        }
        permissionState.status.shouldShowRationale -> {
            // User denied once, show rationale
            PermissionRationale(
                message = "Camera permission is needed to take photos",
                onRequestPermission = {
                    permissionState.launchPermissionRequest()
                }
            )
        }
        else -> {
            // First time or permanently denied
            Column(
                modifier = Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("Camera permission is required")
                
                Button(onClick = {
                    permissionState.launchPermissionRequest()
                }) {
                    Text("Request Permission")
                }
                
                // If permanently denied, show settings option
                if (!permissionState.status.shouldShowRationale) {
                    Spacer(modifier = Modifier.height(8.dp))
                    
                    TextButton(onClick = {
                        val intent = Intent(
                            Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
                            Uri.fromParts("package", context.packageName, null)
                        )
                        context.startActivity(intent)
                    }) {
                        Text("Open Settings")
                    }
                }
            }
        }
    }
}
```

## Multiple Permissions

### Request Multiple Permissions

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun LocationPermissions() {
    val permissionsState = rememberMultiplePermissionsState(
        permissions = listOf(
            android.Manifest.permission.ACCESS_FINE_LOCATION,
            android.Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )
    
    when {
        permissionsState.allPermissionsGranted -> {
            MapScreen()
        }
        permissionsState.shouldShowRationale -> {
            PermissionRationale(
                message = "Location permissions are needed to show your location on the map",
                onRequestPermission = {
                    permissionsState.launchMultiplePermissionRequest()
                }
            )
        }
        else -> {
            Button(onClick = {
                permissionsState.launchMultiplePermissionRequest()
            }) {
                Text("Grant Location Permissions")
            }
        }
    }
}
```

### Check Individual Permissions

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MultiplePermissionsDetailed() {
    val permissionsState = rememberMultiplePermissionsState(
        permissions = listOf(
            android.Manifest.permission.CAMERA,
            android.Manifest.permission.RECORD_AUDIO
        )
    )
    
    Column {
        permissionsState.permissions.forEach { permissionState ->
            when (permissionState.permission) {
                android.Manifest.permission.CAMERA -> {
                    PermissionStatus(
                        name = "Camera",
                        isGranted = permissionState.status.isGranted
                    )
                }
                android.Manifest.permission.RECORD_AUDIO -> {
                    PermissionStatus(
                        name = "Microphone",
                        isGranted = permissionState.status.isGranted
                    )
                }
            }
        }
        
        if (!permissionsState.allPermissionsGranted) {
            Button(onClick = {
                permissionsState.launchMultiplePermissionRequest()
            }) {
                Text("Request Permissions")
            }
        }
    }
}

@Composable
fun PermissionStatus(name: String, isGranted: Boolean) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp),
        verticalAlignment = Alignment.CenterVertically
    ) {
        Icon(
            imageVector = if (isGranted) Icons.Default.CheckCircle else Icons.Default.Cancel,
            contentDescription = null,
            tint = if (isGranted) Color.Green else Color.Red
        )
        
        Spacer(modifier = Modifier.width(8.dp))
        
        Text("$name: ${if (isGranted) "Granted" else "Not Granted"}")
    }
}
```

## Common Permissions

### Camera Permission

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CameraFeature() {
    val permissionState = rememberPermissionState(android.Manifest.permission.CAMERA)
    
    LaunchedEffect(Unit) {
        permissionState.launchPermissionRequest()
    }
    
    if (permissionState.status.isGranted) {
        CameraScreen()
    } else {
        PermissionDeniedScreen()
    }
}
```

### Location Permission

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun LocationFeature() {
    val locationPermissionsState = rememberMultiplePermissionsState(
        permissions = listOf(
            android.Manifest.permission.ACCESS_FINE_LOCATION,
            android.Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )
    
    when {
        locationPermissionsState.allPermissionsGranted -> {
            LocationScreen()
        }
        else -> {
            LocationPermissionRequest(
                onRequest = {
                    locationPermissionsState.launchMultiplePermissionRequest()
                }
            )
        }
    }
}
```

### Storage Permission (Android 13+)

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun PhotoPickerWithPermission() {
    val storagePermissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        listOf(android.Manifest.permission.READ_MEDIA_IMAGES)
    } else {
        listOf(android.Manifest.permission.READ_EXTERNAL_STORAGE)
    }
    
    val permissionsState = rememberMultiplePermissionsState(storagePermissions)
    
    when {
        permissionsState.allPermissionsGranted -> {
            PhotoPicker()
        }
        else -> {
            Button(onClick = {
                permissionsState.launchMultiplePermissionRequest()
            }) {
                Text("Grant Storage Permission")
            }
        }
    }
}
```

### Notification Permission (Android 13+)

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun NotificationPermission() {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        val permissionState = rememberPermissionState(
            android.Manifest.permission.POST_NOTIFICATIONS
        )
        
        LaunchedEffect(Unit) {
            if (!permissionState.status.isGranted) {
                permissionState.launchPermissionRequest()
            }
        }
        
        if (!permissionState.status.isGranted) {
            NotificationPermissionRationale(
                onRequest = { permissionState.launchPermissionRequest() }
            )
        }
    }
}
```

## Image Picker with Permission

### Complete Image Picker

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun ImagePickerScreen() {
    var selectedImageUri by remember { mutableStateOf<Uri?>(null) }
    var showBottomSheet by remember { mutableStateOf(false) }
    
    // Camera permission
    val cameraPermissionState = rememberPermissionState(
        android.Manifest.permission.CAMERA
    )
    
    // Storage permission
    val storagePermissions = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        listOf(android.Manifest.permission.READ_MEDIA_IMAGES)
    } else {
        listOf(android.Manifest.permission.READ_EXTERNAL_STORAGE)
    }
    
    val storagePermissionsState = rememberMultiplePermissionsState(storagePermissions)
    
    // Camera launcher
    val cameraImageUri = remember { mutableStateOf<Uri?>(null) }
    val cameraLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture()
    ) { success ->
        if (success) {
            selectedImageUri = cameraImageUri.value
        }
    }
    
    // Gallery launcher
    val galleryLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let { selectedImageUri = it }
    }
    
    val context = LocalContext.current
    
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // Show selected image
        selectedImageUri?.let { uri ->
            AsyncImage(
                model = uri,
                contentDescription = "Selected image",
                modifier = Modifier
                    .size(200.dp)
                    .clip(RoundedCornerShape(8.dp))
            )
            
            Spacer(modifier = Modifier.height(16.dp))
        }
        
        Button(onClick = { showBottomSheet = true }) {
            Text("Pick Image")
        }
    }
    
    if (showBottomSheet) {
        ModalBottomSheet(
            onDismissRequest = { showBottomSheet = false }
        ) {
            Column(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp)
            ) {
                // Camera option
                TextButton(
                    onClick = {
                        showBottomSheet = false
                        if (cameraPermissionState.status.isGranted) {
                            val uri = ComposeFileProvider.getImageUri(context)
                            cameraImageUri.value = uri
                            cameraLauncher.launch(uri)
                        } else {
                            cameraPermissionState.launchPermissionRequest()
                        }
                    },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Icon(Icons.Default.Camera, contentDescription = null)
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("Take Photo")
                }
                
                // Gallery option
                TextButton(
                    onClick = {
                        showBottomSheet = false
                        if (storagePermissionsState.allPermissionsGranted) {
                            galleryLauncher.launch(
                                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
                            )
                        } else {
                            storagePermissionsState.launchMultiplePermissionRequest()
                        }
                    },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Icon(Icons.Default.PhotoLibrary, contentDescription = null)
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("Choose from Gallery")
                }
            }
        }
    }
}

// File provider for camera
object ComposeFileProvider {
    fun getImageUri(context: Context): Uri {
        val directory = File(context.cacheDir, "images")
        directory.mkdirs()
        val file = File.createTempFile(
            "selected_image_",
            ".jpg",
            directory
        )
        val authority = "${context.packageName}.fileprovider"
        return FileProvider.getUriForFile(
            context,
            authority,
            file
        )
    }
}
```

Don't forget to add FileProvider to `AndroidManifest.xml`:

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

Create `res/xml/file_paths.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="images" path="images/"/>
</paths>
```

## Check Permission Status

### Check if Granted

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun CheckPermissionStatus() {
    val permissionState = rememberPermissionState(android.Manifest.permission.CAMERA)
    
    val status = when {
        permissionState.status.isGranted -> "Granted"
        permissionState.status.shouldShowRationale -> "Denied (Can show rationale)"
        else -> "Denied (First time or permanently)"
    }
    
    Text("Permission status: $status")
}
```

## Reusable Permission Components

### Permission Wrapper

```kotlin
@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun WithPermission(
    permission: String,
    rationaleMessage: String,
    content: @Composable () -> Unit
) {
    val permissionState = rememberPermissionState(permission)
    
    when {
        permissionState.status.isGranted -> {
            content()
        }
        else -> {
            PermissionRationale(
                message = rationaleMessage,
                onRequestPermission = {
                    permissionState.launchPermissionRequest()
                }
            )
        }
    }
}

// Usage
WithPermission(
    permission = android.Manifest.permission.CAMERA,
    rationaleMessage = "Camera permission is needed to take photos"
) {
    CameraScreen()
}
```

### Permission ViewModel

```kotlin
class PermissionViewModel : ViewModel() {
    private val _permissionGranted = MutableStateFlow(false)
    val permissionGranted: StateFlow<Boolean> = _permissionGranted.asStateFlow()
    
    fun onPermissionResult(granted: Boolean) {
        _permissionGranted.value = granted
    }
}
```

## Best Practices

- Provide clear rationale for permissions
- Request permissions contextually
- Handle all permission states
- Guide users to settings when permanently denied
- Test on different Android versions
- Request minimum necessary permissions
- Degrade gracefully when denied

## Common Permissions List

```kotlin
object AppPermissions {
    // Location
    const val FINE_LOCATION = android.Manifest.permission.ACCESS_FINE_LOCATION
    const val COARSE_LOCATION = android.Manifest.permission.ACCESS_COARSE_LOCATION
    
    // Camera & Storage
    const val CAMERA = android.Manifest.permission.CAMERA
    const val READ_STORAGE = android.Manifest.permission.READ_EXTERNAL_STORAGE
    const val WRITE_STORAGE = android.Manifest.permission.WRITE_EXTERNAL_STORAGE
    
    // Android 13+
    const val READ_IMAGES = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        android.Manifest.permission.READ_MEDIA_IMAGES
    } else {
        READ_STORAGE
    }
    
    const val POST_NOTIFICATIONS = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        android.Manifest.permission.POST_NOTIFICATIONS
    } else {
        "" // Not needed before Android 13
    }
    
    // Audio
    const val RECORD_AUDIO = android.Manifest.permission.RECORD_AUDIO
    
    // Contacts
    const val READ_CONTACTS = android.Manifest.permission.READ_CONTACTS
    const val WRITE_CONTACTS = android.Manifest.permission.WRITE_CONTACTS
    
    // Phone
    const val CALL_PHONE = android.Manifest.permission.CALL_PHONE
    const val READ_PHONE_STATE = android.Manifest.permission.READ_PHONE_STATE
}
```

## Resources

- [Accompanist Permissions](https://google.github.io/accompanist/permissions/)
- [Android Permissions Guide](https://developer.android.com/training/permissions/requesting)
- [Permission Best Practices](https://developer.android.com/training/permissions/usage-notes)

