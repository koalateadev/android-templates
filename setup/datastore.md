# DataStore Setup

## Overview
DataStore is a modern data storage solution that replaces SharedPreferences with a safer, more robust API using Kotlin Coroutines and Flow.

## Types of DataStore

1. **Preferences DataStore** - Key-value pairs (like SharedPreferences)
2. **Proto DataStore** - Typed objects with Protocol Buffers

## Preferences DataStore Setup

### 1. Add to Version Catalog (`gradle/libs.versions.toml`)

```toml
[versions]
datastore = "1.1.1"

[libraries]
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
androidx-datastore-core = { group = "androidx.datastore", name = "datastore-core", version.ref = "datastore" }
```

### 2. Add Dependencies

```kotlin
dependencies {
    implementation(libs.androidx.datastore.preferences)
}
```

### 3. Create DataStore Instance

```kotlin
import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.*
import androidx.datastore.preferences.preferencesDataStore

// Create DataStore instance (top-level)
private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

class PreferencesManager(private val context: Context) {
    
    companion object {
        // Define keys
        val USER_NAME = stringPreferencesKey("user_name")
        val USER_AGE = intPreferencesKey("user_age")
        val IS_LOGGED_IN = booleanPreferencesKey("is_logged_in")
        val THEME_MODE = stringPreferencesKey("theme_mode")
    }
    
    // Read value
    val userName: Flow<String?> = context.dataStore.data
        .map { preferences ->
            preferences[USER_NAME]
        }
        .catch { exception ->
            if (exception is IOException) {
                emit(null)
            } else {
                throw exception
            }
        }
    
    // Write value
    suspend fun saveUserName(name: String) {
        context.dataStore.edit { preferences ->
            preferences[USER_NAME] = name
        }
    }
    
    // Read with default value
    val themeMode: Flow<String> = context.dataStore.data
        .map { preferences ->
            preferences[THEME_MODE] ?: "system"
        }
    
    // Update multiple values
    suspend fun updateUserInfo(name: String, age: Int) {
        context.dataStore.edit { preferences ->
            preferences[USER_NAME] = name
            preferences[USER_AGE] = age
        }
    }
    
    // Clear all data
    suspend fun clearAll() {
        context.dataStore.edit { preferences ->
            preferences.clear()
        }
    }
    
    // Remove specific key
    suspend fun removeUserName() {
        context.dataStore.edit { preferences ->
            preferences.remove(USER_NAME)
        }
    }
}
```

### 4. Use with Hilt

```kotlin
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {
    
    @Provides
    @Singleton
    fun providePreferencesManager(
        @ApplicationContext context: Context
    ): PreferencesManager {
        return PreferencesManager(context)
    }
}
```

### 5. Use in ViewModel

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val preferencesManager: PreferencesManager
) : ViewModel() {
    
    val userName: StateFlow<String?> = preferencesManager.userName
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = null
        )
    
    val themeMode: StateFlow<String> = preferencesManager.themeMode
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = "system"
        )
    
    fun saveUserName(name: String) {
        viewModelScope.launch {
            preferencesManager.saveUserName(name)
        }
    }
    
    fun saveThemeMode(mode: String) {
        viewModelScope.launch {
            preferencesManager.context.dataStore.edit { preferences ->
                preferences[PreferencesManager.THEME_MODE] = mode
            }
        }
    }
}
```

### 6. Use in Composable

```kotlin
@Composable
fun SettingsScreen(viewModel: SettingsViewModel = hiltViewModel()) {
    val userName by viewModel.userName.collectAsState()
    val themeMode by viewModel.themeMode.collectAsState()
    
    Column {
        Text("User: ${userName ?: "Not set"}")
        Text("Theme: $themeMode")
        
        Button(onClick = { viewModel.saveUserName("John Doe") }) {
            Text("Save Name")
        }
    }
}
```

## Proto DataStore Setup

### 1. Add Proto Dependencies

```toml
[versions]
protobuf = "4.28.2"

[libraries]
androidx-datastore-core = { group = "androidx.datastore", name = "datastore-core", version.ref = "datastore" }
protobuf-javalite = { group = "com.google.protobuf", name = "protobuf-javalite", version.ref = "protobuf" }

[plugins]
protobuf = { id = "com.google.protobuf", version = "0.9.4" }
```

```kotlin
plugins {
    alias(libs.plugins.protobuf)
}

dependencies {
    implementation(libs.androidx.datastore.core)
    implementation(libs.protobuf.javalite)
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:4.28.2"
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                create("java") {
                    option("lite")
                }
            }
        }
    }
}
```

### 2. Define Proto Schema

Create `app/src/main/proto/settings.proto`:

```protobuf
syntax = "proto3";

option java_package = "com.example.app";
option java_multiple_files = true;

message UserPreferences {
  string name = 1;
  int32 age = 2;
  bool is_premium = 3;
  string theme_mode = 4;
  int64 last_sync_time = 5;
}
```

### 3. Create Serializer

```kotlin
import androidx.datastore.core.Serializer
import com.example.app.UserPreferences
import java.io.InputStream
import java.io.OutputStream

object UserPreferencesSerializer : Serializer<UserPreferences> {
    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()
    
    override suspend fun readFrom(input: InputStream): UserPreferences {
        return try {
            UserPreferences.parseFrom(input)
        } catch (exception: Exception) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }
    
    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}
```

### 4. Create DataStore Instance

```kotlin
import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.dataStore

private val Context.userPreferencesStore: DataStore<UserPreferences> by dataStore(
    fileName = "user_preferences.pb",
    serializer = UserPreferencesSerializer
)

class UserPreferencesRepository(private val context: Context) {
    
    val userPreferences: Flow<UserPreferences> = context.userPreferencesStore.data
        .catch { exception ->
            if (exception is IOException) {
                emit(UserPreferences.getDefaultInstance())
            } else {
                throw exception
            }
        }
    
    suspend fun updateName(name: String) {
        context.userPreferencesStore.updateData { currentPrefs ->
            currentPrefs.toBuilder()
                .setName(name)
                .build()
        }
    }
    
    suspend fun updateUserInfo(name: String, age: Int, isPremium: Boolean) {
        context.userPreferencesStore.updateData { currentPrefs ->
            currentPrefs.toBuilder()
                .setName(name)
                .setAge(age)
                .setIsPremium(isPremium)
                .build()
        }
    }
    
    suspend fun updateTheme(themeMode: String) {
        context.userPreferencesStore.updateData { currentPrefs ->
            currentPrefs.toBuilder()
                .setThemeMode(themeMode)
                .build()
        }
    }
    
    suspend fun clearAll() {
        context.userPreferencesStore.updateData {
            UserPreferences.getDefaultInstance()
        }
    }
}
```

## Common Patterns

### Preferences with Default Values

```kotlin
class SettingsRepository(private val context: Context) {
    
    companion object {
        val NOTIFICATION_ENABLED = booleanPreferencesKey("notifications_enabled")
        val LANGUAGE = stringPreferencesKey("language")
        val FONT_SIZE = floatPreferencesKey("font_size")
    }
    
    data class Settings(
        val notificationsEnabled: Boolean = true,
        val language: String = "en",
        val fontSize: Float = 16.0f
    )
    
    val settings: Flow<Settings> = context.dataStore.data
        .catch { exception ->
            if (exception is IOException) {
                emit(emptyPreferences())
            } else {
                throw exception
            }
        }
        .map { preferences ->
            Settings(
                notificationsEnabled = preferences[NOTIFICATION_ENABLED] ?: true,
                language = preferences[LANGUAGE] ?: "en",
                fontSize = preferences[FONT_SIZE] ?: 16.0f
            )
        }
    
    suspend fun updateSettings(settings: Settings) {
        context.dataStore.edit { preferences ->
            preferences[NOTIFICATION_ENABLED] = settings.notificationsEnabled
            preferences[LANGUAGE] = settings.language
            preferences[FONT_SIZE] = settings.fontSize
        }
    }
}
```

### Migration from SharedPreferences

```kotlin
import androidx.datastore.migrations.SharedPreferencesMigration
import androidx.datastore.migrations.SharedPreferencesView

private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(
    name = "settings",
    produceMigrations = { context ->
        listOf(
            SharedPreferencesMigration(
                context,
                "old_shared_prefs_name"
            ) { sharedPrefs: SharedPreferencesView, currentData: Preferences ->
                // Map old SharedPreferences keys to new DataStore keys
                if (currentData[PreferencesManager.USER_NAME] == null) {
                    val userName = sharedPrefs.getString("user_name", null)
                    if (userName != null) {
                        currentData.toMutablePreferences().apply {
                            this[PreferencesManager.USER_NAME] = userName
                        }
                    } else {
                        currentData
                    }
                } else {
                    currentData
                }
            }
        )
    }
)
```

### Theme Management

```kotlin
enum class ThemeMode {
    LIGHT, DARK, SYSTEM
}

class ThemeRepository(private val context: Context) {
    companion object {
        val THEME_MODE = stringPreferencesKey("theme_mode")
    }
    
    val themeMode: Flow<ThemeMode> = context.dataStore.data
        .map { preferences ->
            when (preferences[THEME_MODE]) {
                "light" -> ThemeMode.LIGHT
                "dark" -> ThemeMode.DARK
                else -> ThemeMode.SYSTEM
            }
        }
    
    suspend fun setThemeMode(mode: ThemeMode) {
        context.dataStore.edit { preferences ->
            preferences[THEME_MODE] = mode.name.lowercase()
        }
    }
}

// In ViewModel
@HiltViewModel
class ThemeViewModel @Inject constructor(
    private val themeRepository: ThemeRepository
) : ViewModel() {
    
    val themeMode: StateFlow<ThemeMode> = themeRepository.themeMode
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.Eagerly,
            initialValue = ThemeMode.SYSTEM
        )
    
    fun setThemeMode(mode: ThemeMode) {
        viewModelScope.launch {
            themeRepository.setThemeMode(mode)
        }
    }
}

// In Composable
@Composable
fun App(themeViewModel: ThemeViewModel = hiltViewModel()) {
    val themeMode by themeViewModel.themeMode.collectAsState()
    
    AppTheme(darkTheme = when (themeMode) {
        ThemeMode.LIGHT -> false
        ThemeMode.DARK -> true
        ThemeMode.SYSTEM -> isSystemInDarkTheme()
    }) {
        // App content
    }
}
```

### User Session Management

```kotlin
data class UserSession(
    val userId: String? = null,
    val accessToken: String? = null,
    val refreshToken: String? = null,
    val expiresAt: Long = 0L
) {
    val isLoggedIn: Boolean
        get() = userId != null && accessToken != null
    
    val isTokenExpired: Boolean
        get() = System.currentTimeMillis() > expiresAt
}

class SessionRepository(private val context: Context) {
    companion object {
        val USER_ID = stringPreferencesKey("user_id")
        val ACCESS_TOKEN = stringPreferencesKey("access_token")
        val REFRESH_TOKEN = stringPreferencesKey("refresh_token")
        val EXPIRES_AT = longPreferencesKey("expires_at")
    }
    
    val userSession: Flow<UserSession> = context.dataStore.data
        .map { preferences ->
            UserSession(
                userId = preferences[USER_ID],
                accessToken = preferences[ACCESS_TOKEN],
                refreshToken = preferences[REFRESH_TOKEN],
                expiresAt = preferences[EXPIRES_AT] ?: 0L
            )
        }
    
    suspend fun saveSession(
        userId: String,
        accessToken: String,
        refreshToken: String,
        expiresIn: Long
    ) {
        context.dataStore.edit { preferences ->
            preferences[USER_ID] = userId
            preferences[ACCESS_TOKEN] = accessToken
            preferences[REFRESH_TOKEN] = refreshToken
            preferences[EXPIRES_AT] = System.currentTimeMillis() + (expiresIn * 1000)
        }
    }
    
    suspend fun clearSession() {
        context.dataStore.edit { preferences ->
            preferences.remove(USER_ID)
            preferences.remove(ACCESS_TOKEN)
            preferences.remove(REFRESH_TOKEN)
            preferences.remove(EXPIRES_AT)
        }
    }
}
```

## Testing

### Test with Fake DataStore

```kotlin
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.PreferenceDataStoreFactory
import androidx.datastore.preferences.core.Preferences
import kotlinx.coroutines.test.TestScope
import org.junit.Rule
import org.junit.rules.TemporaryFolder
import java.io.File

class PreferencesManagerTest {
    
    @get:Rule
    val tmpFolder: TemporaryFolder = TemporaryFolder.builder().assureDeletion().build()
    
    private lateinit var testDataStore: DataStore<Preferences>
    private lateinit var preferencesManager: PreferencesManager
    
    private fun createDataStore(testScope: TestScope): DataStore<Preferences> {
        return PreferenceDataStoreFactory.create(
            scope = testScope,
            produceFile = { File(tmpFolder.newFolder(), "test.preferences_pb") }
        )
    }
    
    @Test
    fun `save and read user name`() = runTest {
        testDataStore = createDataStore(this)
        // Create PreferencesManager with test DataStore
        
        // Save
        testDataStore.edit { it[PreferencesManager.USER_NAME] = "John" }
        
        // Read
        val userName = testDataStore.data.first()[PreferencesManager.USER_NAME]
        
        assertEquals("John", userName)
    }
}
```

## Best Practices

1. ✅ Use DataStore instead of SharedPreferences
2. ✅ Access DataStore only from Repositories
3. ✅ Expose Flow/StateFlow to ViewModels
4. ✅ Use Proto DataStore for complex data structures
5. ✅ Handle IOExceptions when reading data
6. ✅ Use structured concurrency (viewModelScope)
7. ✅ Migrate from SharedPreferences with migrations
8. ✅ Don't store large datasets (use Room instead)
9. ✅ Keep keys organized in companion objects
10. ✅ Write comprehensive tests

## DataStore vs SharedPreferences

| Feature | DataStore | SharedPreferences |
|---------|-----------|-------------------|
| API | Coroutines + Flow | Synchronous |
| Thread Safety | ✅ Yes | ❌ No |
| Type Safety | ✅ Better | ❌ Limited |
| Error Handling | ✅ Exceptions | ❌ Runtime crashes |
| Data Consistency | ✅ ACID | ❌ Can corrupt |
| Performance | ✅ Better | ❌ Blocks UI |

## Resources

- [Official Documentation](https://developer.android.com/topic/libraries/architecture/datastore)
- [DataStore Codelab](https://developer.android.com/codelabs/android-datastore)
- [Protocol Buffers](https://developers.google.com/protocol-buffers)

