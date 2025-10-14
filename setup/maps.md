# Google Maps & Mapbox Setup

## Google Maps Setup

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
play-services-maps = "19.0.0"
maps-compose = "6.2.1"
maps-utils = "3.8.2"

[libraries]
play-services-maps = { group = "com.google.android.gms", name = "play-services-maps", version.ref = "play-services-maps" }
maps-compose = { group = "com.google.maps.android", name = "maps-compose", version.ref = "maps-compose" }
maps-utils = { group = "com.google.maps.android", name = "android-maps-utils", version.ref = "maps-utils" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    // Google Maps
    implementation(libs.play.services.maps)
    
    // Maps Compose (for Jetpack Compose)
    implementation(libs.maps.compose)
    
    // Maps Utils (clustering, geojson, etc.)
    implementation(libs.maps.utils)
}
```

### 3. Get API Key

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project or select existing
3. Enable **Maps SDK for Android**
4. Create API key in Credentials

### 4. Add API Key to AndroidManifest.xml

```xml
<manifest>
    <application>
        <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="${MAPS_API_KEY}" />
    </application>
</manifest>
```

In `gradle.properties`:
```properties
MAPS_API_KEY=your_api_key_here
```

In `build.gradle.kts`:
```kotlin
android {
    defaultConfig {
        manifestPlaceholders["MAPS_API_KEY"] = project.findProperty("MAPS_API_KEY") as String? ?: ""
    }
}
```

### 5. Google Maps in Compose

```kotlin
import com.google.android.gms.maps.model.CameraPosition
import com.google.android.gms.maps.model.LatLng
import com.google.maps.android.compose.*

@Composable
fun MapScreen() {
    val singapore = LatLng(1.35, 103.87)
    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(singapore, 10f)
    }
    
    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        Marker(
            state = MarkerState(position = singapore),
            title = "Singapore",
            snippet = "Marker in Singapore"
        )
    }
}
```

### 6. Advanced Maps Features

```kotlin
@Composable
fun AdvancedMapScreen() {
    val uiSettings = remember {
        MapUiSettings(
            zoomControlsEnabled = true,
            compassEnabled = true,
            myLocationButtonEnabled = true,
            mapToolbarEnabled = true
        )
    }
    
    val properties = remember {
        MapProperties(
            mapType = MapType.NORMAL,  // SATELLITE, TERRAIN, HYBRID
            isMyLocationEnabled = true,
            minZoomPreference = 5f,
            maxZoomPreference = 20f
        )
    }
    
    val cameraPositionState = rememberCameraPositionState()
    
    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState,
        properties = properties,
        uiSettings = uiSettings,
        onMapClick = { latLng ->
            Log.d("Map", "Clicked: $latLng")
        },
        onMapLongClick = { latLng ->
            Log.d("Map", "Long clicked: $latLng")
        }
    ) {
        // Markers
        Marker(
            state = MarkerState(position = LatLng(1.35, 103.87)),
            title = "Marker 1"
        )
        
        // Circle
        Circle(
            center = LatLng(1.35, 103.87),
            radius = 1000.0,
            strokeColor = Color.Red,
            fillColor = Color.Red.copy(alpha = 0.3f)
        )
        
        // Polyline
        Polyline(
            points = listOf(
                LatLng(1.35, 103.87),
                LatLng(1.36, 103.88),
                LatLng(1.37, 103.89)
            ),
            color = Color.Blue,
            width = 10f
        )
        
        // Polygon
        Polygon(
            points = listOf(
                LatLng(1.35, 103.87),
                LatLng(1.36, 103.88),
                LatLng(1.37, 103.87)
            ),
            strokeColor = Color.Green,
            fillColor = Color.Green.copy(alpha = 0.3f)
        )
    }
}
```

### 7. Camera Movement

```kotlin
@Composable
fun MapWithControls() {
    val cameraPositionState = rememberCameraPositionState()
    val coroutineScope = rememberCoroutineScope()
    
    Column {
        GoogleMap(
            modifier = Modifier
                .fillMaxWidth()
                .weight(1f),
            cameraPositionState = cameraPositionState
        )
        
        Row {
            Button(onClick = {
                coroutineScope.launch {
                    cameraPositionState.animate(
                        update = CameraUpdateFactory.newLatLngZoom(
                            LatLng(1.35, 103.87),
                            15f
                        ),
                        durationMs = 1000
                    )
                }
            }) {
                Text("Animate to Location")
            }
            
            Button(onClick = {
                coroutineScope.launch {
                    cameraPositionState.move(
                        CameraUpdateFactory.zoomIn()
                    )
                }
            }) {
                Text("Zoom In")
            }
        }
    }
}
```

### 8. Marker Clustering

```kotlin
import com.google.maps.android.clustering.ClusterItem
import com.google.maps.android.compose.clustering.Clustering

data class MyClusterItem(
    val itemPosition: LatLng,
    val itemTitle: String,
    val itemSnippet: String
) : ClusterItem {
    override fun getPosition() = itemPosition
    override fun getTitle() = itemTitle
    override fun getSnippet() = itemSnippet
    override fun getZIndex(): Float = 0f
}

@Composable
fun ClusteredMapScreen() {
    val items = remember {
        List(100) { index ->
            MyClusterItem(
                itemPosition = LatLng(
                    1.35 + (Math.random() - 0.5) * 0.1,
                    103.87 + (Math.random() - 0.5) * 0.1
                ),
                itemTitle = "Marker $index",
                itemSnippet = "Snippet $index"
            )
        }
    }
    
    GoogleMap(
        modifier = Modifier.fillMaxSize()
    ) {
        Clustering(
            items = items,
            onClusterClick = { cluster ->
                Log.d("Map", "Cluster clicked: ${cluster.size}")
                false
            },
            onClusterItemClick = { item ->
                Log.d("Map", "Item clicked: ${item.itemTitle}")
                false
            }
        )
    }
}
```

## Mapbox Setup

### 1. Add to Version Catalog

```toml
[versions]
mapbox = "11.7.1"

[libraries]
mapbox-android = { group = "com.mapbox.maps", name = "android", version.ref = "mapbox" }
mapbox-compose = { group = "com.mapbox.extension", name = "maps-compose", version = "11.7.1" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.mapbox.android)
    implementation(libs.mapbox.compose)
}
```

### 3. Add Maven Repository

In `settings.gradle.kts`:
```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven {
            url = uri("https://api.mapbox.com/downloads/v2/releases/maven")
            credentials {
                username = "mapbox"
                password = providers.gradleProperty("MAPBOX_DOWNLOADS_TOKEN").orNull
                    ?: System.getenv("MAPBOX_DOWNLOADS_TOKEN")
            }
        }
    }
}
```

### 4. Get Mapbox Token

1. Create account at [Mapbox](https://www.mapbox.com/)
2. Get public and secret tokens from account dashboard
3. Add to `gradle.properties`:

```properties
MAPBOX_DOWNLOADS_TOKEN=sk.ey...
```

### 5. Add Token to Manifest

```xml
<resources>
    <string name="mapbox_access_token">pk.ey...</string>
</resources>
```

### 6. Mapbox in Compose

```kotlin
import com.mapbox.geojson.Point
import com.mapbox.maps.extension.compose.MapboxMap
import com.mapbox.maps.extension.compose.animation.viewport.rememberMapViewportState

@Composable
fun MapboxMapScreen() {
    val mapViewportState = rememberMapViewportState {
        setCameraOptions {
            center(Point.fromLngLat(103.87, 1.35))
            zoom(10.0)
        }
    }
    
    MapboxMap(
        modifier = Modifier.fillMaxSize(),
        mapViewportState = mapViewportState
    )
}
```

### 7. Add Markers and Annotations

```kotlin
import com.mapbox.maps.extension.compose.annotation.generated.PointAnnotation

@Composable
fun MapboxWithMarkers() {
    val mapViewportState = rememberMapViewportState()
    
    MapboxMap(
        modifier = Modifier.fillMaxSize(),
        mapViewportState = mapViewportState
    ) {
        PointAnnotation(
            point = Point.fromLngLat(103.87, 1.35),
            iconImageBitmap = ImageBitmap.imageResource(R.drawable.ic_marker),
            onClick = {
                Log.d("Mapbox", "Marker clicked")
                true
            }
        )
    }
}
```

## Permissions

### Add Location Permissions

```xml
<manifest>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
</manifest>
```

### Request Permissions

```kotlin
import com.google.accompanist.permissions.ExperimentalPermissionsApi
import com.google.accompanist.permissions.rememberMultiplePermissionsState

@OptIn(ExperimentalPermissionsApi::class)
@Composable
fun MapWithPermissions() {
    val permissionsState = rememberMultiplePermissionsState(
        permissions = listOf(
            Manifest.permission.ACCESS_FINE_LOCATION,
            Manifest.permission.ACCESS_COARSE_LOCATION
        )
    )
    
    LaunchedEffect(Unit) {
        permissionsState.launchMultiplePermissionRequest()
    }
    
    when {
        permissionsState.allPermissionsGranted -> {
            MapScreen()
        }
        else -> {
            PermissionRationale {
                permissionsState.launchMultiplePermissionRequest()
            }
        }
    }
}
```

## Location Services

### Get Current Location

```kotlin
import com.google.android.gms.location.*

@Composable
fun MapWithLocation() {
    val context = LocalContext.current
    val fusedLocationClient = remember {
        LocationServices.getFusedLocationProviderClient(context)
    }
    var currentLocation by remember { mutableStateOf<LatLng?>(null) }
    
    LaunchedEffect(Unit) {
        try {
            fusedLocationClient.lastLocation.addOnSuccessListener { location ->
                location?.let {
                    currentLocation = LatLng(it.latitude, it.longitude)
                }
            }
        } catch (e: SecurityException) {
            // Handle permission denied
        }
    }
    
    currentLocation?.let { location ->
        GoogleMap(
            modifier = Modifier.fillMaxSize()
        ) {
            Marker(
                state = MarkerState(position = location),
                title = "You are here"
            )
        }
    }
}
```

## Best Practices

1. ✅ Always request location permissions
2. ✅ Handle permission denials gracefully
3. ✅ Use appropriate zoom levels
4. ✅ Cluster markers when showing many points
5. ✅ Cache map tiles for offline use
6. ✅ Use styled maps for branding
7. ✅ Handle map lifecycle properly
8. ✅ Optimize marker icons
9. ✅ Test on different screen sizes
10. ✅ Secure API keys properly

## Google Maps vs Mapbox

| Feature | Google Maps | Mapbox |
|---------|-------------|---------|
| Free Tier | 25,000 loads/month | 50,000 loads/month |
| Customization | Limited | Extensive |
| Offline Maps | Limited | Full support |
| Pricing | Pay per use | Flexible plans |
| Global Coverage | Excellent | Excellent |
| Custom Styles | Limited | Advanced |

## Resources

### Google Maps
- [Official Documentation](https://developers.google.com/maps/documentation/android-sdk)
- [Maps Compose](https://github.com/googlemaps/android-maps-compose)
- [Code Samples](https://github.com/googlemaps/android-samples)

### Mapbox
- [Official Documentation](https://docs.mapbox.com/android/maps/guides/)
- [Mapbox Studio](https://studio.mapbox.com/)
- [Examples](https://docs.mapbox.com/android/maps/examples/)

