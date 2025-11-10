# Security Patterns

## Table of Contents
- [Secure Data Storage](#secure-data-storage)
- [API Key Management](#api-key-management)
- [Certificate Pinning](#certificate-pinning)
- [Token Management](#token-management)
- [Biometric Authentication](#biometric-authentication)
- [Data Encryption](#data-encryption)
- [ProGuard Security](#proguard-security)
- [Network Security](#network-security)

## Secure Data Storage

### EncryptedSharedPreferences

```kotlin
import androidx.security.crypto.EncryptedSharedPreferences
import androidx.security.crypto.MasterKey

class SecurePreferences(private val context: Context) {
    
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    private val sharedPreferences = EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
    
    fun saveToken(token: String) {
        sharedPreferences.edit {
            putString("auth_token", token)
        }
    }
    
    fun getToken(): String? {
        return sharedPreferences.getString("auth_token", null)
    }
    
    fun saveCredentials(username: String, password: String) {
        sharedPreferences.edit {
            putString("username", username)
            putString("password", password)
        }
    }
    
    fun clearAll() {
        sharedPreferences.edit { clear() }
    }
}
```

### Dependencies

Add to `gradle/libs.versions.toml`:
```toml
[versions]
security-crypto = "1.1.0-alpha06"

[libraries]
androidx-security-crypto = { group = "androidx.security", name = "security-crypto", version.ref = "security-crypto" }
```

### EncryptedFile

```kotlin
import androidx.security.crypto.EncryptedFile
import java.io.File

class SecureFileStorage(private val context: Context) {
    
    private val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()
    
    fun writeSecureFile(fileName: String, content: String) {
        val file = File(context.filesDir, fileName)
        val encryptedFile = EncryptedFile.Builder(
            context,
            file,
            masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build()
        
        encryptedFile.openFileOutput().use { outputStream ->
            outputStream.write(content.toByteArray())
        }
    }
    
    fun readSecureFile(fileName: String): String {
        val file = File(context.filesDir, fileName)
        val encryptedFile = EncryptedFile.Builder(
            context,
            file,
            masterKey,
            EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
        ).build()
        
        return encryptedFile.openFileInput().use { inputStream ->
            inputStream.readBytes().toString(Charsets.UTF_8)
        }
    }
}
```

## API Key Management

### BuildConfig (Not Secure - For Development)

```kotlin
android {
    buildTypes {
        debug {
            buildConfigField("String", "API_KEY", "\"debug_key_12345\"")
        }
        release {
            buildConfigField("String", "API_KEY", "\"${System.getenv("API_KEY")}\"")
        }
    }
}

// Usage
val apiKey = BuildConfig.API_KEY
```

### NDK/Native Code (More Secure)

Create `secrets.cpp`:
```cpp
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_app_SecureKeys_getApiKey(JNIEnv* env, jobject) {
    return env->NewStringUTF("your_api_key_here");
}
```

Kotlin wrapper:
```kotlin
object SecureKeys {
    init {
        System.loadLibrary("secrets")
    }
    
    external fun getApiKey(): String
}

// Usage
val apiKey = SecureKeys.getApiKey()
```

### Keys from Secure Backend

```kotlin
class KeyManager(
    private val securePreferences: SecurePreferences,
    private val apiService: ApiService
) {
    suspend fun getApiKey(): String {
        // Try to get from secure storage
        var key = securePreferences.getToken()
        
        if (key.isNullOrEmpty()) {
            // Fetch from backend
            key = apiService.fetchEncryptedKey()
            securePreferences.saveToken(key)
        }
        
        return key
    }
}
```

## Certificate Pinning

### OkHttp Certificate Pinner

```kotlin
import okhttp3.CertificatePinner
import okhttp3.OkHttpClient

val certificatePinner = CertificatePinner.Builder()
    .add("api.example.com", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
    .add("api.example.com", "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=")
    .build()

val client = OkHttpClient.Builder()
    .certificatePinner(certificatePinner)
    .build()
```

### Get Certificate Hash

```bash
# Get certificate from server
openssl s_client -connect api.example.com:443 | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | openssl enc -base64
```

### Network Security Config

Create `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-01-01">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
</network-security-config>
```

In `AndroidManifest.xml`:
```xml
<application
    android:networkSecurityConfig="@xml/network_security_config">
```

## Token Management

### JWT Token Handling

```kotlin
data class TokenPair(
    val accessToken: String,
    val refreshToken: String,
    val expiresAt: Long
)

class TokenManager(
    private val securePreferences: SecurePreferences,
    private val apiService: ApiService
) {
    
    suspend fun getValidAccessToken(): String? {
        val token = getAccessToken() ?: return null
        
        return if (isTokenExpired(token)) {
            refreshAccessToken()
        } else {
            token
        }
    }
    
    fun getAccessToken(): String? {
        return securePreferences.getString("access_token")
    }
    
    fun getRefreshToken(): String? {
        return securePreferences.getString("refresh_token")
    }
    
    fun saveTokens(accessToken: String, refreshToken: String, expiresIn: Long) {
        val expiresAt = System.currentTimeMillis() + (expiresIn * 1000)
        securePreferences.apply {
            putString("access_token", accessToken)
            putString("refresh_token", refreshToken)
            putLong("expires_at", expiresAt)
        }
    }
    
    private fun isTokenExpired(token: String): Boolean {
        val expiresAt = securePreferences.getLong("expires_at", 0)
        return System.currentTimeMillis() >= expiresAt
    }
    
    suspend fun refreshAccessToken(): String? {
        val refreshToken = getRefreshToken() ?: return null
        
        return try {
            val response = apiService.refreshToken(refreshToken)
            saveTokens(
                response.accessToken,
                response.refreshToken,
                response.expiresIn
            )
            response.accessToken
        } catch (e: Exception) {
            clearTokens()
            null
        }
    }
    
    fun clearTokens() {
        securePreferences.apply {
            remove("access_token")
            remove("refresh_token")
            remove("expires_at")
        }
    }
}
```

### Auth Interceptor with Token Refresh

```kotlin
import okhttp3.Authenticator
import okhttp3.Request
import okhttp3.Response
import okhttp3.Route

class TokenAuthenticator(
    private val tokenManager: TokenManager
) : Authenticator {
    
    override fun authenticate(route: Route?, response: Response): Request? {
        // Avoid infinite loop
        if (response.request.header("Authorization") == null) {
            return null
        }
        
        synchronized(this) {
            val newToken = runBlocking {
                tokenManager.refreshAccessToken()
            }
            
            return newToken?.let {
                response.request.newBuilder()
                    .header("Authorization", "Bearer $it")
                    .build()
            }
        }
    }
}

// Add to OkHttpClient
val client = OkHttpClient.Builder()
    .authenticator(TokenAuthenticator(tokenManager))
    .addInterceptor { chain ->
        val token = runBlocking { tokenManager.getValidAccessToken() }
        val request = chain.request().newBuilder()
            .apply {
                token?.let { addHeader("Authorization", "Bearer $it") }
            }
            .build()
        chain.proceed(request)
    }
    .build()
```

## Biometric Authentication

### Setup

```toml
[versions]
biometric = "1.2.0-alpha05"

[libraries]
androidx-biometric = { group = "androidx.biometric", name = "biometric", version.ref = "biometric" }
```

### Biometric Prompt

```kotlin
import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity

class BiometricAuthManager(private val activity: FragmentActivity) {
    
    fun canAuthenticate(): Boolean {
        val biometricManager = BiometricManager.from(activity)
        return when (biometricManager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG
        )) {
            BiometricManager.BIOMETRIC_SUCCESS -> true
            else -> false
        }
    }
    
    fun authenticate(
        onSuccess: () -> Unit,
        onError: (String) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)
        
        val biometricPrompt = BiometricPrompt(
            activity,
            executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(
                    result: BiometricPrompt.AuthenticationResult
                ) {
                    super.onAuthenticationSucceeded(result)
                    onSuccess()
                }
                
                override fun onAuthenticationError(
                    errorCode: Int,
                    errString: CharSequence
                ) {
                    super.onAuthenticationError(errorCode, errString)
                    onError(errString.toString())
                }
                
                override fun onAuthenticationFailed() {
                    super.onAuthenticationFailed()
                    onError("Authentication failed")
                }
            }
        )
        
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Biometric Authentication")
            .setSubtitle("Log in using your biometric credential")
            .setNegativeButtonText("Use password")
            .build()
        
        biometricPrompt.authenticate(promptInfo)
    }
}
```

### Biometric in Compose

```kotlin
@Composable
fun BiometricAuthScreen() {
    val context = LocalContext.current
    val activity = context as? FragmentActivity ?: return
    
    var authResult by remember { mutableStateOf<String?>(null) }
    
    val biometricManager = remember { BiometricAuthManager(activity) }
    
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        if (biometricManager.canAuthenticate()) {
            Button(onClick = {
                biometricManager.authenticate(
                    onSuccess = {
                        authResult = "Authentication successful!"
                    },
                    onError = { error ->
                        authResult = "Error: $error"
                    }
                )
            }) {
                Icon(Icons.Default.Fingerprint, contentDescription = null)
                Spacer(modifier = Modifier.width(8.dp))
                Text("Authenticate")
            }
        } else {
            Text("Biometric authentication not available")
        }
        
        authResult?.let {
            Spacer(modifier = Modifier.height(16.dp))
            Text(it)
        }
    }
}
```

## Data Encryption

### Android Keystore

```kotlin
import android.security.keystore.KeyGenParameterSpec
import android.security.keystore.KeyProperties
import java.security.KeyStore
import javax.crypto.Cipher
import javax.crypto.KeyGenerator
import javax.crypto.SecretKey
import javax.crypto.spec.GCMParameterSpec

class CryptoManager {
    
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").apply {
        load(null)
    }
    
    private val keyAlias = "MySecretKey"
    
    private fun getOrCreateSecretKey(): SecretKey {
        keyStore.getKey(keyAlias, null)?.let { return it as SecretKey }
        
        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            "AndroidKeyStore"
        )
        
        val keyGenParameterSpec = KeyGenParameterSpec.Builder(
            keyAlias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .build()
        
        keyGenerator.init(keyGenParameterSpec)
        return keyGenerator.generateKey()
    }
    
    fun encrypt(plaintext: String): ByteArray {
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, getOrCreateSecretKey())
        
        val iv = cipher.iv
        val encrypted = cipher.doFinal(plaintext.toByteArray())
        
        // Combine IV and encrypted data
        return iv + encrypted
    }
    
    fun decrypt(encryptedData: ByteArray): String {
        // Extract IV (first 12 bytes for GCM)
        val iv = encryptedData.copyOfRange(0, 12)
        val encrypted = encryptedData.copyOfRange(12, encryptedData.size)
        
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val spec = GCMParameterSpec(128, iv)
        cipher.init(Cipher.DECRYPT_MODE, getOrCreateSecretKey(), spec)
        
        val decrypted = cipher.doFinal(encrypted)
        return String(decrypted)
    }
}
```

### Usage Example

```kotlin
class SecureRepository(
    private val context: Context
) {
    private val cryptoManager = CryptoManager()
    
    suspend fun saveSecureData(data: String) {
        val encrypted = cryptoManager.encrypt(data)
        // Save encrypted data to file or database
        File(context.filesDir, "secure_data.bin").writeBytes(encrypted)
    }
    
    suspend fun getSecureData(): String? {
        return try {
            val encrypted = File(context.filesDir, "secure_data.bin").readBytes()
            cryptoManager.decrypt(encrypted)
        } catch (e: Exception) {
            null
        }
    }
}
```

## Certificate Pinning

### Implementation with OkHttp

```kotlin
import okhttp3.CertificatePinner
import okhttp3.OkHttpClient

object NetworkSecurity {
    
    fun createSecureClient(): OkHttpClient {
        val certificatePinner = CertificatePinner.Builder()
            // Production
            .add(
                "api.example.com",
                "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA="
            )
            // Backup certificate
            .add(
                "api.example.com",
                "sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB="
            )
            .build()
        
        return OkHttpClient.Builder()
            .certificatePinner(certificatePinner)
            .build()
    }
}
```

## Network Security Config

### Create Security Config

Create `res/xml/network_security_config.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- Prevent cleartext traffic -->
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
    
    <!-- Production domain -->
    <domain-config cleartextTrafficPermitted="false">
        <domain includeSubdomains="true">api.example.com</domain>
        <pin-set expiration="2026-12-31">
            <pin digest="SHA-256">AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=</pin>
            <pin digest="SHA-256">BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB=</pin>
        </pin-set>
    </domain-config>
    
    <!-- Debug domain -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">10.0.2.2</domain>
        <domain includeSubdomains="true">localhost</domain>
    </domain-config>
</network-security-config>
```

## ProGuard Security

### Security-Focused Rules

```proguard
# Obfuscate everything
-repackageclasses ''
-allowaccessmodification

# Keep only essential classes
-keep class com.example.app.MainActivity { *; }
-keep class com.example.app.MyApplication { *; }

# Remove logging in production
-assumenosideeffects class android.util.Log {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}

-assumenosideeffects class timber.log.Timber {
    public static *** d(...);
    public static *** v(...);
    public static *** i(...);
}

# Obfuscate API models
-keep class com.example.app.data.** { *; }

# Remove debug code
-assumenosideeffects class * {
    public void setDebug(...);
    public boolean isDebug();
}

# Hide source file names
-renamesourcefileattribute SourceFile
-keepattributes SourceFile,LineNumberTable
```

## Secure Input Handling

### Prevent Screenshots

```kotlin
import android.view.WindowManager

class SecureActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Prevent screenshots and screen recording
        window.setFlags(
            WindowManager.LayoutParams.FLAG_SECURE,
            WindowManager.LayoutParams.FLAG_SECURE
        )
    }
}
```

### Secure Text Input

```kotlin
@Composable
fun SecurePasswordField() {
    var password by rememberSaveable { mutableStateOf("") }
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
        keyboardOptions = KeyboardOptions(
            keyboardType = KeyboardType.Password,
            autoCorrect = false
        ),
        trailingIcon = {
            IconButton(onClick = { passwordVisible = !passwordVisible }) {
                Icon(
                    imageVector = if (passwordVisible) {
                        Icons.Default.VisibilityOff
                    } else {
                        Icons.Default.Visibility
                    },
                    contentDescription = if (passwordVisible) "Hide password" else "Show password"
                )
            }
        },
        modifier = Modifier.fillMaxWidth()
    )
}
```

## Root Detection

### Basic Root Check

```kotlin
class RootDetector {
    
    fun isDeviceRooted(): Boolean {
        return checkBuildTags() ||
                checkSuExists() ||
                checkRootApps()
    }
    
    private fun checkBuildTags(): Boolean {
        val buildTags = android.os.Build.TAGS
        return buildTags != null && buildTags.contains("test-keys")
    }
    
    private fun checkSuExists(): Boolean {
        val paths = arrayOf(
            "/system/app/Superuser.apk",
            "/sbin/su",
            "/system/bin/su",
            "/system/xbin/su",
            "/data/local/xbin/su",
            "/data/local/bin/su",
            "/system/sd/xbin/su",
            "/system/bin/failsafe/su",
            "/data/local/su"
        )
        
        return paths.any { File(it).exists() }
    }
    
    private fun checkRootApps(): Boolean {
        val packages = arrayOf(
            "com.noshufou.android.su",
            "com.thirdparty.superuser",
            "eu.chainfire.supersu",
            "com.koushikdutta.superuser",
            "com.zachspong.temprootremovejb",
            "com.ramdroid.appquarantine"
        )
        
        val packageManager = context.packageManager
        return packages.any {
            try {
                packageManager.getPackageInfo(it, 0)
                true
            } catch (e: Exception) {
                false
            }
        }
    }
}
```

## Input Validation

### Sanitize User Input

```kotlin
object InputValidator {
    
    fun sanitizeString(input: String): String {
        return input
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;")
            .replace("/", "&#x2F;")
    }
    
    fun isValidEmail(email: String): Boolean {
        return android.util.Patterns.EMAIL_ADDRESS.matcher(email).matches()
    }
    
    fun isValidPhoneNumber(phone: String): Boolean {
        return android.util.Patterns.PHONE.matcher(phone).matches()
    }
    
    fun isValidUrl(url: String): Boolean {
        return android.util.Patterns.WEB_URL.matcher(url).matches()
    }
    
    fun isStrongPassword(password: String): Boolean {
        val minLength = 8
        val hasUpperCase = password.any { it.isUpperCase() }
        val hasLowerCase = password.any { it.isLowerCase() }
        val hasDigit = password.any { it.isDigit() }
        val hasSpecialChar = password.any { !it.isLetterOrDigit() }
        
        return password.length >= minLength &&
                hasUpperCase &&
                hasLowerCase &&
                hasDigit &&
                hasSpecialChar
    }
}
```

## Secure WebView

### Configure Secure WebView

```kotlin
@Composable
fun SecureWebView(url: String) {
    AndroidView(
        factory = { context ->
            WebView(context).apply {
                settings.apply {
                    javaScriptEnabled = false  // Disable if not needed
                    allowFileAccess = false
                    allowContentAccess = false
                    allowFileAccessFromFileURLs = false
                    allowUniversalAccessFromFileURLs = false
                    
                    // Clear cache on exit
                    cacheMode = WebSettings.LOAD_NO_CACHE
                    
                    // HTTPS only
                    mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
                }
                
                // Clear cookies
                CookieManager.getInstance().removeAllCookies(null)
                
                loadUrl(url)
            }
        }
    )
}
```

## Best Practices

- Never hardcode secrets in code
- Use EncryptedSharedPreferences for sensitive data
- Implement certificate pinning for production
- Use HTTPS only
- Validate all user input
- Use Android Keystore for cryptographic keys
- Implement token refresh mechanism
- Add FLAG_SECURE for sensitive screens
- Obfuscate code with R8/ProGuard
- Remove debug logs in production

## Security Checklist

### Pre-Release Checklist

- [ ] All API keys in environment variables
- [ ] EncryptedSharedPreferences for tokens
- [ ] Certificate pinning enabled
- [ ] ProGuard/R8 enabled with security rules
- [ ] Debug logging disabled in release
- [ ] FLAG_SECURE on sensitive screens
- [ ] Input validation on all forms
- [ ] HTTPS enforced (no cleartext traffic)
- [ ] Root detection (if needed)
- [ ] Biometric authentication (if appropriate)
- [ ] Token refresh mechanism
- [ ] Session timeout implemented
- [ ] Proper error messages (no sensitive info leaks)
- [ ] Dependencies scanned for vulnerabilities
- [ ] Code review for security issues

## Resources

- [Android Security Best Practices](https://developer.android.com/topic/security/best-practices)
- [Security with HTTPS and SSL](https://developer.android.com/training/articles/security-ssl)
- [Security Tips](https://developer.android.com/training/articles/security-tips)
- [Jetpack Security](https://developer.android.com/topic/security/data)
- [OWASP Mobile Security](https://owasp.org/www-project-mobile-security/)

